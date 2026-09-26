# xCloud session flow and evidence boundary

## Confirmed browser-visible flow

```text
1. User opens xbox.com/play in a compatible browser
2. Host Microsoft sign-in flow establishes browser session
3. Official web app reports signed-in state
4. Host login request returns region configuration and gsToken
5. Official UI obtains/renders catalog and user selects a product
6. Host issues cloud-play session request
7. Host receives configuration and ICE exchange information
8. Browser RTCPeerConnection negotiates and receives media
9. Browser video/audio APIs render decoded stream
10. Host input subsystem sends controller state
```

Better xCloud modifies steps 4, 6, 7, 8, 9, and 10 at browser API boundaries. It is not the owner of the end-to-end session.

## Evidence trace

| Step | Better xCloud evidence | Unavailable / not asserted |
| --- | --- | --- |
| Sign-in | Watches `window.xbcUser.isSignedIn`; redirects host auth navigation as needed | OAuth request/response schema, cookie policy, MFA, device enforcement. |
| Region/token | `/v2/login/user` response reads `gsToken` and `offeringSettings.regions` | Token issuer/claims/expiry and public supportability. |
| Catalog | Intercepts `catalog.gamepass.com/sigls/...`; title helper calls selected base URI | Complete catalog contract, entitlement behavior. |
| Launch | Intercepts `POST .../sessions/cloud/play`; changes existing `settings` and device info | Required request fields, service acceptance, stable endpoint contract. |
| Configuration | Handles endpoint ending in `/configuration` and `clientStreamingConfigOverrides` | Full configuration semantics and session identifiers. |
| ICE | Rewrites response from a session `/ice` GET containing candidate strings | STUN/TURN URLs, candidate selection from live runtime. |
| Negotiation | Hooks `setLocalDescription`; edits SDP H.264 order/bitrate | Remote SDP, DTLS fingerprints, BUNDLE, exact codecs/levels. |
| Media | Browser HTML video/MediaStream/AudioContext, RTC stats | Encoded media packet format/audio codec, decoder implementation. |
| Input | Calls host input API boundary; sees a `message` data channel | Input framing/reliability/data-channel label. |
| Recovery | UI state cleanup observed | Cloud reconnect/resume protocol. |

## Native translation

```text
XamInputGetState -> CloudInput -> relay input protocol
D3D9 UI          -> relay catalog/session status
CloudTransport    -> authenticated relay control/media packets
VideoDecoder      -> D3D9 surface
AudioDecoder      -> XAudio2 PCM queue
```

Direct substitution of the official browser steps with raw calls from an XEX is not currently validated or recommended. See `XCLOUD_STREAMING_PROTOCOL.md` for the protocol evidence ledger.

