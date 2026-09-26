# Relay architecture for a real Xbox 360 cloud client

## Decision context

The Xbox 360 can supply native UI, controller polling, TCP/DNS, D3D9 presentation, XAudio2 PCM output, local storage, and worker threads. The reviewed evidence does not establish direct Microsoft authentication, a native WebRTC stack, cloud-session signaling, H.264 decode, or cloud input framing. A relay isolates those unproven/modern requirements from the XEX.

## Recommended research architecture

```text
                 User-authorized Microsoft sign-in
                              |
                       modern relay host
                              |
                   official Xbox web/client flow
                              |
       HTTPS + service-owned session negotiation + WebRTC
                              |
              [relay terminates its authenticated transport]
                         /                         \
  control metrics/status                           media path
        |                                            |
        v                                            v
 Xbox 360 <--- authenticated low-latency protocol --- relay
     |                         |                        |
 Xam input                 H.264/audio packets       bounded buffers
 D3D9 UI                   or decoded/re-encoded      packet loss/RTT
 XAudio2
```

The relay must be a service owned and operated by the user/team. It must use a user-authorized sign-in flow; it must not embed Microsoft credentials in the XEX, scrape passwords, bypass device/region/subscription checks, or claim access where service terms do not allow it.

## Relay responsibilities

| Function | Relay | Xbox 360 |
| --- | --- | --- |
| Microsoft web/auth requirements | Yes, only through an approved/authorized client flow | No tokens or browser cookies. |
| Cloud session and WebRTC | Yes | No in initial architecture. |
| ICE/STUN/TURN/DTLS/SRTP/RTP | Yes | No in initial architecture. |
| Game catalog transformation | Yes, only from a permitted data source | Native display of relay-provided catalog. |
| Stream packet/codec decision | Yes | Consume only a documented relay profile. |
| H.264/video decode | Prefer no; use only if forwarding is impossible | Required unless a verified hardware path changes design. |
| Audio decode | Prefer no; use only if forwarding is impossible | Decode the selected relay audio codec to PCM. |
| Input bridge | Translate a documented Xbox packet into the authorized cloud-client input path | Poll/map controller and send sequence-stamped packets. |
| Statistics and reconnect | Timestamp/report authoritative metrics | Display relay/decoder/transport measurements. |

## Forwarding versus transcoding

### Preferred: codec-aware packet forwarding

```text
cloud WebRTC SRTP
  -> relay decrypts/validates and depacketizes as required
  -> relay sends authenticated relay-media packets
  -> Xbox reorders/reassembles H.264 and decodes
```

This avoids a decode/re-encode stage, but it is viable only if all conditions hold:

1. The relay can obtain authenticated encoded media from the authorized client stack without breaking protocol/security boundaries.
2. The negotiated video codec/profile is accepted by an Xbox 360 decoder that has been demonstrated on hardware.
3. The audio codec can be decoded by the XEX within budget.
4. The relay packet protocol carries parameter sets/configuration, timestamps, keyframe/discontinuity information, loss/reorder semantics, and authenticated encryption.
5. The Xbox’s custom transport is protected against injection/replay and does not expose the upstream media/session keys.

This is not “transparent SRTP forwarding.” Because SRTP is end-to-end between WebRTC peers, the relay must terminate/decrypt it before transforming transport. The Xbox must never receive upstream browser session keys.

### Fallback: relay transcodes

```text
cloud WebRTC -> relay decode -> relay encoder -> Xbox-friendly profile -> Xbox decode
```

Transcoding is technically unavoidable if the upstream codec/profile/audio cannot be consumed on Xbox hardware, or the authorized client provides only decoded frames. It adds encoder queueing, CPU/GPU cost, bitrate change, and typically meaningful latency. Do not use it until forwarding and target decode are measured.

### Browser capture is not a preferred media source

Capturing an HTML video element often yields decoded frames and forces re-encoding. A more efficient relay would use a permitted native/WebRTC endpoint that exposes encoded media or an authorized service integration. Whether either is acceptable to the cloud provider is **UNKNOWN** and must be reviewed before implementation.

## Relay-to-Xbox protocol

Use one narrow, versioned protocol rather than forwarding proprietary Microsoft traffic.

```text
Control channel (TLS or mutually authenticated reliable transport)
  HELLO / version / capability set
  pairing + short-lived relay credential
  catalog/status/error events
  start/stop/reconnect commands
  statistics and clock probes

Media channel (authenticated UDP preferred after Prototype A)
  stream id, monotonic sequence, timestamp, flags
  codec configuration / keyframe / discontinuity markers
  encrypted media fragment
  replay protection and bounded reorder window

Input channel (authenticated UDP or low-latency reliable datagrams)
  controller snapshot, changed-bit mask, client sequence, timestamp
  ACK/RTT tracking; no invented “success” state
```

The relay’s channel crypto must be a maintained implementation compatible with the XDK target. Existing `XboxTLS` is a possible *research input* only; its suitability for current TLS or datagrams is unverified. Never expose cloud bearer tokens or browser cookies to the Xbox.

## Latency and bandwidth

Numbers below are budgets to measure, not guarantees.

| Stage | Packet-forwarding target | Transcoding impact |
| --- | --- | --- |
| Xbox ↔ nearby relay RTT | Keep under ~20 ms where possible | Same. |
| Relay packet transform | Single-digit milliseconds under load | Single-digit packet work plus decode/encode queues. |
| H.264 decode on Xbox | Must fit frame budget | Same. |
| Relay decode + encode | Avoid | Often adds tens of milliseconds or more; measure p95, not just average. |
| Xbox buffer | 1–3 frames target; bounded | May need more to hide transcode jitter, increasing input latency. |
| Total added relay overhead | Aim for <30 ms beyond the upstream session | Could exceed the useful interactive budget; reject rather than hide it. |

Bandwidth depends on relay codec, frame rate, scene complexity, overhead, loss recovery, and quality selection. Do not advertise a fixed Mbps value until the local test harness measures it. The official Xbox page currently recommends roughly 20 Mbps for best console/PC/tablet cloud performance, but this does not establish an Xbox 360 relay requirement or guarantee provider acceptance.

## Security, privacy, and maintenance

* Store only opaque, short-lived relay pairing material on the console; do not persist Microsoft passwords, cookies, or bearer tokens.
* Provide a visible sign-out/revoke flow at the relay. Encrypt data in transit, validate certificates, and bind each media/input stream to a session id and replay window.
* Keep server logs redacted; never log `Authorization` headers, SDP fingerprints plus account data, or raw input beyond operational retention.
* Use rate limits, session caps, and authorization checks. Treat relay as a security-sensitive service, not a packet proxy.
* Microsoft can change the web client, endpoints, media profile, anti-abuse policy, supported devices, or terms. Expect maintenance and a graceful `SERVICE_CHANGED` failure state.

## Existing GodStix reuse plan

If the original GodStix source is obtained with authority to modify it, retain working abstractions only behind tests:

| Existing capability | Reuse approach |
| --- | --- |
| TCP/DNS/XNet | Add `CloudTransport` adapter; first use for relay control, later test UDP. |
| XboxTLS | Keep only after modern TLS/interoperability/security tests; otherwise replace behind same interface. |
| Controller polling | Map `XamInputGetState` into `CloudInput` snapshots. |
| D3D9 UI | Add cloud screens without changing decoder/render ownership rules. |
| XAudio2 | Retain for UI; add separate stream PCM queue/source voice. |
| Worker system | Use bounded media, network, and decode queues; never unbounded downloader-style buffering. |
| Storage/config | Store preferences/cache, not cloud credentials. |

With the supplied binary alone, direct reuse is not safe or supportable. A clean-room application can reproduce interfaces and behavior without patching the old XEX.

