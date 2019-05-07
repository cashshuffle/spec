# CashShuffle specification

This specification is for version 300 of the CashShuffle protocol.

## Communication

- The server should support TLSv1.2 between the client and server.
- The server should only accept messages on the wire with the following structure:
  1. Magic prefix `0x42bcc32669467873`.
  2. 4-bytes specifying the length of the message in bytes in big endian order.
  3. The protobuf payload.

## Shuffle transaction

In order to avoid client identification and ensure compatibility:

- Transactions should use only ECDSA signing.
- Transactions should use `nLockTime = 0`.
- Transactions should use `nVersion = 1`.
- Each input should use `nSequence = 0xfffffffe`.

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

[Flowchart (work in progress)](https://github.com/Electron-Cash/Electron-Cash/wiki/CashShuffle-Protocol-Flowcharts)

### Entering a Shuffle

After the SSL connection is established, the client should send the server a message containing only the `verification key` of the player and the desired amount to shuffle in satoshis, the type of shuffle and the protocol version. This message should be unsigned.

```
Packets: [
  Signed{
    packet: Packet{
      from_key: VerificationKey{
        key: verification_key
      }
      registration: Registration{
        amount:  bch_in_sats
        version: latest_version
        type:    shuffle_type
      }
    }
  }
]
```

If all is well with the verification key, the server should send back an simple message with a session value and the player's number in the pool.

```
Packets: [
  Signed{
    packet: Packet{
      session: some_UUID
      number:  number_in_pool
    }
  }
]
```

If something goes wrong with the handshake, the server should send a blame message and close the connection.

```
Packets: [
  Signed{
    packet: Packet{
      message: Message{
        blame: Blame{
          reason: blame_reason_enum
        }
      }
    }
  }
]
```

### Forming the pool

The server forms pools of a fixed size `N`. Every player who successfully registers should be added to the current pool and receive a confirmation as above. As long as the pool is not yet full, the server should also broadcast a simple message with the new player's number to all players in the pool, including the new player.

```
Packets: [
  Signed{
    packet: Packet{
      number: new_player_number_in_pool
    }
  }
]
```

After the pool reaches its limit (`N`), instead of the new player broadcast, the server should broadcast that the pool has entered Phase 1 (Announcement) to all pool participants.

```
Packets: [
  Signed{
    packet: Packet{
      phase: ANNOUNCEMENT
      number: pool_size_N
    }
  }
]
```

### Player messaging

After the client is registered as a player (it has a session UUID and number in the pool)
it switches to game mode and can send/receive messages. There are two types of messages `broadcast` and
`unicast`. `broadcast` messages are for everyone in the pool and `unicast` messages are only from one player to another.

Every time the server accepts a message from the client it should check if it has:

- A valid session UUID.
- A valid verification key.
- A valid player number.
- A valid `from_key` field.
- A valid or null `to_key` field where `to_key` must be in the same pool.

If the message has a null value for `to_key` field, it means that it should be broadcast to every pool member
If the message does not have a null value for `to_key` it should be unicast to the specified player.

### Blame messages

In the blame phase, players can exclude unreliable players from the pool.
If server got `N-1` messages from players to exclude a player with verification key `accused_key` then the player with this verification key should be excluded. The message should be in the following form:

```
packet {
  packet {
    session: "session"
    from_key {
      key: "from_key"
    }
    phase: BLAME
    message {
      blame {
        reason: LIAR
        accused {
          key: "excluded_key"
        }
      }
    }
  }
}
Packets: [
  Signed{
    packet: Packet{
      session: session_id
      from_key: VerificationKey{
        key: verification_key
      }
      phase: BLAME
      message {
        blame Blame{
          reason: LIAR
          accused VerificationKey{
            key: accused_key
          }
        }
      }
    }
  }
]
```

### Losing the connection

If one of the players closes their connection during a round of shuffling then the shuffle is terminated and a new shuffle should be initiated by reconnecting to the server.

### Exiting from the Shuffle

If everything goes well, all the clients will disconnect when shuffling is complete.

## Definitions / Glossary

- CoinJoin: A method by which transactions for a Bitcoin-based blockchain can be built to better protect the privacy of the parties involved. CoinJoin builds a transaction so that it uses identical `input` amounts thus normalizing the uniquely identifiable characteristic which `privacy attackers` rely upon to track the owner's of a particular `coin` over time.
- CoinShuffle: A privacy protocol for Bitcoin based blockchains that attempts to achieve the benefits of the `CoinJoin` protocol in a decentralized way that does not rely on a trusted middle man.
- CashShuffle: A companion to the `CoinShuffle` protocol. CashShuffle adds a low-trust infrastructure layer that allows for the discovery of `CashShuffle` participants as well as facilitating anonymous communication between them as they execute the `CashShuffle` protocol to create trustless `CoinJoin` transactions.


- Blame: This may describe either the `Phase` immediately following a failed `CashShuffle` `Round` or the specific `Protocol Message` that is sent by a `Player` to signal a failed `Round`. The `Blame` `Protocol Message` takes a specific form. See the `Server Message` docs for more info.
- Client: Any device or application that connects to a `CashShuffle` server with the intention and capability of carrying out the steps in the `CashShuffle` protocol. Clients manage most of the shuffling complexity thus minimizing the influence and involvement of the `Server`.
- Coin: Frequently used as shorthand for an `Output` in a Bitcoin transaction.
- Connection: The underlying network socket between `Client` and `Server` used to exchange `Protocol Messages`. Note, `Connections` have a 1:1 relationship with a `Player`.
- Equivocation:
- Packet: A specific data type used in the formation of a `Protocol Message`. See the communication spec for syntax.
- Phase: Describes each of the stages within a `CashShuffle` `Round`. For complete descriptions see the "flow specification"
- Player: The unique representation of a `Client` within a `Pool`. A `Client` may participate simultaneously in multiple `Pools`. It may be helpful to think of a `Player` as being linked to its ephemeral keypair ( `Verification Key` and `Signing Key` ) because all three die at the end of a `Round`, regardless of its outcome.
- Pool: A group of `Players` within the `Server` that participate in a `Round`. `Players` are grouped based on the value of the `Coin` they intend to shuffle and the type of shuffle they intend to perform. While `Clients` may have multiple `Players` in different `Pools`, no client should be allowed in the same `Pool` instance as it would reduce the effectiveness of the `CashShuffle` protocol.
- Protocol Message: A standardized message sent by either the `Server` or a `Player` in a shuffle `Round` containing data relevant to the state of a shuffle `Round`. Protocol messages must adhere to the `Protobuf` communcation spec outlined in this document.
- Round: An abstraction describing the sequence of events that take place in a single attempt by a `Pool` of `Players` to shuffle their coins in adherence to the `CashShuffle` protocol.
- Server: An intentionally minimal server whose primary duties are to put `Clients` into `Pools` and act as a blind ( minimal logs ) and dumb ( minimal complexity ) relay of `Protocol Message` during shuffles.
- Signing Key: The private key portion of the ephemeral keypair created to be used exclusively for signing and verifying protocol messages for a particular `CashShuffle` `Round`. The `Verification Key` and its corresponding `Signing Key` are both intended for one time use.
- Verification Key: The public key portion of the ephemeral keypair created to be used exclusively for signing and verifying protocol messages for a particular `CashShuffle` `Round`. The `Verification Key` and its corresponding `Signing Key` are both intended for one time use.

