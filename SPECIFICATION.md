# CashShuffle specification

## Communication

- The server should support TLSv1.2 between the client and server.
- The server should only accept messages on the wire with the following structure:
  1. Magic prefix `0x42bcc32669467873`.
  2. 4-bytes specifying the length of the message in bytes in big endian order.
  3. The protobuf payload.

## Protobuf payload

- The client and server should communicate according to the [protobuf specification](https://github.com/Electron-Cash/Electron-Cash/blob/master/plugins/shuffle/protobuf/message.proto) for messages.
- The server should only receive messages of the `Packets` type.

Example payload content for client registering with the server:

```
Packets: [
  Signed{
    packet: Packet{
      registration: Registration{
        amount:  bch_in_sats
        version: latest_version
        type:    shuffle_type
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

- CoinJoin: A method by which transactions for a Bitcoin-based blockchain can be built to better protect the privacy of the parties involved. CoinJoin builds a transaction so that it uses identical `input` amounts thus normalizing the uniquely identifiable characteristic which `privacy attackers` rely upon to track the owner's of a particular `coin` over time.
- CoinShuffle: A privacy protocol for Bitcoin based blockchains that attempts to achieve the benefits of the `CoinJoin` protocol in a decentralized way that does not rely on a trusted middle man.
- CashShuffle: A companion to the `CoinShuffle` protocol. CashShuffle adds a low-trust infrastructure layer that allows for the discovery of `CashShuffle` participants as well as facilitating anonymous communication between them as they execute the `CashShuffle` protocol to create trustless `CoinJoin` transactions.
- Coin: Frequently used as shorthand for an `Output` in a Bitcoin transaction.
- Client: Any device or application that connects to a `CashShuffle` server with the intention and capability of carrying out the steps in the `CashShuffle` protocol. Clients manage most of the shuffling complexity thus minimizing the influence and involvement of the `Server`.
- Server: An intentionally minimal server whose primary duties are to put `Clients` into `Pools` and act as a blind ( minimal logs ) and dumb ( minimal complexity ) relay of `Protocol Message` during shuffles.
- Connection: The underlying network socket between `Client` and `Server` used to exchange `Protocol Messages`. Note, `Connections` have a 1:1 relationship with a `Player`.
- Player: The unique representation of a `Client` within a `Pool`. A `Client` may participate simultaneously in multiple `Pools`. It may be helpful to think of a `Player` as being linked to it's ephemeral keypair ( `Verification Key` and `Signing Key` ) because all three die at the end of a `Round`, regardless of it's outcome.
- Verification Key: The public key portion of the ephemeral keypair created to be used exclusively for signing and verifying protocol messages for a particular `CashShuffle` `Round`. The `Verification Key` and it's corresponding `Signing Key` are both intended for one time use.
- Signing Key: The private key portion of the ephemeral keypair created to be used exclusively for signing and verifying protocol messages for a particular `CashShuffle` `Round`. The `Verification Key` and it's corresponding `Signing Key` are both intended for one time use.
- Round: An abstraction describing the sequence of events that take place in a single attempt by a `Pool` of `Players` to shuffle their coins in adherence to the `CashShuffle` protocol.
- Pool: A group of `Players` within the `Server` that participate in a `Round`. `Players` are grouped based on the value of the `Coin` they intend to shuffle and the type of shuffle they intend to perform. While `Clients` may have multiple `Players` in different `Pools`, no client should be allowed in the same `Pool` instance as it would reduce the effectiveness of the `CashShuffle` protocol.
- Blame: This may describe either the `Phase` immediately following a failed `CashShuffle` `Round` or the specific `Protocol Message` that is sent by a `Player` to signal a failed `Round`. The `Blame` `Protocol Message` takes a specific form. See the `Server Message` docs for more info.
- Protocol Message: A standardized message sent by either the `Server` or a `Player` in a shuffle `Round` containing data relevant to the state of a shuffle `Round`. Protocol messages must adhere to the `Protobuf` communcation spec outlined in this document.
- Phase: Describes each of the stages within a `CashShuffle` `Round`. For complete descriptions see the "flow specification"
- Packet: A specific data type used in the formation of a `Protocol Message`. See the communication spec for syntax.
- Equivocation: 

