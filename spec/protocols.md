---
layout: default
title: Protocols
parent: Specification
has_children: true
nav_order: 7
permalink: /spec/protocols/
---
# Protocols

A synchronization protocol defines the messages and behavior used to coordinate shared
playback, including state updates and room participation. Protocol implementations use
`SyncClient` to work with SyncNuke's common playback controls.

The transport carries these messages, while a codec handles their encoding and decoding.
See [Networking]({{ '/spec/networking/' | relative_url }}) for transport details.

## Reference behavior

| Protocol | Version |
| --- | --- |
| [DataSaver]({{ '/spec/data-saver/' | relative_url }}) | `0.2.0` |
| Syncplay | `1.7.7` |

`SyncClientFactory` selects the synchronization implementation and checks the requested
protocol version. The selected implementation defines how messages are exchanged and used
to coordinate playback.

The client and server need to support the same protocol and version. Core rejects unsupported
protocol or version selections. See [Establishing a connection]({{ '/spec/coordination/' | relative_url }}#establishing-a-connection)
for how a matching server is selected.

## Reference material

[SyncNuke Core's Protocols](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/client/internal/protocol).
