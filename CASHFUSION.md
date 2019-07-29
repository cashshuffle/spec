# CashFusion 

Authors: Jonald Fyookball, Mark B. Lundeberg
 
## Introduction

**THE PROBLEM:**

CashShuffle is a powerful tool for obfuscating the origin of a coin.  However, after shuffling a wallet, a user will inevitably wish to consolidate several coins, and for this another tool is needed.    We need a method to coordinate coinjoin transactions with multiple inputs per user.  This is inherently challenging because we want to hide input linkages while simultaneously attempting to blame/ban users who don't sign all their inputs.  

**THE SOLUTION:**

We propose a blind verification scheme in which each input and output is validated by a random player, using a series of cryptographic commitments that allow an uncooperative participant to be identified and banned, while revealing zero knowledge in most cases.  

CashFusion allows flexibility by allowing each participant to use an essentially arbitrary number of inputs and outputs, of arbitrary amounts.  And it provides anonymous, trustless coordination with zero-knowledge of linkages revealed to other players or the server, in most cases. 

# PART I: Protocol Overview

To review the big picture, the final result is a large coinjoin transaction with many participants and many inputs and outputs.  In the simplest form, this is achieved by players covertly sending their transaction components (inputs and outputs) to the server.  The server then shares all the information with all players, the players sign all their inputs, then (covertly) send them back to the server, and finally the server again shares with everyone so that all players have everything necessary to assemble a valid transaction.

But, in order add the blame capabilities (to allow banning a user who didn't sign all her inputs), each player first creates a salted hash of each of his inputs and outputs, and sends them to the server.  Once all the players have joined the pool and have sent their commitments, then proceed to covertly announce their transaction components to the server over TOR so the server.

It there is a problem with the transaction, each component is assigned to another player at random for verification.  The owner of the component sends the secret information, encrypted with one of the communication keys that's paried with each commitment.

To prevent players from cheating by using the same component in multiple commitments ( by hashing it with a different salt),  the salt value itself  is also hashed and paired  with the covert announcement of the transaction component.

If there is a problem during verification, the verifier reveals his private  communication key to the server, and the server determines whether to blame the accused or the accuser.

Finally, to prevent DOS attacks, the blame mechanism will kick out the bad player and attempt the round again with one less player.  The process can be repeated as many times as desired.

## Inputs

The scheme described so far would generally be sufficient to provide trustless coinjoins with multiple inputs.  However, it would have strict limitations of requiring fixed input amounts and a single output. 

The challenge with variable input amounts is ensuring  every player contributed sufficient funds to cover their outputs. 

To solve this, a set of Pedersen commitments are utilized. Using  its homomorphic property, the server can ensure the sum of inputs and outputs for each player is zero, without knowing any of the values. And, each input or output that is verified will be also cross-checked against its paired amount commitment, thus linking the component commitment to the amount commitment in a simple way.

Inputs are represented by a positive integer indicating the number of satoshis, and outputs by a negative integer. Fees are handled elegantly by subtracting a standard amount from an input or adding a standard amount to an output. For example, if an input is 10,000 satoshis and the standard fee for an input is 150 satoshis, the Pedersen commitment should be for 9850.  This idea can be extended to allow players to add extra fees by relaxing the rule to allow the commit to be less than or equal to the expected value. (e.g. x <= 9850).

## Outputs

Allowing a flexible number of outputs presents its own challenge. Whereas inputs could  be signed over all outputs and "extra" inputs ignored, extra outputs would invalidate the transaction and are hard to detect and blame when announced covertly.

To solve this, we utilize 2 more puzzle pieces:  blind signatures, and "blank" placeholder components.

Here, blind signatures function like a submission "token" that is required with ALL covert announcements (not just outputs) , as follows:  Players blind a message (the message is the transaction component plus the salt commitment) and present the blinded message to the server for signing.  Since it is blinded, the server doesn't know the actual contents of the transaction component.

When attempting a covert announcement, the player unblinds the message and the server can verify their signature matches the message.  The server is prevented from learning any linkages but it does set a known limit on how many "components" each player is submitting. 

Next, we set a requirement that each player must covertly announce exactly 23 components.  Why 23?  It could be anything, but this allows for example 8 outputs and 15 inputs.  By forcing each player to commit to exactly 23 components, this prevents Alice from submitting anything extra because the server won't accept the same signature more than once and will disallow the covert announcement.

Since a player normally will use less than 23 inputs and outputs, the remaining components are considered "blanks" but still need some tangible form:  they still need to be a data string that produces a salted hash, etc.  For example, "BLANK33ed2a3a280a549a672b".  These can be randomly generated by the user and are assigned an amount value of zero.  They are verified like any other transaction component.

Therefore, Alice is prevented from covertly announcing a transaction input or output and not including it in her component commitements because then she'll be missing a component somewhere else.

# PART II: Protocol Phases

## Phase 1.  Registration on the Pool 

This phase begins the fusion process with a player registering on the pool for some coinsize tier.

Message 1 (from client): `<MESSAGE_TYPE><TIER>`

## Phase 2.  Blind Signature Setup

In this phase, the server replies back to the player who just joined and shares 2 things:  

**a.** The public key the server will use for the blind signatures
 
**b.** 23 nonce points which will be used for blind signatures

The nonce points will be unique for each player but the public key needs to be the same for all players.  If the public key is not the same, then the server could use this to unmask all the hidden linkages by grouping the covert announcements based on the key.  This is prevented by making the public key part of the OP_RETURN output.
 
The public key also will serve a second function by doubling as a pool identifier so that the server knows which pool a covert announcement belongs to.

Message 2 (from server to a specific client): `<MESSAGE_TYPE><BLIND_SIGNATURE_PUBLIC_KEY><NONCE_1>...<NONCE_23>`

## Phase 3.  Player Commitments

In this phase, the player sends a message containing 23 commitments to the server (one for each of their transaction components).  Each commitment is a tuple consisting of a salted SHA-256 hash of their component, an ephemeral "communication" public key, and the (Pedersen) amount commitment for the component.  Additionally, there is a SHA-256 hash of random number which serves as a random number commitment from the player.

Message 3 (from client to server): `<MESSAGE_TYPE><PAYLOAD_1>...<PAYLOAD_23><BLIND_SIGNATURE_REQUEST_1>...<BLIND_SIGNATURE_REQUEST_23><RANDOM_NUMBER_COMMITMENT><COMBINED_PEDERSEN_SECRET_VALUE>`

Each payload is of the form:

`<SALTED_HASH_OF_COMPONENT><COMMUNICATION_KEY><AMOUNT COMMITMENT>`

Each `<SALTED_HASH_OF_COMPONENT>` is of the form:

SHA-256(<256-bit random salt>, `<COMPONENT>`),

where `<COMPONENT>` = `<TYPE><DATA>`, where:
- for inputs, TYPE=1 and DATA = `<PREV_TXID><PREV_INDEX>`
- for outputs, TYPE=2 and DATA = `<PUBKEYHASH><AMOUNT>`
- for blanks, TYPE=3 and DATA = `<RANDOM NUMBER>`

Each blind signature request is a 32-byte blob.

If any duplicate component commitments (salted hash commitments) are detected (either from the same player or from different players), all the players involved in the duplication should be blamed.  The server also needs to check that the `<COMBINED_PEDERSEN_SECRET_VALUE>` matches with a zero sum for the set of Pedersen commitments from each player.

## Phase 4.  Sharing Blinded Signatures

This phase does not begin until the sufficient number of players has joined the pool.  Once this occurs, the server will send a separate message to each player containing 23 blinded signatures. 
 
Message 4 (from server to a specific client): `<MESSAGE_TYPE><BLIND_SIGNATURE_1>...<BLIND_SIGNATURE_23>` 

## Phase 5.  Covert Announcements of Inputs and Outputs

In this phase, each player covertly sends 23 separate messages.  Each message will be either an unsigned transaction input, a transaction output, or a "blank".  The client sends 23 instances of this message:

Message 5 (covertly from client to server): `<MESSAGE_TYPE><SERVER_KEY><TRANSACTION_COMPONENT><UNBLINDED_SIGNATURE><SALT_COMMITMENT>`

The salt commitment is a SHA-256 hash of the salt used for the component commitment in phase 3.

When the client receives Message 4, there will be time window of 15 seconds during which the client should randomly announce his transaction components, using a different TOR route for each.

## Phase 6.  Sharing Components and Commitments

This phase does not begin until the server-side time period for covert announcements ends.  If the client side time window is 15 seconds then it is suggested the server side window be 20 seconds.

Before sharing the information, the server can also perform a "sanity check" to make sure there's enough inputs and outputs and enough variation in the transaction amounts.  This prevents malicious parties from including too many amounts on the wrong tier, or attempting to join with too few inputs/outputs.  If the sanity check fails, the round is aborted.

When ready, the server sends a message to all players containing all the components and their salt commitments (from phase 5) and all the other commitments (from phase 3).   The ordering here should be randomized to avoid any of the players trivially grouping them.

Message 6 (from server to all clients): `<MESSAGE_TYPE><TRANSACTION_COMPONENT_1><SALT_COMMITMENT_1>...<TRANSACTION_COMPONENT_N><SALT_COMMITMENT_N><COMMITMENT_PAYLOAD_1>...<COMMITMENT_PAYLOAD_N>`

These commitment payloads are the commitments from phase 3.

## Phase 7.  Covert Announcements of Signatures

First the players will create a special OP_RETURN with a payload that contains a SHA-256 hash of the concatenation of all the commitment tuples, sorted numerically ascending, and additionally: the server's public blinding key from phase 2. The players will assemble the transaction by sorting the components, including the OP_RETURN output (according to BIP69).
  
There will be time window of 15 seconds beginning upon receipt of message 6, during which the client should randomly announce his signatures, using a different TOR route for each.

Message 7A (covertly from client to server): `<MESSAGE_TYPE><SERVER_KEY><PUBKEY><TRANSACTION_SIGNATURE><COMPONENT_INDEX>`

The pubkey for the address is included as part of normal bitcoin script operations. The component index corresponds to the order of components announced in message 6 so that the server can easily pair unsigned inputs with their signatures.  

The client will also send a separate message to reveal (non-covertly) the random number commited to in Phase 3.

Message 7B (non-covertly from client to server): `<MESSAGE_TYPE><RANDOM_NUMBER>`

## Phase 8.  Executing the Transaction and Assigning Verifiers

This phase does not begin until the server-side time period for covert announcements of signatures ends.  If the client side time window is 15 seconds then it is suggested the server side window be 20 seconds.

The server will send back the signatures to all clients and will also include verifier information for each player in a separate message.  If the transaction is valid, it will be broadcast to the BCH network and the fusion process is complete.  If the transaction is invalid, the players will use the verifier information in Phase 9.

Message 8A (from server to all clients): `<MESSAGE_TYPE><SIGNATURE_1>...<SIGNATURE_N>`

The verifier information is based on a determinstic function that selects a random verifier to each commitment, where the "verifier" is an owner of a commitment not belonging to the commitment being verified.  The function operates using a random seed generated from the client's secret number, and selects from the set of all the commitments excluding those of the player to obtain a verifier for each commitment.

Because the process is deterministic, the player will be able to obtain the same information about who is verifiying his inputs, so there is no need to send it to the player.  However, the server should inform each verifier about what message to expect.  Otherwise, nothing forces the verifier to actually send the correct proof to the right person.

Since the server knows the list of commitments for each player, it can then send each player a list of which commitments to verify, each paired with an indication of which of his commitments' communication keys are used for the encryption.

Message 8B (from server to a specific client): `<MESSAGE_TYPE><COMMITMENT_TO_VERIFY_1><VERIFYING_COMMITMENT_1>...<COMMITMENT_TO_VERIFY_N><VERIFYING_COMMITMENT_N>`
  
## Phase 9.  Sending and Sharing Proofs

Message 9A (from client to server): `<MESSAGE_TYPE><ENCRYPTED_PROOF_1>...<ENCRYPTED_PROOF_N>`

Each proof is of the form: `<COMMITMENT_TO_VERIFY><SECRET_SALT><TRANSACTION_COMPONENT><PEDERSEN_AMOUNT>` and encrypted with the verifier's communication key.

Message 9B (from server to all clients): `<MESSAGE_TYPE><COMMITMENT_1><PROOF_1>...<COMMITMENT_N><PROOF_N>`

## Phase 10. Blame

If a player doesn't receive any proof for any of the commitments the server assigned to hm in phase 8, or if the proof is incorrect, blame is issued: 

Message 10A (from client to server): `<MESSAGE_TYPE><FAULTY_COMMITMENT><COMMITMENT_OF_BLAMER><BLAME_REASON><PRIVATE_KEY>`

The blame reason should one of several blame reasons from a list, including "Missing Proof" or "Incorrect Proof".
 
The player issuing blame should include the `<PRIVATE_KEY>`.  With the private key, the server determines if the proof was sent correctly and whether Alice or Bob is really to blame.
 
The server then responds:

Message 10B (from server to all clients): `<MESSAGE_TYPE><FAULTY_COMMITMENT><BLAME_REASON>`
  
The server will then start over and send a new Message 2, attempting the round again with the same set of players minus the blamed player.

# PART III: Discussion of Protocol Design

The ideal goal of CashFusion is to allow trustless coinjoins with arbitrary amounts and values for inputs and outputs (obviously respecting the rule that the sum of a player's outputs cannot be less than the sum of his inputs), with each player revealing zero knowledge to any other player, or to the server.  (The impetus to prove one's correct behavior here is implied, as it is the key to blame and anti-DOS.)

## Zero Knowledge

Theoretically, a pure ZKP approach is impossible.  Since by definition nothing is revealed about any player's transaction components, there is nothing preventing Alice from colluding with Bob and sharing an input.  In other words, there is no information about any common elements in the sets of components of 2 players.  Thus, exclusion or uniqueness can't be proven in zero knowledge.  

The perfect solution would therefore be some form of multi-party computation (mpc) in which the goal is fuliflled whilst making it computationally infeasible to gain any knowledge.  However, this is a difficult problem and thus as a practical matter, we have instead chosen to create an approximate solution using a clever combination of some more basic tools, including garden variety hash functions.

## The Basic Approach

The approach in CashFusion is to have each player make a cryptographic commitment to each of their transaction components (both inputs and outputs).  When there is a problem with the transaction, each player proves each component individually to one of the other players in the group, chosen randomly.

Each player's list of commitments is a grouping that identifies a player that can be banned, but another key ingredient in CashFusion is that this list is only known to the server.  This prevents the players from learning anything about linkages, no matter how many commitments any of them verify.

The server doesn't participate in the verification process itself, so it also learns nothing.  Some collusion is possible between a malicious server and players it controls, but even in that case, the information leakage is minimal.  We will elaborate on this momentarily.

## Design Trade-Offs

When making design choices in this situation, there is a trade-off between patching tiny security holes vs adding complexity and unreliability in the form of more protocol phases.  One must first understand that even with a perfect fusion protocol, sybill attacks are possible and can be taken to the extreme if the server is malicious.  (Many of the trade-offs center around the server).  When judging the merits of a security trade-off, it is helpful to compare what the security hole allows versus the existing attacking vectors, which may be already larger attack surfaces.   

Returning to the discussion about collusion between a malicious server and a player it controls:  In CashFusion, a player who is randomly verifying other players' transaction components could report all that information to the server, and the server could cross-reference this with its commitment lists from the players, looking for instances of multiple components on the same player's list.

As a result, the server may learn a few random linkages, and this is more than what two colluding players would learn in a perfect protocol.  (Multiple rounds of shuffling and fusion could likely render this kind of attack inert.)  As more colluding players are added in the current protocol, more linkages would be learned, contrasted with more colluding players being added in a perfect protocol, which only reveals more probabalistic taint information.  

Therefore, this imperfection in CashFusion is manifested as only a moderate increase of information leakage to an adverserial group.  Moreover, the additional leakage requires a malicious server, which already has the more powerful weapon of extreme sybilling.  This means it is not worth attempting to patch this hole, particularly when doing so would mean a fundamentally more complex setup that doesn't rely on the server as much.  (See Appendix B).

## MITM Attacks

Another class of security considerations consists of various Man-in-the-Middle (MITM) attacks from the server, designed to reveal more information about the players' transaction components to the server.  

One solution that was implemented is the use of an OP_RETURN output to store some global-state information.    Although using OP_RETURN in this way may be considered inelegant, it broadly handles many problems.  For example, if the server were to maliciously give each player a different public key for the blind signatures, it would be able to group inputs together.  Unfortunately, including the key inside a component commitment doesn't work because we might not reach the blame phase, making the bad behavior undetectable.  An alternative solution to OP_RETURN would be to check the commitments anyway either before or after broadcasting the transaction, but this would make the protocol more complicated and slower.

In addition to the blind signatures, the communication keys (used for encrypting a verification message) could also be spoofed.  Here, the OP_RETURN prevents the server from executing the MITM attack in a non-disruptive manner, which would be most dangerous.  

But if the protocol reaches the blame phase, the server could use previously committed fake transaction components, causing each user to prove all their information to the server.  This would require the server to create a number of transaction inputs, although would take less effort than an extreme sybill attack. 

In the aim of reducing complexity, we allow this attack vector as a limitation for now.  It seems the best way to patch this in a future version will be to add client-side logging to detect high failure rates for servers.  In addition, we can take advantage of the birthday paradox (there is a high probability that any players I'm verifying are also verifying me.)  This makes the attack almost impossible without directly sybilling or making the attack detectable by the absense of birthday collisions between verifiers.

It is also possible for the server to spoof commitments, causing a player to blame a non-existent player, which would reveal the private key of the commitment.  But this is of minor concern since there is only 1 blame per player per round anyway.  The most the server could do would be to learn 1 component from each player per round and attempt to build information based on exclusion over multiple rounds.

If this ever becomes a real concern, it could be handled by adding additional layered encryption between players before blame is issued; the extra complexity is not worth it.

As a final point of discussion regarding MITM attacks, the public keys of the transaction inputs themselves may initially appear to be the best method of authentication, but that doesn't seem to work as it's part of the same information we need to keep hidden. 
 
## Avoiding Amount Linkages Through Combinatorics

Naive multiparty coinjoin schemes are vulnerable to chain analysis based on amount linkages. For example, if you see a joined transaction of (0.5, 0.5, 0.4, 0.7) -> (0.4, 0.6, 0.3, 0.8), this can be uniquely decomposed into (0.5, 0.5) -> (0.4, 0.6) and (0.4, 0.7) -> (0.3, 0.8). (references: https://www.coinjoinsudoku.com/, https://github.com/Samourai-Wallet/boltzmann).

As a result of the above analysis, modern coin shuffling schemes have focused on making equal-amount coins, which intrinsically are indistinguishable (CashShuffle, Wasabi, etc). In isolation, these shuffle schemes are essentially perfect, especially since the cryptographic protocol allows parties to hide information even from each other.

Unfortunately, the production of equal-amount coins is impractical for various reasons. Foremost, it has a "toxic waste" problem: producing identical amounts leaves unmixed and fully-linkable change in the majority of cases. Although the change coin can be successively shaved down in size, this requires many steps and also leaves behind a string of weakly linked shuffled coins. If those shuffled coins are all spent together, the weak linkage becomes a strong linkage (reference: https://cointelegraph.com/news/samourai-wallet-wasabis-coinjoin-management-lacks-privacy).

Moreover, equal-amount outputs provides no easy avenue to private consolidation. A joined consolidation of (0.5, 0.5, 0.4, 0.6) -> (1.0, 1.0), while now providing indistinguishable amounts, has now unfortunately linked the inputs together: (0.5, 0.5) belongs to one party, and (0.4, 0.6) to the other. And, due to the repeated shaving process producing many unspent coins, consolidation is highly desirable. Even simple re-shuffling to increase the small initial anonymity set is difficult since real transactions pay fees; you can't shuffle a 1.0 coin to another 1.0 coin, rather you need a bit more on the input.

In CashFusion, we have opted to abandon the equal-amount concept altogether. While this is at first glance no different than the old naive schemes, mathematical analysis shows it in fact becomes highly private by simply increasing the numbers of inputs and outputs. For example, with hundreds of inputs and outputs, it is not just computationally impractical to iterate through all partitions, but even with infinite computing power, one would find a large number of valid partitions.

Consider a transaction where 10 people have each brought 10 inputs of arbitary amounts in the neighborhood of ~0.1 BCH. One input might be 0.03771049 BCH; the next might be 0.24881232 BCH, etc. All parties have chosen to consolidate their coins, so the transaction has 10 outputs of around 1 BCH. So the transaction has 100 inputs, and 10 outputs. The first output might be 0.91128495, the next could be 1.79783710, etc. 

Now, there are 100!/(10!)^10 ~= 10^92 ways to partition the inputs into a list of 10 sets of 10 inputs, but only a tiny fraction of these partitions will produce the precise output list. So, how many ways produce this exact output list? We can estimate with some napkin math. First, recognize that for each partitioning, each output will typically land in a range of ~10^8 discrete possibilities (around 1 BCH wide, with a 0.00000001 BCH resolution). The first 9 outputs all have this range of possibilities,  and the last will be contstrained by the others. So, the 10^92 possibilies will land somewhere within a 9-dimensional grid that cointains (10^8)^9=10^72 possible distinct sites, one site which is our actual output list. Since we are stuffing 10^92 possibilties into a grid that contains only 10^72 sites, then this means on average, each site will have 10^20 possibilities.

Based on the example above, we can see that not only are there a huge number of partitions, but that even with a fast algorithm that could find matching partitions, it would produce around 10^20 possible valid configurations. With 10^20 possibilities, there is essentially no linkage. The Cash Fusion scheme actually extends this obfuscation even further. Not only can players bring many inputs, they can also have multiple outputs. 

In the future, if desired, the combinatoric effect can be improved; it's easy to 'fuzz' the link between input and output totals by having fees be randomized or having outputs be quantized to more than 1 satoshi resolution.

## Expected usage

Wallets can repeatedly use CashFusion to mix multiple inputs to multiple outputs, to increase the anonymity set. We expect this regular 'churn' of re-mixing to provide a strong foundation of anonymity set and liquidity.

Initially, wallets contain few utxos and so will primarily use Cash Fusion to break these utxos apart. Once a wallet contains more utxos that are delinked, it can start to make Cash Fusions where a few inputs are brought in, but still outputting several coins. At equilibrium, a wallet would contain around 100 coins, and randomly choose ~10 inputs to mix together into ~10 outputs.

# Appendix A.  Proof of Soundness


## Proof of Soundness

We wish to prove that **zero knowledge (beyond what is publicly available) is revealed about any of the players' inputs or outputs** during the fusion process.  It works because the linkages between players' commitments are only kept by the server, and the server doesn't participate in verification.  

Let *X* be the set of a player's transaction components and commitments in this scheme.  Let *a* represent a transaction component and *b* its commitment.  We identify the basic unit of knowledge as showing that 2 components (inputs or outputs) belong to the same player.

a1 ∈ X, a2 ∈ X

A component and its commitment form an ordered pair (a,b).

If two commitments are known to belong to the same player and it also known which components the commitments correspond to, then we can trivially deduce that the two components belong to the same player.  

Given (a1,b1), (a2,b2): (b1 ∈ X, b2 ∈ X) → (a1 ∈ X, a2 ∈ X)

**For the Players:**

Since the distinct list of each player's component committments are kept on the server and not shared with the players, the players have no knowledge of the linkages and cannot state (b1 ∈ X, b2 ∈ X).

Let *Y* be the total set of all the players' components.  Since no subsets (ownership of components) is announced, no knowledge of linkages exist.

(X ⊂ Y, a ∈ Y) → ( a ∈ X = ???)

**For the Server:**

The server knows the linkages of the commitments but does not participate as a verifier and thus doesn't learn the link between the component and its commitment.  The exception is the counterblame phase where one component and its commitment are revealed, but one pairing is insufficient to gain knowledge.

Given only (a1,b1): (b1 ∈ X, b2 ∈ X) → (a1 ∈ X), but (a2 ∈ X = ???)

# Appendix B.  Sketch of an MPC-based Scheme

As mentioned in the discussion section, the ultimate solution would probably be some kind of multi-party computation (mpc).  Although we are not attempting to solve that problem today, we can still contemplate more advanced schemes.  The CashFusion scheme attempts to shard the information between the server and all the players, but it could be improved by relying less on the server.  This may not have a large practical impact, but it is a good area of research and may help create more distributed solutions for the future.

What follows is a hybrid scheme that combines an mpc component with a similar structure to CashFusion:
 
Instead of sharing lists of component commitments with the server, the component commitments would also be announced covertly so that initially, no one knows which player owns a particular commitment.  

For each commitment, a random subset of players (including the actual owner) each produce a random number and join a round of multiparty computation to sum their secrets,  and commit to the numbers under a homomorphic encryption scheme.  A player will choose an even number if he is not the owner, and an odd number if he is.  

The normal fusion verification process is performed, and when a faulty component commitment is found, the non-owner participants of the mcp reveal their commitments along with the homomorphic sum.  By simple subtraction, the owner is revealed to have commited to an odd number.

The dishonest disrupter can attempt to avoid this by not participating in some rounds or falsely submitting the wrong information, but then these commitments will be retried and anti-DOS countermeasures taken, perhaps including another homomorphic operation showing each player committed to ownership of some number of inputs.

In addition, the Pedersen commitments used in CashFusion need to be unbinded from the component commitments.  Instead, the players submit a zero-knowledge proof (ZKP) for each amount that shows the value was commited to in one of the Pedersen commitments, without revealing which commitment.  C(a) = x, x ∈ X, where *a* is the amount, and *X* is the set of all players' commitments.  This requires all inputs and outputs to be unique amounts.
