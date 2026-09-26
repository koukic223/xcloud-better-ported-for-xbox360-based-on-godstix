# xCloud streaming protocol: evidence ledger

## Scope

This document traces what Better xCloud actually observes at commit `f8397043f6d2148d2345d508902a38c69cf1ee20`. Better xCloud runs *inside* the official Xbox web app, so it exposes browser-visible boundaries rather than a complete Microsoft protocol specification. Microsoft has not supplied a public native Xbox 360 client contract in the reviewed material.

Status labels are deliberate:

* **Confirmed from source** — directly represented in the Better xCloud or recovered GodStix source/binary evidence.
* **Confirmed from runtime/network evidence** — not available in this review; none is claimed.
* **Inferred** — follows from a named standard/API, but exact service configuration is not visible.
* **Unknown** — requires an authorized live capture or official documentation.

## Observed end-to-end path

```text
Browser loads xbox.com/play
  -> Microsoft-hosted sign-in page and browser session state
  -> official web app declares signed-in state
  -> host request: /v2/login/user
  -> response carries gsToken + regional offering settings
  -> official UI selects a catalog product
  -> host request: POST .../sessions/cloud/play
       Better xCloud may change settings/device metadata/locale
  -> host request: session configuration and GET .../sessions/.../ice
       Better xCloud may reorder ICE candidates
  -> official web app owns RTCPeerConnection offer/answer and connection
  -> browser renders MediaStream in HTMLVideoElement and Web Audio path
  -> official input sink sends gamepad state; Better xCloud can intercept it
```

Better xCloud does **not** show the whole sequence from its own code because it delegates authentication, selection, launch, signaling, media decode, and most input transport to the official web application.

## Transport and session evidence

| Layer / concern | Status | Evidence and bound on the claim |
| --- | --- | --- |
| Microsoft account authentication | Confirmed from source | Better xCloud detects `window.xbcUser.isSignedIn` and handles the host `/auth/msa` navigation. The OAuth/identity parameters, cookies, MFA and anti-abuse flow are host-owned and not reproduced. |
| HTTPS | Confirmed from source | Browser `fetch`/XHR contacts host service; Better xCloud observes `/v2/login/user`, title/wait-time helpers, play/configuration/ICE requests. Exact TLS versions/ciphers are browser/service negotiated and unknown. |
| `gsToken` bearer use | Confirmed from source | `XcloudApi` uses `Authorization: Bearer ${gsToken}` for observed title and wait-time helpers. Token issuance/expiry/claims are unknown. |
| Game catalog | Confirmed from source | The host uses catalog URLs, including `catalog.gamepass.com/sigls/...`; Better xCloud modifies some results. Full product/catalog contract is unknown. |
| Session creation | Confirmed from source | Better xCloud intercepts a host `POST` ending in `/sessions/cloud/play` and reads/modifies an existing JSON settings body. Required fields, response schema, and provider authorization checks are unknown. |
| Session configuration | Confirmed from source | An endpoint ending in `/configuration` returns `clientStreamingConfigOverrides`, which Better xCloud edits. Full configuration schema is unknown. |
| ICE candidate exchange | Confirmed from source | A session `/ice` GET response contains `exchangeResponse`, JSON-encoded `a=candidate` strings and `a=end-of-candidates`; Better xCloud can reorder candidates. |
| STUN / TURN | Inferred | ICE normally uses STUN/TURN as needed. The source does not expose server URLs, relay allocation behavior, or candidate types from a live session. |
| WebRTC peer connection | Confirmed from source | Better xCloud monkey-patches `RTCPeerConnection`, its local SDP, and created data channels. The browser implements the protocol. |
| SDP | Confirmed from source | It rewrites local SDP: reorders H.264 payload formats in `m=video` and adjusts `b=AS` bitrate. The captured SDP is not supplied. |
| DTLS | Inferred | The source’s SDP comment format is `UDP/TLS/RTP/SAVPF`, and browser WebRTC is used. DTLS version, certificate type/fingerprint, and cipher suites are unknown. |
| SRTP/SRTCP | Inferred | WebRTC media transport and `UDP/TLS/RTP/SAVPF` imply protected RTP in a conforming browser session. Exact SRTP profile and key lifecycle are unknown. |
| RTP/RTCP | Inferred | `RTCInboundRtpStreamStats` and video packet/loss metrics identify RTP media statistics. Payload types, RTCP feedback, retransmission, FEC, and pacing configuration are unknown. |
| H.264 video preference | Confirmed from source | Better xCloud searches SDP `profile-level-id` and can prioritize `42e`, `420`, or `4d` profile prefixes. Service acceptance and exact level are unknown. |
| Audio codec | Unknown | Better xCloud handles decoded `MediaStream` audio but does not enumerate codec parameters. Do not label it Opus or AAC until an authorized SDP or RTC codec capture proves it. |
| Controller input | Confirmed from source at API boundary | It sees a host `inputSink.onGamepadInput(...)` and typed `sendGamepadInput(...)`. The serialization, cadence, reliability choice, and underlying data channel are unknown. |
| `message` data channel | Confirmed from source | Better xCloud listens to a channel labeled `message` for title information. That does not prove controller input uses the same channel. |
| Resolution selection | Confirmed from source | Better xCloud changes device-info/settings (`android`, `windows`, `tizen`) to influence requested resolution. Actual selected rendition remains server/browser negotiated. |
| Bitrate control | Confirmed from source | It inserts/replaces SDP `b=AS` video/audio limits. Actual congestion control, server cap, and applied bitrate are unknown. |
| Stream stats | Confirmed from source | It reads `getStats()` for frames, RTP loss, jitter, decode time, candidate-pair RTT, and bytes. |
| Keep-alive | Unknown for cloud session | A Remote Play keep-alive patch exists elsewhere, but this review does not establish a cloud-session keep-alive protocol. |
| Reconnect | Unknown | Better xCloud handles UI state; it does not document a native reconnection protocol or resume token. |
| WebSocket signaling | Unknown | The reviewed source does not expose a WebSocket as the cloud-session signaling transport. Do not assume one. |

## Required replacement on Xbox 360

| Browser-provided function | Direct Xbox 360 equivalent today | Required work |
| --- | --- | --- |
| Modern sign-in, cookies, MFA | None in supplied XEX | Do not replicate with embedded credentials. Use a relay-mediated, user-authorized flow if allowed. |
| Fetch/XHR/JSON/DOM | Raw sockets + existing HTTPS only | Native HTTP/JSON is feasible; service contract is not validated. |
| RTCPeerConnection | None | ICE, DTLS, SRTP/SRTCP, RTP/RTCP, SDP/JSEP, UDP transport, media feedback, and likely SCTP must be supplied or terminated at relay. |
| Browser H.264 decode to `HTMLVideoElement` | None proved | Find an actual licensed decoder path, then present decoded surfaces with D3D9. |
| Browser decoded audio/AudioContext | XAudio2 output only | Decode the actual negotiated codec to PCM, then feed XAudio2. |
| Gamepad API and host input sink | `XamInputGetState` | New mapping and a documented authenticated relay input protocol. |
| `getStats()` | None | Define measurements from real packets/decoder/relay timestamps. |

## What must be captured before a direct-client claim

Perform this only using an account/session and traffic inspection authorization held by the operator:

1. Record browser DevTools/NetLog metadata for one successful cloud session without storing tokens or personal data.
2. Capture sanitized offer/answer SDP: media sections, codecs, payload types, ICE candidate types, DTLS fingerprint algorithm, RTCP feedback, and BUNDLE/mid arrangement.
3. Capture the host session request/response schemas with secrets redacted, and confirm whether a supported public client flow exists.
4. Identify the input transport by correlating a controller action with data-channel/transport activity; do not infer it from the `message` channel.
5. Measure first-frame time, steady bitrate, RTT, loss, jitter, video profile, audio codec, and reconnect behavior.
6. Repeat after service/site updates. These browser-facing endpoints are not stable APIs.

## Protocol conclusion

The confirmed service boundary is **HTTPS + browser-hosted WebRTC session establishment**, with browser WebRTC responsible for ICE/SDP/DTLS/SRTP/RTP and browser media APIs responsible for decode and presentation. A direct Xbox 360 client must replace every one of those browser capabilities and also satisfy a currently unverified Microsoft authentication/session contract. This evidence supports a relay research path; it does not support claiming direct xCloud compatibility.

