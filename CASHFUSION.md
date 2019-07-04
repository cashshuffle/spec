# CashFusion

Authors: Jonald Fyookball and the BCH Wallet Developers
   
## Introduction

THE PROBLEM:

CashShuffle is a powerful tool for obfuscating the origin of a coin.  However, after shuffling a wallet, a user will inevitably wish to consolidate several coins, and for this another tool is needed.    We need a method to coordinate coinjoin transactions with multiple inputs per user.  This is inherently challenging because we want to hide input linkages while simultaneously attempting to blame/ban users who don't sign all their inputs.  

THE SOLUTION:

This scheme takes a "sharding" approach whereby each player gives each other player only 1 input to verify.  In a group of 11 players, each player has 10 transaction inputs, and each input is validated by 1 other player.  This scheme improves on previous ideas because it does not require trusting the server with information about linkages between inputs.   

## Components of CashFusion

There are a few "moving parts" needed to implement the solution:

1. Covert announcements of inputs
2. Commitment scheme
3. Random ordering of players
4. Sharding grid
5. Ephemeral encryption keys
6. Standardized input amounts
7. Blame
8. Fees

## Covert Announcements of Inputs

Each player needs to ultimately share all his inputs with the group because they need to be included in the transaction, but he does not want any other player (or the server) to know his inputs belong to the same user.  

The layered encryption method used in CashShuffle (but repeated for multiple inputs) may be unwiedly here because it could take a long time to perform layered encryption on 110 different inputs, and this could create a large DOS attack surface.   Instead, players can simply announce their inputs to the server over TOR using a different route for each input.  In fact, the entire protocol can take place over TOR. Delays can be added to thwart timing attacks.  

Each player will also have a single transaction output (no change output here) that is announced to the server at the beginning of the process.

## Commitment Scheme

Within CashFusion, the commitment scheme must allow each player to prove they submitted 10 unique inputs which are part of the total set of inputs submitted by all the players for the joined transaction.  Each player is expected to reveal each input to a separate verifying party.

This is a challenging requirement in of itself.  For example, if the commitment was simply a hash value, then it could be trivially broken with a dictionary attack, since the data set is small (100 inputs).  Using salted hashes prevents this attack, but opens a new attack vector in which a player commits the same input more than once.  (The salts prevent what is commited from being necessarily unique, and the non-uniqueness would be undectable since the information about each input is revealed to different parties.)

An additional requirement is that players need to commit to a certain order for their inputs.  In other words, Alice needs to identity which is her "first" input, her "second" input etc, and commit to it early in the process so that collusion attacks are prevented.  For example, if Bob colludes with Alice and knows ahead of time which input of Alice he will validate, he can spoof that commitment.

Our goals can be met as follows:

The commitment for each input X is based on a set of 2 secret numbers A, and B, such that X ⊕ A ⊕ B = 0.  (B can be chosen first as a random number, and A calculated as X ⊕ B.)  A is encrypted with public key k ("commitment key"), which a unique ephemeral key.  The resulting ciphertext (A') is announced along with, and at the same time that the input itself is coveretly announced.  B is announced separately as part of a list, which we can call the B list.  

The B list consists of 10 random numbers B<sub>1</sub>... ...B<sub>10</sub> where each element represents a unique B value.  Unlike the A component, no actual inputs are included here.  The list is simply a set of 10 random numbers, which by itself reveals zero knowledge.  This list also serves the secondary functioning of commiting to an input order (the first item in the list is part of the commitment for the "first" input, the second item for the second input, and so on.)

To reveal an input, the player simply sends X ,A, and k to the verifier. 

For example, imagine Bob is responsible for verifying Alice's first input.  (We will discuss later how this came to be in the sharding grid section).   

Recall that the A number that was announced along with the actual input is encrypted as A', and since the input was covertly announced, it has no linkage to any other input.  No one knows it belongs to Alice, let alone that it is her "first" input.  

Alice gives the key k to Bob, and now Bob can check that A' = Ek(A), and also that X ⊕ A ⊕ B = 0.

Can Alice cheat by submitting the same input to both Bob and Carol?  No, because the input itself is attached to a commitment that can only be decrypted to a single value, and that won't work with 2 different values in the B list.  

Or, more formally: if X is the input Alice wishes to double submit, we have:

 X ⊕ A<sub>1</sub> ⊕ B<sub>1</sub> = 0
 
 X ⊕ A<sub>2</sub> ⊕ B<sub>2</sub> = 0

But, since B<sub>1</sub> ≠ B<sub>2</sub> (the values in the B list must all be unique), then A<sub>1</sub> ≠ A<sub>2</sub>.  Therefore, A' = Ek(A<sub>1</sub>) → A' ≠ Ek(A<sub>2</sub>).


## Random Ordering of Players

In addition to each player creating an order for her inputs, there needs to be an overall order of the players (Alice is player 1, Bob is player 2, etc).  This can left up to the server, since the server is already generally trusted to be non-disruptive.  (Theoretically, the ordering could also can be generated with some additional steps in a trustless fashion via secret sharing.)

## Setting up the Sharding Grid

Once we have the order of players and the order of each players' inputs, then we can set up a "sharding grid" which assigns a verifing party to each input.   Each input is encrypted and sent for verification to the player with a corresponding relative index (the sending players' index plus the index of their input using modular arithmetic).  

That's a mouthful, but the scheme is simple.  For example, if we had only 3 players: Alice (1), Bob (2), and Carol(3), then,  Alice sends input 1 to Bob, and input 2 to Carol.  Bob sends his input 1 to Carol, and his input 2 to Alice.  Carol sends her input 1 to Alice and input 2 to Bob.

With the full set of 11 players and 10 inputs, the sharding grid appears as follows.  The number inside each cell is the number of the player who must validate the input.

<img src="https://raw.githubusercontent.com/cashshuffle/spec/master/shardinggrid.png">

## Ephemeral Encryption Keys

Each input involved in the transaction requires an ephemeral encryption key, used for secure communication to the validating party.  

Each player creates a set of keypairs (one for each input they are responsible for validating), and then shares the public keys with the group.  Separately, each player also creates a set of encrypted proofs (one for each of their own inputs) by encrypting the input along with its secret value and commitment key, using the communication public key of the validating player. 

The public ephemeral keys, along with the encrypted proofs, are sent to all players so that blame can be accurately assigned and witnessed by all.    

## Standardized Input Amounts

All inputs involved in a CashFusion round should be identical, for example 0.1 BCH.  Wallets implementing CashFusion should take existing UTXOs of slightly larger amounts and "shave" off some change in order to assemble a group of 10 coins that will be used as inputs.  Standardizing the input amounts solves two problems:  First, it prevents "weakness from uniqueness" whereby the linkages can be logically grouped. Secondly, it ensures that all players are contributing sufficient funds to the transaction, which would be otherwise difficult.

## Blame

In this scheme, every input (except fees) is validated by a participant in the fusion.  If an input cannot be validated, the validating player issues a blame message.  If a blame accusation is issued incorrectly, the accused can prove their innocence while simultaneously proving the accuser is incorrect, by revealing the ephemeral private key which can be used to decrypt the input along with the secret salt.  Players can then ensure the input is valid and matches the previously announced hash.    
  
Because blame accusations require revealing an input, it is best to minimize them.  In the normal case without blame, each player reveals only one input to each counterparty, so no additional information about any linkages is revealed.  If a player is blamed unfairly and must publicly reveal an additional input, then this leaks a small amount of information to all participants.  

If more than one blame message is issued in the same round, only one blame message needs to be processed.   

## Fees

Fees can be covertly announced to avoid any additional linkage issues between other inputs.  Each player would contribute one small input to cover the fee and expect no change. Enforcing fees could overcomplicate the protocol.  For now, we propose simply including the proper fee in the software implemenation.  Theoretically, in the future, fees could be enforced with the same sharding idea, with each player having one other random player verify their fee. 

# Protocol Phases

## Phase 1.  Registration on the Pool 

Message 1A (from client):  ( `<MESSAGE TYPE><“REGISTER”><COIN_SIZE><OUTPUT_ADDRESS>` )

Message 1B (from server): ( `<MESSAGE TYPE><“POOL READY”><POOL_SESSION_ID><LIST_OF_OUTPUT_ADDRESSES>` )

Players connect to and register on the pool, while announcing their output address.  The server should refuse to register players with banned IP addresses (because of blame from a recent round).  Once 10 players register on the pool, the server can announce the list of 10 output addresses to all players. 


## Phase 2. Announce "B List" Commitments and Ephemeral Keys 

Each player will create a "B list" (see commitment scheme above) which is a list of 10 random numbers.  Each player will also create 10 ephemeral encryption keys, one for each input they will validate, plus one additional key for blaming ("blame PubKey").

Message 2 (from client): `<MESSAGE TYPE><POOL_SESSION_ID><BLAME PUBKEY><PUBKEY E1>...<PUBKEY E10><B_LIST_VALUES>`

If any player fails to send a correctly formatted Message 2, then blame is assigned to that player and the round aborts.

## Phase 3. Share Commitments and Keys

Once the server has received message 2 from all players, it sends message 3, announcing all payloads for all players, where each payload contains the information sent in Message 2.  

Message 3 (from server): `<MESSAGE TYPE><POOL_SESSION_ID><PAYLOAD_PLAYER 1>...<PAYLOAD_PLAYER 11>`  

The Blame Pubkey is included in the payload, which helps to identify each player, while the ordering of the message itself refelects the ordering of the players.

## Phase 4. Covert Announcement of Signed Inputs 

Once message 3 has been received, each client should covertly announce their inputs (using TOR), and only the POOL_ID, sending 11 different instances (one for each input of standard size, plus one input for the fee) of the following message:

Message 4 (from client): `<MESSAGE TYPE><POOL_SESSION_ID><SIGNED SERIALIZED INPUT><ENCRYPTED A_VALUE>`

(The encrypted "A value" is described previously in the commitment scheme.)

Note that only the pool session is required to post this information because it is covert; however only those who registered on the pool should have this unique id for the session.  

The transaction will use the sighash type ALL|ANYONECANPAY, which reduces the interaction required between players.  Specifically, this allows users to sign the inputs ahead of time without knowing ahead of time all the inputs that will be part of the transaction.  This setup automatically handles the case of users not signing inputs.

Fee inputs are part of the transaction using the same sighash type and are included in this message.
 
## Phase 5. Sharing the Inputs

Clients should announce all inputs to the server within a specified time window (for example 15 seconds), after which the server closes the announcement window, and proceeds to rebroadcast all the inputs to all players.

Rebroadcasting all inputs together (rather than one at a time as each comes in) helps prevent timing timing attacks since only the server would know the times.  The input announcements should have stagerred delays to mitigate timing attacks from the server itself.  Also, this approach prevents Bob from maliciously re-submitting Alice's input with his own (incorrect) commitment "A" value.  In the case where Alice resubmits her own input (or Bob resubmits Alice's transaction from a prior round with a now-bogus signature), the server should include all the inputs and let the blame phases handle the problem.

Message 5 (from server) `<MESSAGE TYPE><POOL SESSION_ID><INPUT 1>...<INPUT INDEX n>`

(Input here means serialized input plus the encrypted A value).

The client should check all signatures (this should be fast due to libsecp256k1) before broadcasting the transaction.  Note that it is possible to have extra bogus inputs (invalid or missing signatures) but those can be just discarded.  The transaction will still work if it has enough valid inputs, and should be executed if valid.  Thus, the client code needs to loop through the set and determine if it can assemble the transaction.  

If the transaction is valid, the client should broadcast it to the BCH network.  Note the client doesn't have to check if the transaction was accepted or was malleated.  Last second invalidations are always possible due to the race condition of double spending.

## Phase 6. Invoke Blame Process

If the client finds any problems ih phase 5, we need to invoke the blame portion of the protocol.  How do we know whether or not to enter the blame phases?  We simply determine if the transaction is valid on the basis of valid inputs.  If invalid inputs can be discarded but the transaction is still valid without them, there is no blame.

An interesting edge case arises when the same UTXO appears in more than one input.  If that happens, we also enter the blame phase. 

Message 6 (from server) `<MESSAGE TYPE><POOL SESSION_ID>`

This is just a junction that tells the server to continue with blame phases.

## Phase 7. Announce Ordering

Once the server knows we're in the blame portion, it creates a random order for the players.

Message 7 (from server): `<MESSAGE TYPE><POOL_SESSION_ID><BLAME_PUBKEY_PLAYER 1>...<BLAME_PUBKEY_PLAYER 11>`  

The Blame Pubkey is included in the payload, which helps to identify each player, while the ordering of the message itself refelects the ordering of the players.

## Phase 8. Send Proofs

Each player will create 10 “proofs”.  Each proof shall consist of the X ,A, and k values (described in the commitment scheme), where X is the signed serialized input, A is a secret number, and k is the public "commitment key". The entire proof is encrypted by the verifying player’s key, based on the sharding grid.

Message 8 (from client): `<MESSAGE TYPE><POOL_SESSION_ID><BLAME PUBKEY><SIG_FOR_BLAME_PUBKEY><PROOF 1>...<PROOF 10>`

## Phase 9. Share and Validate Proofs

Then the server sends to all:

Message 9 (from server): `<MESSAGE TYPE><POOL_SESSION_ID><SIG FOR BLAME PUBKEY PLAYER 1><PROOF 1>...<PROOF 9>...<BLAME PUBKEY Player 11><PROOF 1>...<PROOF 10>`

After reciving Message 9, the client will extract the proofs that it is responsible for, and checks each one.    If the client finds any issues with either the transaction inputs, it assigns blame. 

## Phase 10. Assign Blame

First the client notifies the server:

Message 10A (from client): `<MESSAGE TYPE><POOL_SESSION_ID><BLAME PUBKEY><SIG BLAME PUBKEY><INPUT INDEX TO BE BLAMED>`

Then the server notifies all clients with a similar message:

Message 10B (from server) `<MESSAGE TYPE><POOL_SESSION_ID><BLAME PUBKEY><SIG BLAME PUBKEY><INPUT INDEX TO BE BLAMED>`

If the server receives several instance of Message 10A, it should only pick one input to be blamed (for example the lowest one by lexicographical order).  

In the "repeated UTXO" edge case (which we mentioned in phase 6), the mere presence of 2 valid signed inputs indicates maliciousness; it is not necessary to specifically determine which members of this set satisfy their commitments.  Instead, clients should assign blame via Message 10A if they receive any proof containing such a repeat.  Nothing special is required to indicate the edge case because all players will see the duplicated UTXO.  When the message comes in blaming the index, the other clients will understand the reason and do not even have to verify the blame message's accuracy as they usually would do.  Also, note that this kind of blame cannot be refuted.

## Phase 11. Refute Blame

If Alice blames Bob, but Bob is innocent, Bob can refute the blame, while counter-blaming the accuser (Alice).  He does that by sharing his ephemeral private key.

Message 11A (from client): `<MESSAGE TYPE><POOL_SESSION_ID><BLAME PUBKEY><SIG BLAME PUBKEY><EPHEMERAL PRIVATE KEY>`

The server can then rebroadcast the same message

Message 11B (from server): `<MESSAGE TYPE><POOL_SESSION_ID><BLAME PUBKEY><SIG BLAME PUBKEY><EPHEMERAL PRIVATE KEY>`

## Phase 12: Process Blame

After determing who is to blame, the client and server should terminate the round.

# Further Discussion

This document is still evolving.  In no particular order, here are a few points which can be clarified:

**1. Announcement of Outputs**

It may not be clear why outputs are announced at the beginning, but there doesn't appear to be a better way to do it.  For example, including the output in the sharding grid as just another part of the transaction doesn't work well because a new set of problems arises should a malicious actor include an extra output.  
 
**2. Pool Size**

The same scheme would work with a smaller number of players, and this may be necessary for a time if liquidity is insufficient, while providing some additional privacy.  In the medium-long term, we should strive toward the full 10 players in order to achieve statistically unlinkable inputs.

**3. Wallet Considerations**

When shaving inputs in preparation for a fusion transaction, the wallet should take care to avoid having the standardized output naively marked as "unshuffled", leading to a reshuffling attempt in the normal CashShuffle protocol, and disrupting the fusion.  Also, the change portion of the shaved amount *should* be marked as unshuffled so it can be put into a CashShuffle round.

As a separate development effort, or in the future, we may want to add additional flagging and coin control to best direct the wallet how and when to perform fusions, in order to maximize privacy and UX.

**4. DOS Attacks**

CashFusion provides a structure that theoretically allows blame to be assigned to disruptive parties, but there are different ways this can be implemented.  One issue is that when clients keep changing their IP address over TOR, as required by this protocol, banning IPs becomes ineffective.  In the simplest implementation, blame is assigned but is irrelevant because the only action taken is to terminate the round anyway.  Unless the network is under heavy attack, eventually the transactions will succeed due to randomness (eventually only honest players will be in a round together)

In more sophisticated implementations of CashFusion, the offending player can be removed and a replacement player can be added.  An even stronger anti-DOS scheme would involve recursively shrinking the pool size each time someone is blamed, similar to coinshuffle++.  It is also possible for players to reveal all the inputs of the blamed player and utilize UTXO-level banning. 
