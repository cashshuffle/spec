# CashFusion

Authors: Jonald Fyookball, Mark B. Lundeberg
   
## Introduction

THE PROBLEM:

CashShuffle is a powerful tool for obfuscating the origin of a coin.  However, after shuffling a wallet, a user will inevitably wish to consolidate several coins, and for this another tool is needed.    We need a method to coordinate coinjoin transactions with multiple inputs per user.  This is inherently challenging because we want to hide input linkages while simultaneously attempting to blame/ban users who don't sign all their inputs.  

THE SOLUTION:

This scheme takes a "sharding" approach whereby each player gives each other player only 1 input to verify.  In a group of 11 players, each player has 10 transaction inputs, and each input is validated by 1 other player.  This scheme improves on previous ideas because it does not require trusting the server with information about linkages between inputs.   


## Commitment Scheme

Within CashFusion, the cryptographic commitment scheme must allow each player to prove they submitted 10 unique inputs which are part of the total set of inputs submitted by all the players for the joined transaction.  Each player is expected to reveal each input to a separate verifying party.

This is primarily accomplished by having each player commit a list of 10 hashes ("list commitment").  Each item in the list is a salted hash of one of their inputs, with a separate secret value for each.  To prove they included the input in the transaction, the player reveals the secret salt value along with which input it corresponds to.  The verifier can then reconstruct the hash value and verify it matches.

But, this initial commitment still has a problem.  Alice could hash the same input two (or more) different ways with two different salts, and then present one to Bob and one to Carol.  Since neither Bob nor Carol can see each other's verification of Alice's inputs, they won't know the inputs are the same.  Thus, Alice can avoid having others detect that not all her inputs are unique. 

Fortunately, we can prevent this problem with a second commitment ("salt commitment"), which is a (non-salted) hash of the secret salt value used in the first commitment.  

While the list commitment is associated with a player but not publicly linked to any specific inputs, the salt commitment is announced with and "attached" to an input itself, but not publicly linked to any specific player.  Alice is now prevented from using two different salts for the same input, because the verifier also checks that the secret salt hashes to the salt commitment for the input.
 
Finally, each player needs to commit to a certain order for their inputs.  This is automatically handled because the list of salted hashes has an order.  We'll see why this is important in the following section.


## Sharding Grid

The sharding grid is a simple way to assign a verifying party to each input.  Specifically, each input is verified by a player with a corresponding relative index (the sending players' index plus the index of their input using modular arithmetic).  For instance, if we had only 3 players: Alice (1), Bob (2), and Carol(3), then,  Alice sends input 1 to Bob, and input 2 to Carol.  Bob sends his input 1 to Carol, and his input 2 to Alice.  Carol sends her input 1 to Alice and input 2 to Bob.

With the full set of 11 players and 10 inputs, the sharding grid appears as follows.  The number inside each cell is the number of the player who must validate the input.

<img src="https://raw.githubusercontent.com/cashshuffle/spec/master/shardinggrid.png">

Setting up the sharding grid requires two things:  First, a distinct order of each players' inputs (as discussed in the previous section).  Secondly, an overall order of the players so it is clear who is player 1, etc.  The ordering of the players must be established only after all the inputs and commitments are announced to avoid collusion attacks.  For example, if Bob colludes with Alice and knows ahead of time which input of Alice he will validate, he can spoof that commitment.  

The ordering of players can be performed by server, since it is already trusted to be non-disruptive.  (Theoretically, the ordering could also can be generated with some additional steps in a trustless fashion via secret sharing.)

## Additional Components

**1. Inputs and Outputs**

Inputs and outputs should be announced to the server over TOR to avoid the server learning the linkages from the clients' IP addresses.  Delays can be added to thwart timing attacks. (Layered encryption à la CashShuffle is probably too slow.) 

Outputs can't be treated the same as inputs because a new set of problems would arise based on a malicious actor including extra outputs.  Instead, each player will announce a single transaction output (no change output) at the beginning of the process.  

All inputs involved in a CashFusion round should be identical, for example 0.1 BCH.  This prevents "weakness from uniqueness" whereby the linkages can be logically grouped. It also ensures that all players are contributing sufficient funds to the transaction, which would otherwise be difficult.

Wallets implementing CashFusion can take existing UTXOs of slightly larger amounts and "shave" off some change in order to assemble a group of 10 coins that will be used as inputs.   Here, the wallet should take care to avoid having the standardized output naively marked as "unshuffled", leading to a reshuffling attempt in the normal CashShuffle protocol, and disrupting the fusion.  Also, the change portion of the shaved amount *should* be marked as unshuffled so it can be put into a CashShuffle round.

Each player should contribute one fee input, covertly announced along with his other inputs.  The protocol does not enforce the inclusion of a fee, but theoretically could by including it in the scheme.

**2. Ephemeral Encryption Keys**

Every input will have a "communication" keypair created by the verifier.  This allows input proofs to be encrypted by its owner, sent to the server, and rebroadcast to all players (but only the verifier can decrypt the message).  Although only one player (the input's owner) will be using the public key in this manner, all players receive the public key so that blame can be accurately assigned and witnessed by all. The private key can be revealed to counter-blame any false blame accusations.

There is also an "identity" keypair to ensure that critical messages can't be spoofed by other players.

**3. Blame and DOS**

Whereas a successful fusion and even a blame reveal zero knowledge, a counter-blame leaks some information to all players.  Consequently, only 1 set of blame/counter-blame should be processed per round.  

CashFusion provides a structure that theoretically allows blame to be assigned to disruptive parties, but there are different ways this can be implemented.  One issue is that when clients keep changing their IP address over TOR, as required by this protocol, banning IPs becomes ineffective.  In the simplest implementation, blame is assigned but is irrelevant because the only action taken is to terminate the round anyway.  Unless the network is under heavy attack, eventually the transactions will succeed due to randomness (eventually only honest players will be in a round together)

In more sophisticated implementations of CashFusion, the offending player can be removed and a replacement player can be added.  An even stronger anti-DOS scheme would involve recursively shrinking the pool size each time someone is blamed, similar to coinshuffle++.  It is also possible for players to reveal all the inputs of the blamed player and utilize UTXO-level banning. 

# Protocol Phases

## Phase 1.  Registration on the Pool 

Message 1A (from client): `<MESSAGE_TYPE><COIN_SIZE><OUTPUT_ADDRESS>` 

Message 1B (from server): `<MESSAGE_TYPE><POOL_SESSION_ID><LIST_OF_OUTPUT_ADDRESSES>`

Players connect to and register on the pool, while announcing their output address.   Once 11 players register on the pool, the server can announce the list of 11 output addresses to all players. 

## Phase 2. Announce Salted Hashes and Communication Keys 

To prepare, each player will serialize each of his signed inputs, generate a unique random salt for each input, and from that, a salted hash for each input.   The list of salted hashes should then be put in a random order. Each player will also create 10 "communication keys", one for each input they will validate , plus one additional key for identity ("identity key").

Message 2 (from client): `<MESSAGE TYPE><POOL_SESSION_ID><IDENTITY_KEY><COMMUNICATION_KEY_1>...<COMMUNICATION_KEY_10><IDENTITY_SIGNATURE>`

In this phase and all subsequent phases that include `<IDENTITY_SIGNATURE>`, the signature is the final piece of the message.  It is a schnorr secp256k1 signature over the entire preceeding section of the message.

If any player fails to send a correctly formatted Message 2, then blame is assigned to that player and the round aborts.

## Phase 3. Share Commitments and Keys

Once the server has received message 2 from all players, it sends message 3, announcing all payloads for all players, where each payload contains the information sent in Message 2.  

Message 3 (from server): `<MESSAGE TYPE><POOL_SESSION_ID><PAYLOAD_PLAYER 1>...<PAYLOAD_PLAYER 11>`  

## Phase 4. Covert Announcement of Signed Inputs 

Once message 3 has been received, each client should covertly announce their inputs (using TOR), sending 11 different instances (one for each input of standard size, plus one input for the fee) of the following message:

Message 4 (from client): `<MESSAGE TYPE><POOL_SESSION_ID><SIGNED_SERIALIZED_INPUT><SALT_COMMITMENT>`

The salt commitment at the end of this message is a SHA-256 hash of the secret salt value generated in phase 2 for each input.

Note that only the pool session is required to post this information because it is covert; however only those who registered on the pool should have this unique id for the session.  

The transaction will use the sighash type ALL|ANYONECANPAY, which reduces the interaction required between players.  Specifically, this allows users to sign the inputs without knowing ahead of time all the inputs that will comprise the transaction.  This setup automatically handles the case of users not signing inputs.

 ## Phase 5. Sharing the Inputs, Ordering the Players, and Executing the Transaction

The purpose of this phase is primarily to rebroadcast all the inputs to all the players so they can assemble and broadcast the transaction.  Secondarily, the server also generates and shares a random ordering of the players, which will be used later in the blame phases, if blame is necessary.

As opposed to rebroadcasting each covertly announced input as it arrives, the server rebroadcasts them all together.  This limits the possibility of timing attacks to the server itself, which can be further mitigated by announcing inputs randomly within a specified time window (such as 15 seconds).  The server sends Message 5 when this window expires, but the server should wait an extra few seconds to account for latency and error.

Rebroadcasting all inputs together also prevents Bob from maliciously re-submitting Alice's input with his own salt commitment.  In the case where Alice resubmits her own input (or Bob resubmits Alice's input from a prior round with a bogus signature), the server should include all the submitted inputs (including duplicates) and let the blame phases handle the problem.

Message 5 (from server) `<MESSAGE_TYPE><POOL_SESSION_ID><INPUT_1><SALT_COMMITMENT_1>...<INPUT_11><SALT_COMMITMENT_11><IDENTITY_KEY_1>...<IDENTITY_KEY_11>`

The list of identity keys at the end of this message designates the order of the players.  This is used in the next phase in case of blame.  In the normal case when the fusion is successful, the ordering isn't used.

Once the client receives Message 5, it should check all signatures before broadcasting the transaction.  Note that it is possible to have extra bogus inputs but those can be just discarded.   If a valid transaction can be constructed on the basis of valid inputs, the client should do so, and broadcast it to the BCH network.  The client doesn't have to check if the transaction was accepted by the BCH network, nor if it was malleated.  Last second invalidations are always possible due to the race condition of double spending.

A set of edge cases arises when the same UTXO appears in more than one input.  If that happens, the client knows it needs to enter the blame phases.  In fact, it might be most efficient for the client to first check for any duplicate UTXO, since their presence automatically triggers blame.

## Phase 6. Send Proofs

This is the first of the blame phases.  Using the ordering of identity keys it received in the previous message, the ordering of each players' list commitment from phase 3, and the logic of the sharding grid, each player will create 10 proofs.  Each proof is of the form:

`<INPUT><SECRET_SALT_VALUE>`

This proof is then encrypted by the verifying player’s key, based on the sharding grid, and sent as:

Message 6 (from client): `<MESSAGE TYPE><POOL_SESSION_ID><IDENTITY_KEY><ENCRYPTED_PROOF_1>...<ENCRYPTED_PROOF_10><IDENTITY_SIGNATURE>`


## Phase 7. Share and Validate Proofs

If we consider each player's payload from the previous Message 6 as:

`<IDENTITY_KEY><ENCRYPTED_PROOF_1>...<ENCRYPTED_PROOF_10><IDENTITY_SIGNATURE>`

Then Message 7 is simply the server rebroadcasting all payloads to all players.

Message 7 (from server): `<MESSAGE_TYPE><POOL_SESSION_ID><PAYLOAD_1>...<PAYLOAD_11>`

After reciving Message 7, the client will extract the proofs that it is responsible for, and checks each one.    If the client finds any issues with either the transaction inputs, it assigns blame. 

## Phase 8. Assign Blame

To assign blame, the client simply sends a message to the server with the `<FAULTY_SALTED_HASH>`, which is the salted hash they were attemping to verify but could not. 

Message 8A (from client): `<MESSAGE_TYPE><POOL_SESSION_ID><IDENTITY_KEY><FAULTY_SALTED_HASH><IDENTITY_SIGNATURE>`

Then the server rebroadcasts the information:

Message 8B (from server) `<MESSAGE_TYPE><POOL_SESSION_ID><IDENTITY_KEY><FAULTY_SALTED_HASH><IDENTITY_SIGNATURE>`

If the server receives several instance of Message 8A, it can ignore the subsequent messages and simply rebroadcast 8B based on the first instance of blame it sees.  

Regarding the edge cases of multiple inputs with the same UTXO:  They are mostly all handled by entering the blame phases upon witnessing them.  However, there is one particular case that needs special treatment, and that is when Alice produces multiple valid signed inputs.  In this case, the client still sends Message 8A as normal, but all clients need to be aware when edge case is in play.  That way, if Alice attempts to counter-blame,  clients can still award blame to Alice when checking her claim.  (Even though she might have a valid counter-proof, she cheated by submitting multiple UTXO).  

Note that this edge case must have multiple **valid signed** inputs.  In the case when 2 or more inputs share the same UTXO but only 1 is valid, the protocol proceeds with a normal blame message since there is no ambiguity.

## Phase 9. Counter-Blame

If Alice blames Bob but Bob is innocent, then Bob can refute the blame while counter-blaming Alice.  He does that by sharing his ephemeral "communciation" private key for the input in question.  Again, it is not possible for Alice to counterblame in the special case where she signed multiple inputs using the same utxo.

Message 9A (from client): `<MESSAGE_TYPE><POOL_SESSION_ID><IDENTITY_KEY><COMMUNICATION_PRIVATE_KEY><IDENTITY_SIGNATURE>`

The server can then rebroadcast the same message

Message 9B (from server):  `<MESSAGE_TYPE><POOL_SESSION_ID><IDENTITY_KEY><COMMUNICATION_PRIVATE_KEY><IDENTITY_SIGNATURE>`

## Phase 10: Process Blame

After determing who is to blame, the client and server should execute any additional anti-DOS measures, and terminate the round.
 
