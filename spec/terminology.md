---
layout: default
title: Terminology
parent: Specification
nav_order: 2
permalink: /spec/terminology/
---
# Terminology

These definitions describe the terms used throughout the SyncNuke documentation.

| Term | Meaning |
| --- | --- |
| Sync client | Component that coordinates playback using a synchronization protocol. |
| Player integration | Adapter between a media player and SyncNuke playback controls and observations. |
| Playback state | Whether playback is playing or paused in the current reference model. |
| Position | Location within the media timeline. |
| Playback rate | Speed relative to normal playback. |
| Room | Named group of participants, represented by its name and user list in `RoomInfo`. |
| Protocol | Rules and message representation for synchronization communication. |
| Transport | Mechanism carrying protocol messages. |
| Observed state | Playback state returned by the media player. |
| Desired state | Playback state provided to the media player. |
| Expected state | Playback state that the sync client believes the server expects. |
| Codec | Component that encodes protocol messages into bytes and decodes received bytes into messages. |
| Keep-alive | Periodic communication used to maintain a connection. DataSaver uses playback-state updates for this, or `PING` if the expected state is not yet available. |
| Server endpoint | Host and port identifying the server a client connects to. |

## Reference material

SyncNuke Core's [PlayerState](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/player/data/PlayerState.java).

[RoomInfo](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/client/data/RoomInfo.java).
