# Better xCloud → GodStix → Xbox 360 porting matrix

Labels: **YES** means demonstrated on the supplied XEX; **NO** means absent in reviewed evidence; **UNKNOWN** means no claim without a target test.

| Feature | Better xCloud technology | GodStix equivalent/evidence | Xbox 360 support | Required work / decision |
| --- | --- | --- | --- | --- |
| Native executable | Browser userscript | XEX2/PPC/Xenon | YES | Preserve target platform; source is required for actual reuse. |
| UI shell | HTML/CSS/DOM | D3D9/D3DX9/XGraphC custom UI | YES | Rebuild cloud screens in D3D9. |
| Game catalog display | Official web app + browser fetch | Existing store catalog UI/API | YES for display only | Feed from permitted relay catalog; never fake it. |
| Game search/favorites | DOM/local storage | Store search/favorites files | YES | Rebuild local state with bounded storage. |
| Microsoft authentication | Host browser/cookies | Store-specific login only | NO | Relay/browser-managed approved flow; do not reuse store auth. |
| Cloud token | Host `gsToken` | None | NO | Keep token at relay only. |
| Cloud session creation | Host web app, play request | None | NO | Requires relay/provider-approved integration. |
| HTTPS/TCP/DNS | Browser fetch | sockets/DNS/custom `XboxTLS` | YES, compatibility UNKNOWN | Reuse only after source/security/TLS tests; otherwise replace behind interface. |
| WebRTC | Browser `RTCPeerConnection` | None | NO | Relay recommended; direct port is major R&D. |
| ICE/STUN/TURN | Browser WebRTC | None | NO | Relay, or full native stack after proof. |
| DTLS/SRTP/RTP | Browser WebRTC | None | NO | Relay, or maintained native dependencies after proof. |
| H.264 SDP preference | Browser SDP rewrite | None | YES for parsing only, decode UNKNOWN | Relay exposes a tested profile; prove target decoder. |
| Video decode | Browser media pipeline | D3D9 output only | UNKNOWN | Local 480p/720p decoder prototype. |
| Video display processing | HTML video/WebGL2/WebGPU | D3D9 | YES | Optional D3D9 post-process after decoder. |
| Audio decode/output | Browser MediaStream/AudioContext | XAudio2 UI audio | Output YES; cloud codec decode UNKNOWN | Decode confirmed codec to PCM then XAudio2. |
| Controller transport | Browser Gamepad + opaque host input sink | `XamInputGetState` | Polling YES; upstream transport NO | Define authenticated relay input protocol. |
| Stream stats | `getStats()` | None | YES to implement | Use real packet/decoder/timestamp counters. |
| Keepalive/reconnect | Host/browser behavior | Download retries, workers | Partial concepts only | New session/reconnect state machine. |
| Workers/synchronization | Browser event loop/workers | XEX threads/events/semaphores | YES | Bounded media/transport queues, not downloader queues. |
| Cache/config | localStorage/IndexedDB | INI/files/cache | YES | No cloud credentials; cache only art/preferences. |
| Update flow | Browser/userscript release | No XEX self-update proved | UNKNOWN | Out of first scope; signed update design later. |

## Result

GodStix contributes the **native shell**. Better xCloud contributes **web-client behavior and product ideas**, not portable transport/media code. The missing elements are exactly the cloud-specific ones, so use a relay or prove them one by one with local tests.

