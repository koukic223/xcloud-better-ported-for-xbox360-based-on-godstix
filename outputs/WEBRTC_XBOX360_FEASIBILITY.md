# Native WebRTC on Xbox 360: feasibility

## Verdict

**Direct native WebRTC is possible only as a substantial, unproven port; it is not a practical first implementation.** It should not begin with Chromium or full libwebrtc. The smallest credible direct route would require an independently ported and audited C/C++ stack for ICE/STUN/TURN, DTLS, SRTP/SRTCP, RTP/RTCP, SDP/JSEP, plus the service’s unknown session signaling/input contract and media decoders.

No reviewed project offers a supported Xbox 360/PowerPC/XDK build.

## Candidate assessment

| Candidate | What it provides | Xbox 360 verdict |
| --- | --- | --- |
| Chromium/libwebrtc | Full browser-grade media, network, device, codecs, congestion control | NOT PRACTICAL. GN/Ninja-centric, large Chromium-era dependency graph, modern compiler/runtime assumptions, and no demonstrated Xenon target. |
| OpenWebRTC | GStreamer/GLib ecosystem integration | NOT PRACTICAL. Runtime/dependency footprint is mismatched to XDK/XEX. |
| libdatachannel | C++ WebRTC data channels/media transport; ICE, DTLS, SRTP, SCTP, RTP helpers | SMALLER BUT UNPROVEN. It still needs crypto, libSRTP, libjuice/libnice, usrsctp, CMake/C++ compatibility, socket adapters, and an Xbox 360 port. It does not solve Microsoft session/auth signaling or media decode. |
| Custom minimal WebRTC | Only required RFC components | HIGH-RISK. Smallest runtime footprint but security-critical protocol implementation. Do not write custom DTLS/SRTP cryptography. |
| Relay terminates WebRTC | Modern server owns browser/WebRTC stack | RECOMMENDED RESEARCH PATH. Xbox 360 uses a purpose-built, narrow protocol. |

The current libdatachannel project documents that WebRTC media transport entails ICE/STUN/TURN, DTLS/UDP, SRTP/SRTCP, RTP, and optionally SCTP data channels, with crypto and multiple dependencies. Its documented supported native environments do not include Xbox 360. That makes it a useful architectural reference, not a drop-in XDK solution.

## Direct-client requirements

| Requirement | GodStix evidence | Direct implementation need |
| --- | --- | --- |
| UDP sockets | TCP socket calls are proved; UDP not specifically recovered | Validate XNet UDP behavior, NAT traversal, MTU and select/non-blocking behavior. |
| DNS / HTTPS | Existing DNS/custom TLS exists | Verify IPv6, SNI, current trust/ciphers, and session endpoint behavior. |
| ICE | Absent | Candidate gathering, connectivity checks, trickle behavior, STUN/TURN auth, nomination, timeout/restart. |
| DTLS | Absent | Maintained DTLS library compatible with XDK compiler and PowerPC endian behavior; certificate generation/fingerprint validation. |
| SRTP/SRTCP | Absent | Maintained libSRTP-equivalent, DTLS exporter/key derivation, replay/index handling, RTCP feedback. |
| SCTP data channels | Absent | Port usrsctp or prove cloud input avoids it. Do not assume the observed `message` channel is controller input. |
| SDP/JSEP | Absent | Parse/create/validate offer/answer, BUNDLE/mid, codecs, extmaps, candidates. |
| RTP/RTCP media | Absent | H.264 depacketization, sequence/reorder/jitter, NACK/PLI/FIR/RTX only if negotiated. |
| Congestion control / pacing | Absent | At least a safe bounded sender/receiver policy; service requirements unknown. |
| Auth/session signaling | Absent | A service-approved method; browser web endpoints are not a public native contract. |
| H.264/audio decoding | Absent | Measured media pipeline from `XBOX360_MEDIA_PIPELINE.md`. |

## Toolchain and platform risks

* The supplied title was built with a legacy XDK C/C++ environment on big-endian PowerPC. Modern WebRTC libraries expect newer C++ libraries, build tools, atomics, thread facilities, and POSIX/Win32 socket abstractions.
* Every third-party component must be compiled with XDK-compatible ABI, exceptions/RTTI choices, endian correctness, and static memory constraints. Modern prebuilt Windows binaries are unusable.
* DTLS/SRTP is security-sensitive. A custom legacy TLS layer is not evidence it supports DTLS or the current WebRTC cryptographic profiles.
* Certificate creation, fingerprint verification, time validity, entropy, and secure storage must work on hardware. No fallback that disables verification is acceptable.
* WebRTC input may require SCTP over DTLS. If live capture proves a different channel, the scope can shrink; until then it cannot be omitted.

## Smallest viable implementation, if direct is revisited

Do this only after the cloud service presents an authorized non-browser session contract and media decode is proven:

```text
XNet UDP adapter
  + maintained DTLS backend
  + maintained libSRTP backend
  + ICE/STUN/TURN implementation
  + SDP/BUNDLE parser for captured, supported profile only
  + RTP/RTCP H.264 receiver and feedback subset
  + proven input transport
  + H.264 + audio decoders
```

Deliberately exclude browser DOM, Chromium, WebGPU, WebGL, touch, mouse/keyboard, arbitrary codec fallback, and speculative endpoint logic. Even this “small” stack remains a multi-stage port and security audit, not a quick library integration.

## Recommendation

Use a relay to terminate browser/WebRTC in the first real prototype. Maintain a direct-client spike only as a research branch after these gates are met: authorized session contract, sanitized SDP capture, codec confirmation, XNet UDP test, target media benchmark, and a successful DTLS/SRTP interoperability test against a server the operator controls.

