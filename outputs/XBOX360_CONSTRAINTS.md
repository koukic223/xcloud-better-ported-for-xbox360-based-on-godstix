# Xbox 360 constraints relevant to cloud streaming

| Area | Constraint | Impact |
| --- | --- | --- |
| CPU | Big-endian, three-core PowerPC Xenon | Every third-party media/crypto/network dependency needs a tested PPC/XDK port. Avoid assuming x86/SIMD code works. |
| Memory | 512 MiB physical hardware; title budget is runtime-dependent | Bounded packet/jitter/frame/audio queues are mandatory. Do not infer budget from PE heap reserve. |
| Executable | XEX2 native title, legacy XDK ABI | Modern Windows/Linux libraries and binaries are not drop-in dependencies. |
| Graphics | D3D9 path is proved | Good for UI/presentation; does not establish H.264 decode. |
| Input | XAM controller polling is proved | Excellent native input source; upstream cloud protocol still missing. |
| Audio | XAudio2 output is proved; XMA facilities exist | PCM playback is usable; cloud codec conversion still needed. |
| Network | TCP/DNS/HTTPS exists in GodStix | UDP/IPv6/current TLS/MTU/NAT behavior needs hardware tests. |
| TLS | Existing `XboxTLS` + CA bundle | No proof of modern TLS, DTLS, SRTP, ALPN, SNI, or provider compatibility. |
| Browser | No browser/webview/JS engine in evidence | Browser-only Better xCloud code is non-portable. |
| WebRTC | No XEX support in evidence | Must be ported at high cost or terminated by relay. |
| Video decode | No callable H.264 decoder proved | Primary technical gate. |
| Service support | Current Xbox public cloud guidance names supported devices/browsers, not Xbox 360 | A direct client cannot be presumed accepted even if technically built. |

## Design rules

1. 720p must be measured before exposing 1080p.
2. Decoder and transport queues must have caps and drop late data.
3. Keep UI/audio/decoder/network ownership separate; never let a downloader-style worker block presentation.
4. Store no Microsoft credentials on-console.
5. Keep Xbox-specific XDK code behind interfaces so relay/local tests can run independently.

