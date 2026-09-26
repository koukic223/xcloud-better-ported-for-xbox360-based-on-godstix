# GodStix architecture and reuse map

This companion view summarizes the evidence in [GODSTRIX_ANALYSIS.md](GODSTRIX_ANALYSIS.md). It is intentionally an architecture inference, not recovered source code.

```text
                        GodStixStore.xex
                               |
      +------------------------+------------------------+
      |                        |                        |
 Native D3D9 UI           Store services            Platform services
 D3D9/D3DX9/XGraphC       HTTPS catalog             XAM/Xbox kernel
 PNG cover cache          TCP + XboxTLS             XamInputGetState
 XAudio2 UI sounds        queue/resume/install      threads/events/TLS
      |                        |                        |
 Home/search/favorites    Content/7z files          HDD/USB/FATX paths
 download screens         verification/cache        title launcher
```

## Dependency graph

```text
UI loop
 ├─ D3D9 renderer ── cover/font assets
 ├─ XamInputGetState ── controller navigation
 ├─ XAudio2 ── WAV navigation feedback
 ├─ Store controller
 │   ├─ catalog client ── TCP/DNS ── XboxTLS ── HTTPS API
 │   └─ download manager ── workers/events ── ranges/retry/resume
 └─ installer/verifier ── FATX paths ── Content directory ── XamLoaderLaunchTitle
```

## Reuse boundary

Only platform concepts are candidates for a clean-room replacement. No supplied source authorizes direct code reuse.

| Existing area | New cloud-client equivalent | Decision |
| --- | --- | --- |
| D3D9 renderer/UI loop | `D3D9UI` | Rebuild. |
| XamInputGetState | `InputManager` | Rebuild behind a small controller abstraction. |
| XAudio2 UI sound | `UiAudio` | Rebuild; keep stream audio separate. |
| Store HTTPS client | `RelayTransport` only | Do not inherit for Microsoft-facing traffic. |
| Download workers | `SessionManager`/`ReconnectManager` | Rebuild; semantics are different. |
| File caches/settings | `SettingsStore`/art cache | Rebuild with bounded storage and no tokens in plaintext. |
| Game installer/launcher | none in the cloud path | Exclude from the cloud client. |

## Security boundary

The XEX includes an HTTPS store implementation, but its TLS version/cipher/SNI behavior and audit status are unknown. Treat it as untrusted legacy implementation detail. The new design must not reuse its query-string credential pattern, service credentials, certificates, or endpoints.

