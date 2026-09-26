# Final cloud architecture decision

## Selected architecture: B — Relay research path

```text
Xbox 360 native XEX
  -> controller / D3D9 / XAudio2 / local UI / bounded transport
  -> user-owned relay protocol
  -> modern, user-authorized relay
  -> official Xbox Cloud Gaming web/client path, only if permitted
```

This is a conditional engineering recommendation, not a claim that Xbox Cloud Gaming works on Xbox 360 today.

## Why not Architecture A (direct Xbox 360 → Xbox Cloud Gaming)

The following required links have not been demonstrated:

1. A supported non-browser Microsoft account/session-creation contract for Xbox 360.
2. Current TLS/interoperability and device/service acceptance using the existing `XboxTLS` stack.
3. ICE/STUN/TURN, DTLS, SRTP/SRTCP, RTP/RTCP, SDP/BUNDLE, and possibly SCTP/WebRTC DataChannel support on XDK/PowerPC.
4. Cloud input transport framing and reconnection rules.
5. Target-hardware H.264 and actual cloud-audio decoding at interactive latency.

Better xCloud proves that the official browser client uses browser-hosted WebRTC boundaries; it does not supply a native client protocol or authorize direct use of the observed endpoints. Current official Xbox Cloud Gaming public guidance names supported browsers/devices and does not list Xbox 360. Direct architecture is therefore **BLOCKED pending evidence**, not merely inconvenient.

## Why relay is the smallest honest path

The relay reduces the Xbox work to capabilities the binary already demonstrates or can reasonably add: controller state, D3D9 presentation, XAudio2 PCM, simple settings/cache, worker queues, TCP/DNS, and a narrow authenticated transport. It consolidates changing browser/service/WebRTC complexity in one maintainable modern component.

It still has two hard gates:

* An authorized, terms-compliant way for the relay to establish and maintain cloud sessions.
* A measured video/audio codec path that the Xbox 360 can consume with low latency.

If either gate fails, the valid outcome is Architecture C: use the same relay/XEX design with a local or third-party backend that offers a documented protocol and a tested codec. Do not substitute mock game sessions or fake catalog/play feedback.

## Delivery sequence

1. Obtain the original GodStix source plus permission/toolchain, or create a clean-room XEX project. Do not binary-patch the supplied store.
2. Complete Prototypes A–D from `LOCAL_STREAM_TEST.md` on physical Xbox 360 hardware.
3. Choose a decoder only after 720p measurements.
4. Build and secure a relay using owned test media.
5. Obtain provider authorization and capture a sanitized live protocol only if service use is permitted.
6. Integrate catalog/session behavior only after the relay can establish a real session.

## Non-negotiable truth conditions

* `Play` is enabled only after a real relay session is allocated.
* `Streaming` is shown only after authenticated media packets and decoder frames arrive.
* Statistics come from timestamps/counters, never defaults.
* Catalog entries must be supplied by a permitted live source and labeled unavailable on failure.
* Authentication tokens/cookies never enter the XEX or user-visible logs.
* Any unproven capability remains `BLOCKED`, `UNKNOWN`, or `REQUIRES RELAY` in UI and documentation.

