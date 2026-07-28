# moonlight-common-c (Caracal fork)

A fork of [moonlight-stream/moonlight-common-c](https://github.com/moonlight-stream/moonlight-common-c)
carrying the protocol changes that the Caracal pair depends on.

This repository is a dependency, not a product. It is consumed as a git submodule by:

- **[Serval](https://github.com/RootBloader/Serval)** — the macOS client. Compiles this
  library in full and is the only consumer of the patches below.
- **[Caracal](https://github.com/RootBloader/Caracal)** — the Windows host, a Sunshine fork.
  Compiles only `src/Input.h`, `src/Rtsp.h`, `src/RtspParser.c`, `src/Video.h`, `nanors/`
  and `enet/` from this tree, none of which the patches touch. It tracks this fork for a
  single source of truth, not because the patches change its build.

Both of those repositories are private, so those links will 404 unless you have access.

## Branches

| Branch | Contents |
| --- | --- |
| `master` | Untouched mirror of upstream. Nothing of ours lands here. |
| `caracal` | Upstream plus the three commits below. Default branch; this is what the submodules pin. |

The patches are based on upstream `703a069` ("Bump enet from `0eb84dc` to `aca8784`", #144).

Upstream's `.github/workflows/` is removed on `caracal` — it built and linted for platforms
this pair does not target, against an upstream release cadence that is not ours. It is still
present on `master`.

Upstream's own warning still applies: the bundled ENet submodule is a *specific* fork with
breaking API/ABI changes for IPv6 and retransmission reliability. Linking against a system
ENet will crash at connect time. Use the submodule.

---

## The changes

### 1. Fixed Caracal transport ports — `5f2f8c9`

GameStream scatters its traffic across seven historical ports (TCP 47984/47989/48010,
UDP 47998/47999/48000/48010, plus 47996 for the Gen-3 first-frame trick). Caracal is one
host talking to one client, so it uses five fixed, contiguous ports and drops the probing
that existed to cope with the old scheme.

**New file — `src/CaracalPorts.h`**

| Constant | Port |
| --- | --- |
| `CARACAL_API_PORT` | 9411 |
| `CARACAL_RTSP_PORT` | 9412 |
| `CARACAL_CONTROL_PORT` | 9413 |
| `CARACAL_VIDEO_PORT` | 9414 |
| `CARACAL_AUDIO_PORT` | 9415 |

**`src/Limelight.h`** — the public port tables are renumbered onto that range. The seven
`ML_PORT_INDEX_*` and `ML_PORT_FLAG_*` entries collapse to five (TCP 9411, TCP 9412,
UDP 9413, UDP 9414, UDP 9415). The `ML_ERROR_NO_VIDEO_TRAFFIC` comment now names UDP 9414
rather than 47998.

**`src/Connection.c`** — when RTSP port parsing fails, the fallback is `CARACAL_RTSP_PORT`
instead of the well-known 48010. Host name resolution no longer walks 47984 → 47989 → 48010
looking for whichever port a given GameStream generation happens to be listening on; it
resolves the one RTSP port, and retries once after 1 s because launch can answer just
before the RTSP listener finishes binding.

**`src/VideoStream.c`** — the Gen-3 first-frame socket is removed entirely: `FIRST_FRAME_PORT`
(47996), `FIRST_FRAME_MAX`, `firstFrameSocket` and `readFirstFrame()`. That socket existed
only so that closing it would kick video loose on Gen-3 servers. Caracal is never a Gen-3
host, so the socket was opened and closed for nothing.

**`src/SdpGenerator.c`** — the SDP tail advertises `VideoPortNumber` unconditionally,
dropping the `AppVersionQuad[0] < 4 ? 47996 : VideoPortNumber` generation check.

**`src/PlatformSockets.c`** — the 3DS UDP port constant follows `CARACAL_VIDEO_PORT`.

**`src/ConnectionTester.c`, `src/RtspConnection.c`, `src/ControlStream.c`,
`src/Limelight-internal.h`** — follow the new constants.

Net effect: 147 lines removed, 58 added. Most of this commit is deletion.

### 2. Cursor Sync control packets — `71cb987`

Adds a Caracal-only control message so the host and client can agree on where the pointer
is and which side currently owns it.

**Wire format.** Packet type `0x7f20`, chosen to sit outside upstream's packet-type table so
that a host which does not understand it ignores it rather than misreading it as something
else. The receive path rejects any packet whose payload length is not exactly
`sizeof(LI_CURSOR_SYNC_EVENT)` before dispatching it.

**New public struct — `LI_CURSOR_SYNC_EVENT`** (20 bytes, little-endian on the wire):

| Field | Type | Meaning |
| --- | --- | --- |
| `kind` | `uint8_t` | Position update, or an ownership transfer |
| `flags` | `uint8_t` | Per-kind modifiers |
| `edge` | `uint8_t` | Which screen edge the pointer crossed, when relevant |
| `reserved` | `uint8_t` | Zero |
| `generation` | `uint32_t` | Bumped when ownership changes, so stale positions can be dropped |
| `sequence` | `uint32_t` | Orders position updates within a generation |
| `x`, `y` | `uint32_t` | Position, normalised to 0…65535 with a top-left origin, matching Windows `GetCursorPos` and the absolute mouse path |

**Two new control channels** in `src/Limelight-internal.h`: `CTRL_CHANNEL_CURSOR_STATE`
(`0x07`) and `CTRL_CHANNEL_CURSOR_OWNER` (`0x08`).

**New API** in `src/Limelight.h`:

```c
int LiSendCursorSyncEvent(uint8_t kind, uint8_t flags, uint8_t edge,
                          uint32_t generation, uint32_t sequence,
                          uint32_t x, uint32_t y);
```

Position updates go out unreliable/sequenced — a dropped one is superseded a frame later and
is not worth a retransmit. Ownership changes go out reliable. Returns `-2` if the control
stream is not up, or if the host reports a major version below 5.

**New callback** `ConnListenerCursorSync cursorSync` on `CONNECTION_LISTENER_CALLBACKS`, with
a no-op added to `src/FakeCallbacks.c` so a caller that never sets it stays safe.

### 3. Shorter control stream disconnect timeout — `a2f0133`

`src/ControlStream.c`: `enet_peer_timeout()` drops from 10 s to 5 s on non-3DS builds.

ENet starts this clock at the first reliable send that goes unacknowledged and pings every
500 ms, so the host has to go quiet for 5–5.5 s before the session is declared dead. That is
still far longer than a Wi-Fi roam or an AWDL burst, and it halves the wait before Serval's
stream window stops and offers Reconnect. The retransmit cap moves with it
(`timeoutMaximum / 5`), so retransmits go out every second instead of every two — which
helps a stream that is stumbling rather than actually gone.

---

## Updating from upstream

```bash
git fetch upstream
git checkout master && git merge --ff-only upstream/master && git push origin master
git checkout caracal && git rebase master
```

Expect conflicts in `src/Limelight.h` and `src/ControlStream.c` — those are where both
upstream and the patches above are most active.
