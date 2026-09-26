# GodStix Store binary analysis

## Scope and evidence

This report analyzes the supplied binary, whose on-disk name and internal PE-module name are **`GodStixStore`** (not `GodStrix`).  “GodStrix” is therefore used only as the proposed product name, not as a claim about the supplied executable.

| Item | Value |
| --- | --- |
| Examined file | `GodStixStore.xex` |
| SHA-256 | `29017A6B5A8D14D088E69BD2D2FC69A4E3971EF3A9AC45A58C01D318E1C8D67F` |
| File size | 5,054,464 bytes |
| Source supplied | No |
| Binary modified | No |
| Payload recovery | Decrypted and inspected in memory only; no PE or asset was written out |
| Reader-note SHA-256 | `65ABA6161BEE3CCAC878710240ECA841076E1CCFE318D72FD7AFDF61D4F7C933` |

The reader note is useful product documentation, but binary evidence takes precedence when they differ. Import names were resolved using the public Xbox 360 ordinal map in the `emoose/xbox-reversing` project, commit `5f85b9ec8c771577532ca1cfa20c691e9033f2c2`. That map identifies imports; it does not reveal high-level source structure.

## Executable and XEX structure

The file is a normal Xbox 360 **XEX2 title process** (`XEX2`, module flag `0x00000001`), not an Xbox One/Series executable, Windows executable, or a browser bundle.

| Field | Observed value | Meaning |
| --- | --- | --- |
| Header size | `0x00002000` | First base-image byte begins at file offset `0x2000`. |
| Security-info offset | `0x000000F8` | Standard XEX2 security header location. |
| Optional headers | 12 | Includes sections, file descriptor, entry point, base, imports, static libraries, TLS, stack, execution ID, and LAN key. |
| Entry point | `0x821C8EF0` | PowerPC code address. |
| Image base | `0x82000000` | Xbox 360 title address range. |
| Base image descriptor | encrypted, raw blocks | AES-CBC encrypted XEX image; not compressed. |
| Decryption validation | devkit all-zero boot key yielded `MZ`/`PE` | The payload could be inspected in memory. This is not an authorization to redistribute or modify it. |
| PE machine | `0x01F2` | PowerPC / Xenon big-endian target. |
| Linker | 10.0 | Microsoft/XDK-era PE toolchain. |
| PE sections | 9 | `.rdata`, `.pdata`, `.text`, `.data`, `.XBMOVIE`, `.tls`, `.idata`, `.XBLD`, `.reloc`. |
| PE image size | `0x58FC00` | PE-declared mapped image size. |
| TLS slots | 64 | Native TLS is present. |
| Stack reserve | `0x40000` | 256 KiB main reserve declared in XEX metadata. |
| Heap reserve | `0x100000` | 1 MiB PE reserve; this is not a total application-memory budget. |

The module is PowerPC/Xenon compatible by construction. Its header contains no evidence of an export table intended for third-party reuse. Treat it as an application, not a linkable SDK component.

## Linked libraries and recovered imports

The XEX names two import modules and records 362 import references (175 function thunks and 187 import variables, with paired references for many APIs):

* `xam.xex` — 54 references.
* `xboxkrnl.exe` — 308 references.

Its XEX static-library metadata also records `XAPILIB`, `XBOXKRNL`, `XNET`, `XAUDIO2`, `XAPOBA`, `XMCORE`, `LIBCPMT`, `D3D9`, `D3DX9`, and `XGRAPHC`. XDK library build metadata is mostly `2.0.21256.16384`; C/C++ runtime metadata is `11886`. This proves an XDK-native C/C++ program with D3D9, networking, and XAudio2 dependencies. It does **not** prove that every capability in an SDK library is used.

Recovered call-level evidence is stronger:

| Area | Proved calls / data | Conclusion |
| --- | --- | --- |
| TCP and DNS | `NetDll_WSAStartup`, `socket`, `connect`, `send`, `recv`, `select`, `ioctlsocket`, `setsockopt`, `XNetDnsLookup` | Native IPv4-style socket client with non-blocking connection handling. |
| Ethernet status | `NetDll_XNetGetEthernetLinkStatus` | The app can inspect wired-link state. |
| Controller | `XamInputGetState` | Native controller polling is implemented. |
| Threads and synchronization | `ExCreateThread`, `ExTerminateThread`, events, semaphores, critical sections, TLS APIs | Worker-thread architecture, appropriate for covers/downloads. |
| Filesystem | `NtCreateFile`, `NtOpenFile`, `NtReadFile`, `NtWriteFile`, directory/volume queries, flush | Real persistent storage and installation work. |
| Memory | virtual and physical allocation/free APIs | Native memory management; exact allocation policy needs disassembly/source. |
| Audio | XAudio2/XMA-related kernel APIs plus navigation-WAV strings | XAudio2 playback is real. The existing app is not a cloud-audio decoder. |
| Launcher | `XamLoaderLaunchTitle`, `XamLoaderTerminateTitle` | It can launch/terminate titles. |
| UI dialogs | XAM keyboard and message-box APIs | Native text entry/errors are available. |

No imports or decrypted strings establish a webview, JavaScript engine, HTML renderer, WebRTC stack, ICE, STUN, TURN, DTLS, SRTP, SCTP, H.264/AVC decoder, or cloud-stream protocol.

## Network, HTTP, and TLS

This is an actual HTTPS store client, not a mock download screen. Decrypted strings prove the following implementation traits:

* Raw TCP sockets, DNS lookup, HTTP/1.1 request construction (`GET %s HTTP/1.1`, `Host`, keep-alive/close), redirects, chunked transfer decoding, ranges (`Content-Range`), cancellation, and resume/multi-part downloads.
* A custom `XboxTLS_*` layer with `XboxTLS_Connect`, `XboxTLS_Read`, `XboxTLS_Write`, and `XboxTLS_Free` paths.
* X.509 initialization and embedded trust anchors, including modern CA labels such as Microsoft, DigiCert, ISRG/Let’s Encrypt, Go Daddy, ZeroSSL, and Starfield.
* An HTTPS-only store policy: strings explicitly reject a non-HTTPS `api_base_url`.
* Catalog/download endpoints for games, homebrew, categories, files, and server-selected URLs.

This **does not** prove TLS 1.2/1.3 support, cipher-suite coverage, SNI behavior, ALPN, certificate-chain freshness, HTTP/2, OAuth compatibility, or whether Microsoft’s current endpoints will accept it. The TLS source and a live, authorized interoperability test are required before reusing any part of this stack for a cloud relay or an Xbox service.

A security issue should not be carried forward: the recovered store endpoint template contains a login/password query-string format. The new project must not reuse that protocol or expose credentials in URLs, logs, or settings.

## Graphics, UI, assets, audio, and input

### Graphics and UI

`D3D9`, `D3DX9`, and `XGRAPHC` are linked. The recovered image includes many embedded PNG assets and an XEX resource entry named `arial`/`font`; it also contains custom cover-cache and UI strings. There are no XUI imports. The most defensible conclusion is a custom native D3D9 UI, rather than a browser or XUI application.

The `.XBMOVIE` PE section is only 12 bytes of metadata in this binary; it is not evidence of a playable video pipeline. Image assets and D3D rendering must not be mistaken for video decode support.

### Audio

Strings identify `XAudio2Create`, mastering/source voices, navigation WAV parsing, and failure handling. This supports menu/UI sound. No stream audio codec (Opus, AAC, Vorbis, etc.) is evidenced.

### Input

`XamInputGetState` and the product documentation both corroborate controller-first navigation. A new native UI can reuse the *concept* of controller navigation, but source-level reuse is unavailable.

## Filesystem, catalog, and download/install behavior

The reader note claims catalog browsing, search, favorites, download queue/history, sorting, three languages, covers, auto-install, verification, HDD/USB target choice, and Ethernet advice. Binary strings corroborate the central claims:

* Catalog endpoints include games, game details/files, categories, homebrew, auth/guest checks, and download URL lookup.
* Persistent files include `game:\godstix.ini`, `favorites.txt`, `downloads.db`, `download_queue.db`, `install_history.json`, and `verify_cache.txt`.
* It enumerates FATX-style drives, writes under `Content\0000000000000000`, uses a `godstix_tmp` directory, and reports install verification outcomes.
* It handles concurrent download workers, per-part resume, HTTP 206 validation, hash checks, retry/cancel, disk cover cache, and 7z/homebrew extraction paths.

There is no binary evidence of a self-update mechanism that downloads and replaces this XEX. “Always updated” in the reader note should be interpreted as catalog/content freshness until source or runtime traffic proves executable updating.

## Reuse assessment

| Component | Assessment | Reason |
| --- | --- | --- |
| Native D3D9 shell ideas | Reimplement | Binary-only; no safe source-level reuse. |
| Controller navigation model | Reimplement | Native API is proven, but code is unavailable. |
| Local settings/favorites model | Reimplement | File concepts are useful; data format and security need redesign. |
| TCP/DNS design | Reimplement or isolate behind a new transport interface | Existing low-level capability is real, but code quality and IPv6/TLS behavior are not verified. |
| Existing TLS code | Do not reuse yet | Source, protocol level, audits, and live compatibility are missing. |
| Store/catalog/download/install subsystem | Do not reuse for cloud gaming | It is materially different and carries content/service/security concerns. |
| Audio/UI assets | Do not copy by default | Ownership and licensing are unknown. |
| Web/browser/streaming | Not reusable | No evidence that it exists. |

## What can be recovered or reimplemented

The recovered PE can support authorized static reverse engineering: control-flow analysis, API call identification, resource inventory, config format recovery, and protocol observation against services the operator owns or is authorized to test. It cannot recreate original C++ names, types, comments, build scripts, source license, or asset rights.

For `GodStrix-Better-Cloud.xex`, use a clean-room native implementation. Preserve only documented external behavior that is lawful and desired; do not patch the existing store binary into a streaming client.

## Bottom line

GodStix is a capable Xbox 360 **store/downloader with native D3D9, XAudio2, controller, filesystem, threading, TCP, and custom HTTPS capabilities**. It is not a cloud-gaming client. The existing UI/input/platform experience is a useful reference, but the authentication, transport, decode, streaming, and session-management layers required for cloud gaming would be new engineering.

