# CashShuffle specification

## Communication

- The server should support TLSv1.2 between the client and server.
- The server should only accept messages on the wire with the following structure:
  1. Magic prefix `0x42bcc32669467873`.
  2. 4-bytes specifying the length of the message in bytes in big endian order.
  3. The protobuf payload according to protobuf specification.

## Protobuf payload

- The client and server should communicate according to the [protobuf specification](https://github.com/Electron-Cash/Electron-Cash/blob/master/plugins/shuffle/protobuf/message.proto) for messages.
- The server should only receive messages of the `Packets` type.

Example payload content for client registering with the server:

```
Packets: [
  Signed{
    packet: Packet{
      registration: Registration{
        amount:  one_bch_in_sats
        version: latest_version
        type:    default_shuffle_type
      }
    }
  }
]
```

Example payload serialized bytes:

```
Work in Progress
```

## Flow

- [Flowchart (work in progress)](https://github.com/Electron-Cash/Electron-Cash/wiki/CashShuffle-Protocol-Flowcharts)
- Additional flow specifications (work in progress)
- Example messages for each step in flowchart (work in progress)

## Definitions / Glossary

- CoinJoin: A method by which transactions for a Bitcoin-based blockchain can be built to better protect the privacy of the parties involved.  CoinJoin transactions aim to obscure the chain of ownership of the `coins` contained within.  This is done by building the transactions so that they use identical `input` amounts thus normalizing the uniquely identifiable characteristic which `privacy attackers` rely upon to track the owner's of a particular `coin` over time. 
- CoinShuffle: A privacy protocol for Bitcoin based blockchains that attempts to achieve the benefits of the earlier `CoinJoin` protocol in decentralized way that doesn't rely on a trusted middle man.
- CashShuffle: A companion to the `CoinShuffle` protocol. CoinShuffle adds a trustless infrastructure layer that allows for the discovery of `CoinShuffle` participants as well as facilitating anonymous communication between them as they execute the `CoinShuffle` to create trustless `CoinJoin` transactions.
- Coin: Frequently used as shorthand for an `Output` to Bitcoin transaction.
- Client: Any device or application that connects to a `CashShuffle` server with the intention and capability of carrying out the steps in the `CoinShuffle` protocol. Clients manage most of the shuffling complexity thus minimizing the necessary role of the `Server`.
- Server: An intentionally minimal server who's primary duties are to put `Clients` into `Pools` and acts as a blind ( minimal logs ) and dumb ( minimal complexity ) relay of `Protocol Message` during shuffles.
- Connection: The underlying network socket by the `Client` to exchange `Protocol Message`s in a particular shuffle `Round`.  Note, Connections have a 1:1 relationship with a `Player` ( and by extension a `Round` )
- Player: The representation of a user within a single `Round`. A `Client` may be involved in multiple simultaneous shuffles at any time. Their presence in each of those is represented by a unique `Player`.  It may be helpful to think of a `Player` as being linked to it's ephemeral keypair ( `Verification Key` and `Signing Key` ) because all three die at the end of a `Round`, regardless of it's outcome.
- Verification Key: The public key portion of the ephemeral keypair created to be used exclusively for signing and verifying protocol messages for a particular CashShuffle `Round`.  The `Verification Key` and it's corresponding `Signing Key` are both intended for one time use.
- Signing Key: The private key portion of the ephemeral keypair created to be used exclusively for signing and verifying protocol messages for a particular `CashShuffle` `Round`.  The `Verification Key` and it's corresponding `Signing Key` are both intended for one time use.
- Round: An abstraction describing the sequence of events that take place in a single attempt by a group of `Players` to shuffle their coins in adherence to the `CoinShuffle` protocol.  Note, the term isn't perfectly analogous to shuffle because there may be many failed Rounds before a successful shuffle occurs.
- Pool: A grouping of `Players` within the `Server` and between which a shuffle `Round` will take place. `Players` are grouped based on the value of the `Coin` they intend to shuffle.  While `Clients` may have multiple `Players` in `Pools` of the same value, no client should be allowed in the same `Pool` instance as it would reduce the effectiveness of the `CoinShuffle` protocol.
- Blame: This may describe either the `Phase` immediately following a failed `CoinShuffle` `Round` or the specific `Protocol Message` that is sent by a `Player` to signal a failed `Round`.  The `Blame` `Protocol Message` takes a specific form.  See the `Server Message` docs for more info.
- Protocol Message: A standardized message sent by either a `Player` in a shuffle `Round` or the shuffle `Server` containing data relevant to the state of a shuffle `Round`.  Protocol messages must adhere to the `Protobuff` communcation spec outlined in this document.
- Phase: Describe one of three possible stages within a `CoinShuffle` `Round`.  For complete descriptions see the "flow specification"
- Packet: A specific data type used in the formation of a `Protocol Message`. See the communication spec for syntax.
- Equivocation: 

