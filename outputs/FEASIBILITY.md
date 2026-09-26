# Cloud-gaming feasibility verdict

## A. Definitely feasible

* Native Xbox 360 D3D9 cloud-style UI: home, search, favorites, details, quality settings, connection/error states.
* Controller-first navigation/input collection using `XamInputGetState`.
* Local settings, art cache, worker-based I/O, real network diagnostics, and a real statistics overlay.
* Xbox-to-user-owned-relay control transport, subject to a secure, tested transport implementation.

## B. Feasible with substantial engineering

* D3D9 presentation of decoded frames and XAudio2 PCM output.
* H.264/audio software decode **only if physical-hardware benchmarks pass**.
* An authenticated, low-latency relay protocol with loss/reorder/reconnect handling.
* A relay catalog/UI bridge built from a permitted source.

## C. Requires an intermediary server

* Browser-only sign-in/session environment.
* Modern WebRTC, ICE/STUN/TURN, DTLS/SRTP/RTP, and any SCTP/data-channel path.
* Translation of real cloud session state to a narrow Xbox protocol.
* Current service compatibility monitoring and safe secret handling.

## D. Browser-dependent

* Better xCloud’s userscript, DOM/CSS injection, browser storage, browser Gamepad API, Web Audio, WebGL2/WebGPU, `getStats`, pointer lock, and official web-app internals.

## E. Currently blocked

1. Direct supported Microsoft auth/session contract for a native Xbox 360 client.
2. Native WebRTC portability/security validation.
3. H.264 and actual cloud-audio decode on retail Xbox 360 at target latency.
4. Exact cloud input framing, audio codec, keepalive, reconnection, and negotiated SDP from authorized runtime evidence.
5. Original GodStix source/build environment: binary-only evidence cannot safely be “reused” as requested.

## Recommended implementation strategy

Choose **relay architecture research**, starting with legal local-stream Prototypes A–D. Do not modify the XEX until original source or a clean-room project exists, 720p local decode is measured, and the relay has a real session/transport test. If provider authorization or media performance cannot be demonstrated, retain the native UI and use a documented local/third-party backend instead.

