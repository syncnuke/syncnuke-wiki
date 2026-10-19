---
layout: default
title: DataSaver
parent: Protocols
grand_parent: Specification
nav_order: 1
permalink: /spec/data-saver/
---
# DataSaver

DataSaver is the dedicated SyncNuke synchronization protocol. It's main goal is to provide reliable synchronization with low communication overhead over connection-oriented
communication.

## Reference behavior

`DataSaverClient` implements protocol `datasaver`, version `0.2.0`. It exchanges playback
state and room membership information while filtering updates that match expected progress.
The current client uses `QuicClient` with `BaseCodec`.

## Session flow

1. After the [master-server connection]({{ '/spec/coordination/' | relative_url }}#establishing-a-connection),
   the client connects to the selected server and sends `JOIN_ROOM` with its username,
   room, and optional password.
2. Local playback changes can produce `UPDATE_STATE` messages. Incoming state updates
   update the client's expected state and are applied through `PlayerManager`.
3. Incoming `JOIN_ROOM` and `LEAVE_ROOM` messages update the user list for the current room.
4. A server `CONNECT` message redirects the client to another host and port. The client
   reconnects and sends `JOIN_ROOM` again with the stored session details.
5. Closing the client sends `LEAVE_ROOM` before releasing the connection.

## State updates

State messages carry play/pause state, position in seconds, playback speed, and an optional filename. They contain
no timestamp; the client records its own current time when updating its expected state.

The client waits for room information before sending state, and for the server state when
joining other participants. The first local observation, filename changes, and changes
between playing and paused trigger an update. Playback updates from the client are filtered
out of local observations.
For other observations, Core applies these filters in order:

- A position difference within [MIN_PROG_CHANGE](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/client/internal/protocol/datasaver/DataSaverClient.java)
  of the stored expected state is suppressed.
- A larger difference while paused, or with no positive elapsed time, triggers an update.
- While playing, the client compares actual progress with elapsed time multiplied by the
  observed playback speed. It sends an update when the relative error exceeds
  [DRIFT_THRESHOLD](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/client/internal/protocol/datasaver/DataSaverClient.java).
  Expected advances that become zero after
  [integer truncation](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/client/internal/protocol/datasaver/DataSaverClient.java) are suppressed.

A speed-only change can therefore be suppressed by the position filter. The latest expected
state is recorded before sending a local update or applying an incoming update.

The default keep-alive interval is 10 seconds, defined by
[DEFAULT_KEEP_ALIVE_INTERVAL_MILLIS](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/client/internal/protocol/datasaver/DataSaverClient.java).
At each scheduled check, if no message has
been sent for that interval and an expected state is available, the client sends an
`UPDATE_STATE` with its projected position. When no expected state is found, a `PING` is sent instead.

Using a state update as the keep-alive keeps the connection active and regularly refreshes
synchronization information. Relying only on change-triggered messages could allow a missed
or suppressed update to cause drift that grows over time. Periodic state updates provide
an ongoing synchronization reference to help prevent that divergence.

## Message format

Messages are consecutive bytes on an ordered stream. The first byte identifies the command;
its fields determine the message length. There is no outer length prefix or separator.

All multibyte values use big-endian byte order. `u16` is an unsigned 16-bit integer;
`f64` is an eight-byte IEEE 754 double. `string` is a `u16` UTF-8 byte count followed by
that many bytes, with a maximum encoded length of 65,535 bytes.

An optional string starts with a one-byte presence marker: `0` means absent; `1` is followed
by a `string`. An empty, present string has marker `1` and length `0`.

Fields follow the command byte in the order shown below.

<table class="protocol-fields">
  <thead>
    <tr><th scope="col">Command</th><th scope="col">Code</th><th scope="colgroup" colspan="4">Fields</th></tr>
  </thead>
  <tbody>
    <tr><th scope="row"><code>CONNECT</code></th><td><code>0x01</code></td><td colspan="2"><code>host</code><br>string</td><td colspan="2"><code>port</code><br>u16</td></tr>
    <tr><th scope="row"><code>UPDATE_STATE</code></th><td><code>0x02</code></td><td><code>state</code><br>byte</td><td><code>position</code><br>f64</td><td><code>playbackSpeed</code><br>f64</td><td><code>fileName</code><br>optional string</td></tr>
    <tr><th scope="row"><code>JOIN_ROOM</code></th><td><code>0x03</code></td><td><code>username</code><br>string</td><td><code>room</code><br>string</td><td colspan="2"><code>password</code><br>optional string</td></tr>
    <tr><th scope="row"><code>LEAVE_ROOM</code></th><td><code>0x04</code></td><td><code>username</code><br>string</td><td><code>room</code><br>string</td><td colspan="2"><code>password</code><br>optional string</td></tr>
    <tr><th scope="row"><code>PING</code></th><td><code>0x05</code></td><td colspan="4">None</td></tr>
  </tbody>
</table>

### Client behavior

- **CONNECT:** Receives a server redirect. The client reconnects to the supplied endpoint
  and rejoins the room; its codec does not encode this command.
- **UPDATE_STATE:** Sends local and keep-alive state updates, and applies incoming server state.
- **JOIN_ROOM:** Sends a room join and receives membership notifications.
- **LEAVE_ROOM:** Sends a departure and receives membership notifications.
- **PING:** The decoder consumes the command without delivering a message to the client.

### Playback state

| State | Code |
| --- | --- |
| `PAUSED` | `0x00` |
| `PLAYING` | `0x01` |

An `UPDATE_STATE` message is 19 bytes with an absent filename; a present filename adds
a two-byte length and its UTF-8 bytes. For example, paused at position zero with normal
speed and an absent filename is:

```text
02 00 00 00 00 00 00 00 00 00 3f f0 00 00 00 00 00 00 00
```

Incoming room messages consume the optional password field but discard its value.
The codec rejects unknown command/state codes, invalid optional-string markers, and
truncated messages. Playback value validation is performed by `PlayerManager`.

The DataSaver `CONNECT` message carries only host and port. It is separate from the master
protocol's `ConnectData`, which also carries the synchronization protocol and version.


## Reference material

SyncNuke Core's [DataSaver command model (internal)](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/client/internal/protocol/datasaver/data/Command.java).

[DataSaverClient](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/client/internal/protocol/datasaver/DataSaverClient.java) and
[BaseCodec](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/client/internal/protocol/datasaver/data/BaseCodec.java)
implement the behavior and message format described above.
