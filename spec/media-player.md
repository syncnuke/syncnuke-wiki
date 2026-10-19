---
layout: default
title: Media player
parent: Specification
nav_order: 3
permalink: /spec/media-player/
---
# Media player

Media player integrations implement `MediaPlayer` to expose playback controls and state
observations to SyncNuke Core.

## Reference behavior

The current `MediaPlayer` interface exposes these operations:

| Capability | SyncNuke Core operation |
| --- | --- |
| Start or pause playback | `play()`, `pause()` |
| Change timeline position | `seek(position)` |
| Change playback rate | `setPlaybackSpeed(playbackSpeed)` |
| Load media | `load(file)` |
| Observe current state | `getStatus()` |
| Release resources | Inherited `AutoCloseable` contract |

The reference state distinguishes `PLAYING` and `PAUSED`, with position in seconds, duration in seconds,
playback speed relative to normal speed, media filename and an
observation timestamp in milliseconds since epoch.
A load argument is a string; this does not yet specify a portable media identifier.

## Integration requirements

Every `MediaPlayer` integration MUST support all operations listed above, including playback
speed changes and `close()`. `getStatus()` MUST return the player's observed state. Issuing
a command does not by itself confirm that the player has reached the requested state.

`getStatus()` MUST return a `PlayerState` with these values:

| Field | Required value |
| --- | --- |
| `playbackState` | `PLAYING` or `PAUSED`; must not be null. |
| `position` | Finite, nonnegative position in seconds. |
| `playbackSpeed` | Finite, positive rate; `1.0` is normal playback speed. |
| `lastUpdateTime` | Nonnegative update timestamp in milliseconds since epoch. |

`PlayerState` also carries `fileName` and `duration` metadata, defaulting to null and zero
respectively. Core does not validate these fields.

The returned `PlayerState` itself MUST NOT be null. Integrations need to keep its timestamp
consistent with the observed position, since Core uses elapsed time and playback speed to
project playback progress.

When no media is loaded, `getStatus()` can signal `NoMediaLoadedException`.
The integration's `close()` method releases its resources.

## Player management in Core

`PlayerManager` provides a common way for synchronization clients to observe and control
`MediaPlayer` implementations. This section describes its internal behavior for contributors
working on SyncNuke Core.

### Observing playback

At startup, `PlayerManager` reads an initial state from the player. It then polls `getStatus()`,
copying and validating each observation. The polling interval defaults to
[DEFAULT_POLL_INTERVAL_MILLIS](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/player/internal/PlayerManager.java)
and can be configured through `SyncManager`.

Play/pause, playback speed, filename, and duration changes notify the listener. To detect seeks, the manager
projects the previous position forward using elapsed time and playback speed, then compares
it with the observed position. A difference above
[POSITION_DRIFT_TOLERANCE_SECONDS](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/player/internal/PlayerManager.java)
also triggers a notification. Each valid observation becomes the baseline for the next poll.

If a poll raises `NoMediaLoadedException`, the manager continues polling. The initial read
at startup must succeed before polling begins.

### Applying playback updates

A synchronization client supplies a desired playback state through `updateStatus()`.
`PlayerManager` reads the current player state and issues the play/pause, speed, and seek
commands needed to approach it. Seeks use the same drift tolerance as playback observations.

The requested state is confirmed through subsequent player observations.

### Releasing resources

Replacing a player implementation closes the previous one. Closing `PlayerManager` stops
polling and closes the current player implementation.

See [Coordination]({{ '/spec/coordination/' | relative_url }}#coordination-in-syncnuke-core)
for how `SyncManager` connects the player manager with the synchronization client.

## Reference material

SyncNuke Core's [MediaPlayer](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/player).

[PlayerState](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/player/data/PlayerState.java) and [PlaybackState](https://github.com/syncnuke/syncnuke-core/tree/main/src/main/java/io/github/syncnuke/player/data/PlaybackState.java).
