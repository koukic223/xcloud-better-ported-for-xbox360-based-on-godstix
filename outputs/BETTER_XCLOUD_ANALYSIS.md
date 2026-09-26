# Better xCloud analysis

## Revision analyzed

| Item | Value |
| --- | --- |
| Repository | `redphx/better-xcloud` |
| Commit | `f8397043f6d2148d2345d508902a38c69cf1ee20` |
| Commit date/message | 2026-07-14, `chore: bump version to 6.7.12` |
| Language/build | TypeScript, Bun build pipeline, Stylus, browser userscript output |
| License | MIT (retain copyright/license notices for copied portions) |
| Primary runtime | A modern browser already running `https://www.xbox.com/*/play*` |

Better xCloud is **not an xCloud client implementation**. It is a userscript that runs at document start inside the official Xbox Cloud Gaming web application. It modifies the host page, fetches, browser APIs, and WebRTC objects that the official web application already owns.

The repository’s own userscript header requires the Xbox web page and modern browser globals. Its declared browser target is Chrome 80+; its supported-platform documentation lists desktop/mobile/TV browser environments, not Xbox 360.

## Observed responsibilities

### Authentication, catalog, and session preparation

* Better xCloud observes `window.xbcUser.isSignedIn`; Microsoft account login itself is performed by the host site at `/auth/msa`.
* It intercepts the host’s `fetch`/XHR calls rather than implementing Microsoft authentication.
* On the observed `/v2/login/user` response it reads a short-lived `gsToken` and region/base-URI settings supplied by the host service.
* It can call the selected region’s `/v2/titles` and `/v1/waittime/<id>` with `Authorization: Bearer <gsToken>` for title/wait-time UI.
* The host page owns normal catalog rendering. Better xCloud only adjusts Game Pass gallery responses (`catalog.gamepass.com/sigls/...`) for features such as custom touch layouts.
* It intercepts the host’s `POST .../sessions/cloud/play` request to alter existing settings such as reported device profile, desired resolution, and locale. It does not independently construct a supported cloud session from scratch.

These routes are source-observed implementation details, not public Microsoft API contracts. They may be changed, gated, or rejected at any time. Do not hard-code them into a native client without permission and an authorized compatibility test.

### Stream transport and media

The source hooks `RTCPeerConnection`, `RTCRtpTransceiver`, `RTCDataChannel`, `HTMLVideoElement`, `MediaStream`, `AudioContext`, WebGL2, and optionally WebGPU. The important distinction is:

* **The browser’s WebRTC engine owns the actual connection, ICE, DTLS, SRTP, RTP, media decode, and data-channel transport.**
* Better xCloud hooks `setLocalDescription` to reorder H.264 SDP payloads and set an SDP `b=AS` bitrate cap. It does not implement an RTP receiver, DTLS, SRTP, H.264 decoder, audio decoder, or WebRTC signaling stack.
* It recognizes H.264 `profile-level-id` prefixes for constrained-baseline (`42e`), a low-profile selection (`420`), and main (`4d`) preference. The actual offer/answer and accepted profile are service/browser negotiated.
* Its default player is the page’s `HTMLVideoElement`. WebGL2/WebGPU players upload already-decoded video frames into a canvas for filtering; they are post-processing renderers, not decoders.
* It uses an `AudioContext`/`MediaStream` gain node for volume. The source does not state a negotiated cloud audio codec. **Audio codec is UNKNOWN — NEEDS AUTHORIZED SDP/RTC-STATS CAPTURE.**

### Input, settings, and statistics

* Browser `Gamepad` APIs, keyboard events, Pointer Lock, touch, fullscreen, vibration, and optional microphone APIs support Better xCloud features.
* A patch accesses the host’s `inputSink.onGamepadInput(...)`; project types also expose `sendGamepadInput(...)`. The exact framing and transport of gamepad input remain hidden in the official host bundle. A `message` RTC data channel is observed for title-information messages, but the supplied source does not prove that it is the input channel.
* Settings use browser `localStorage`; controller/layout/shortcut records use IndexedDB (`BetterXcloud`, version 4). Session telemetry buffers are also manipulated through `sessionStorage`/IndexedDB.
* Stream stats call `RTCPeerConnection.getStats()` for inbound video resolution/FPS/loss/jitter/bitrate/decode time and selected candidate-pair RTT/bytes.

## Component classification

| Component | Evidence | Classification | Xbox 360 action |
| --- | --- | --- | --- |
| Userscript bootstrap / DOM injection | `window`, `document`, CSS, history patches | REQUIRES MODERN BROWSER | Do not port. |
| Microsoft login page flow | Host `/auth/msa`, browser identity/cookies | REQUIRES MODERN BROWSER; REQUIRES MICROSOFT SERVICE | Relay/browser-managed proof only. |
| `gsToken` use | Token read from host login response | REQUIRES MICROSOFT SERVICE | Never mint or fake; use only an authorized service flow. |
| Catalog presentation | Host Xbox page plus Game Pass gallery interception | REQUIRES MODERN BROWSER | Reimplement a licensed relay catalog/API contract. |
| Title and wait-time helper calls | `v2/titles`, `v1/waittime` with bearer token | REIMPLEMENT IN C/C++; REQUIRES MICROSOFT SERVICE | Do not rely on as supported public API. |
| Cloud play request tuning | Intercepts `/sessions/cloud/play` | REQUIRES MODERN BROWSER; REQUIRES MICROSOFT SERVICE | Not a native session implementation. |
| ICE candidate rewrite | Parses `/sessions/.../ice` candidate strings | REQUIRES WEBRTC | Native ICE implementation or relay. |
| SDP codec/bitrate preferences | `RTCPeerConnection.setLocalDescription` | REQUIRES WEBRTC | Reimplement only after transport is proven. |
| WebRTC transport | Browser `RTCPeerConnection` | REQUIRES WEBRTC | Major native stack; direct route currently blocked. |
| H.264 profile preference | SDP text manipulation | REIMPLEMENT IN C/C++ | Only meaningful after negotiation/decoder proof. |
| Video display filters | HTML video / WebGL2 / WebGPU | REQUIRES MODERN BROWSER | Rebuild selected post-processing in D3D9 only after decode. |
| Audio volume | `AudioContext` gain node | REQUIRES MODERN BROWSER | Reimplement PCM output; codec remains unknown. |
| Controller mapping | Browser Gamepad and host input sink | REIMPLEMENT IN C/C++ | Map XAM input to a documented relay protocol. |
| Mouse/keyboard/touch/mic | Browser APIs | REQUIRES MODERN BROWSER | Exclude from first Xbox 360 scope. |
| Settings | localStorage + IndexedDB | REIMPLEMENT IN C/C++ | Bounded local settings file/database. |
| Stream stats | WebRTC `getStats()` | REQUIRES WEBRTC | Relay must expose equivalent metrics. |
| Screenshots | Canvas/video APIs | REQUIRES MODERN BROWSER | Optional native D3D9 capture later. |
| Remote Play features | Host xhome browser paths | REQUIRES MODERN BROWSER; UNKNOWN — NEEDS TESTING | Out of first cloud-client scope. |
| Update check / GitHub pages | browser fetch | REQUIRES MODERN BROWSER | Exclude; use signed app update process if needed. |

## Browser and build dependencies

The repository uses TypeScript 5.9, Bun APIs/macros, Node FS utilities in the build, ESLint, Stylus, DOM APIs, Fetch/XHR, `Request`/`Response`, URL parsing, Web Components, WebGL2, WebGPU, Web Audio, Gamepad, pointer lock, fullscreen, storage APIs, IndexedDB, browser performance/time APIs, `RTCPeerConnection`, SDP, and RTC stats. This is a browser customization project, not a portable C++ library.

## Porting conclusion

Do not compile or mechanically translate Better xCloud into an XEX. The reusable product ideas are: controller-oriented stream settings, region/status UI, conservative stats, accessibility-oriented navigation, and a clean state model. The code that interacts with xCloud depends on the official web application and browser runtime and is therefore either browser-dependent or must be independently reimplemented and validated.

