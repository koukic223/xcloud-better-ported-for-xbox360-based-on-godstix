# Local streaming test environment

## Purpose

Validate the Xbox-side transport, buffering, decoder, D3D9 presentation, XAudio2 output, controller input, statistics, and reconnect logic without using Microsoft services or copyrighted game streams. Use self-created test patterns, open sample media, or media the operator has the rights to stream.

## Test topology

```text
PC test host                                      Xbox 360 test XEX
  test-pattern generator / owned sample              CloudTransport
       -> local relay/test server                     -> VideoDecoder
       -> documented relay packets                    -> D3D9 present
       <- controller echo endpoint                    -> AudioDecoder -> XAudio2
                                                   -> real statistics overlay
```

Start with a PC on the same wired Ethernet switch. Do not make internet access, Xbox Cloud Gaming, or an account prerequisite for the first four prototypes.

## Phased prototypes

| Prototype | Scope | Real pass condition | Explicit non-goal |
| --- | --- | --- | --- |
| A — control | Xbox HTTPS to an operator-owned test endpoint | Certificate validation succeeds; endpoint returns a signed capability response; UI shows actual response/error and RTT. | No cloud session claim. |
| B — video | Relay media transport and owned H.264 test clip/pattern | Decoder presents measured frames; overlay reports real received/dropped/decode/present values. | No Microsoft stream. |
| C — input | Xbox controller to relay echo | Relay records sequence/timestamp and returns an acknowledgement; Xbox shows measured loss/RTT. | No game-control claim. |
| D — A/V/input | B+C with PCM/audio test | A/V synchronization, bounded buffering, controller round-trip, reconnect recovery all measured. | No Microsoft integration. |

## Test server requirements

The PC host may use FFmpeg/GStreamer for *local owned test media*, but the server must expose the same logical protocol the XEX will use later:

1. Generate a deterministic video pattern with embedded frame counter and timestamp.
2. Produce a known H.264 profile/level, record SPS/PPS and frame rate.
3. Produce known test audio (for example PCM after server decode) with audio timestamps.
4. Fragment, sequence, timestamp, authenticate, and optionally encrypt data using the future relay framing.
5. Echo controller packets with server receive/send timestamps.
6. Simulate 1–5% loss, reordering, jitter, pause, endpoint restart, and path disconnect.

Avoid testing raw RTP first unless the eventual Xbox protocol will truly be RTP. The important value is end-to-end framing and bounded latency; keeping the local protocol stable makes later relay work comparable.

## Instrumentation contract

The overlay is permitted to display only values with a concrete source:

```text
Resolution:        decoded width x height (or N/A)
FPS:               presented frames / measured interval
Bitrate:           received payload bytes / measured interval
Transport RTT:     acknowledged timestamp round trip (or N/A)
Packet loss:       missing sequence numbers in current stream window
Buffer:            media timestamp head - presented timestamp
Decoder:           selected implementation name + p95 decode duration
Audio queue:       queued PCM duration
```

No value may be hard-coded to `60`, `0 ms`, `Hardware`, `Connected`, or a target bitrate. Before first media packet, show `Idle`; after a real disconnect, show `Disconnected` with the transport error code; during reconnect, show `Reconnecting` only while a real handshake is active.

## Test matrix

| Test | Record | Pass threshold to proceed |
| --- | --- | --- |
| 480p30, clean LAN | decode p50/p95/p99, memory, present rate | 10 minutes stable, bounded queues. |
| 720p30, clean LAN | same plus A/V sync | Stable with measured latency. |
| 720p60, clean LAN | per-frame decode and UI contention | p95 decode/present fits the 16.67 ms frame budget or document failure. |
| 1080p30/60 | CPU, memory, loss, present | Exploration only; do not enable customer-facing option unless passing. |
| 1% loss/reorder | recovery/drop policy | No runaway latency/memory; correct loss counters. |
| relay restart | reconnect time/state | Reconnect attempts are bounded and user-visible. |
| controller soak | sequence gaps/RTT | No stale input applied after reconnection. |

## Preconditions that block execution today

The supplied material is a binary, not a writable C++ project, and no Xbox 360 target kit/hardware session or local relay application is present in this workspace. Therefore this document defines a real test plan; it does not claim that Prototypes A–D have been built or passed.

