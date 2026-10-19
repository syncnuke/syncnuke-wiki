---
layout: default
title: Sync client
parent: Specification
nav_order: 4
permalink: /spec/sync-client/
---
# Sync client

Synchronization protocol implementations extend `SyncClient` to connect playback
observations and controls with a synchronized session.

## Implementation responsibilities

Each implementation supplies these methods:

| Operation | Responsibility |
| --- | --- |
| `getNetClient()` | Return the transport client used for protocol communication. |
| `login(username, room, password)` | Implement the protocol's login and room-joining behavior. |
| `getRoomInfo()` | Return room, participant, and media information as `RoomInfo`. |
| `onStatusChange(status)` | Handle observed local playback changes and decide what to send. |
| `handleResponse(data)` | Interpret incoming protocol messages and apply their effects. |
| `sendKeepAlive()` | Supply the protocol's periodic keep-alive communication. |
| `onClose()` | Supply protocol-specific departure messages and cleanup. |

The constructor passes a `PlayerManager` and a keep-alive interval in milliseconds to the
base class. The implementation supplies its transport and codec, then calls
`connect(host, port, codec)` to establish communication.

### Working with playback state

Use `getPlayerStatus()` for live playback observations and `updatePlayerStatus()` to queue
player controls. The base class runs these operations and
tracks pending updates, which implementations can inspect through `hasPendingPlayerUpdate()`
to filter observations caused by their own commands.
The implementation decides how to filter local changes, represent them as messages, and
interpret incoming updates. Any expected state used to track what the server expects belongs to the
protocol implementation; the base class does not maintain it.

See [Player management in Core]({{ '/spec/media-player/' | relative_url }}#player-management-in-core)
for observation and control behavior, and [Coordination]({{ '/spec/coordination/' | relative_url }}#playback-information-flow)
for the flow between components.

### Room information

`RoomInfo` contains a room name, a list of users and a `files` map from usernames to `MediaInfo`, and a room-level `fileName`. `MediaInfo` contains
a filename and duration in seconds. Available media information depends on the protocol.
The implementation maintains this information as participants and media change. The base
class does not store membership information or interpret room messages.

### Sending and receiving

Use the inherited `send(data)` method for outgoing messages so queued sends update the
base class's keep-alive tracking. Incoming decoded messages reach `handleResponse(data)`
through the listener registered by `connect()`.

The base class serializes incoming responses and requests through
`SyncEventService`. Implementations use `executeEvent()` for local state callbacks, login,
and other state changes, and `readState()` for consistent state reads.

See [Establishing a connection]({{ '/spec/coordination/' | relative_url }}#establishing-a-connection)
for how Core selects and starts a synchronization implementation.

## Client management in Core

This section describes the behavior supplied by the base class for contributors working
on SyncNuke Core.

### Connection and message dispatch

`connect()` delegates connection setup to `getNetClient()` with the host, port, and codec,
then registers `handleResponse()` as a receive listener. The transport performs encoding
and decoding through that codec.

`send()` ignores null messages. For other messages, it queues the transport send and records
the current time. A failed network operation is logged and closes the sync client.

### Keep-alive scheduling

On the first `connect()`, a positive constructor interval schedules checks at that interval,
with the first check occurring after one interval. Zero or a negative interval disables scheduling.

At each check, the base class compares the current time with the last queued outgoing
send. Once the elapsed time reaches the interval, it calls `sendKeepAlive()`. Incoming
messages do not reset this timer. The callback defines the message to send; for example,
[DataSaver]({{ '/spec/data-saver/' | relative_url }}#state-updates) sends a projected playback state.

The base class catches and logs exceptions from the scheduled callback so later checks can
continue. `getCurrentTime()` obtains milliseconds from the supplied `TimingService`; the
constructor overload accepting that service allows timing to be controlled in tests.

### Releasing resources

`close()` cancels the keep-alive task and closes the transport, logging exceptions from
transport cleanup. An implementation with protocol-specific departure messages or resources
should implement `onClose()` to define its own clean-up requirements.

Closing a sync client does not close its supplied player manager. See
[Lifecycle]({{ '/spec/coordination/' | relative_url }}#lifecycle) for how Core manages their lifetimes.

## Reference material

SyncNuke Core's [SyncClient](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/client/SyncClient.java) and
[RoomInfo](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/client/data/RoomInfo.java).

[DataSaverClient](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/client/internal/protocol/datasaver/DataSaverClient.java)
provides a concrete synchronization implementation.
