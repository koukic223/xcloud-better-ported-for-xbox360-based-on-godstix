# Xbox 360 native media-pipeline feasibility

## Evidence-based baseline

The supplied GodStix XEX links D3D9/D3DX9/XGraphC and XAudio2, and it includes PNG and UI-WAV handling. This proves rendering and menu audio. It does **not** prove H.264, MPEG-4, WebRTC, Opus, AAC, or streaming-video decode.

`D3D9` presents GPU surfaces; it is not a video decoder. `XAudio2` plays PCM/XMA-oriented audio buffers; it is not a general WebRTC audio decoder.

## Decoder inventory

| Capability | Evidence | Practical status |
| --- | --- | --- |
| D3D9 scan-out/compositing | Proved in GodStix linked libraries | Suitable for UI and presentation of an already decoded frame. |
| XAudio2 PCM output | Proved in GodStix linked libraries and WAV strings | Suitable for queued PCM from a separately implemented audio decoder. |
| XMA hardware decode | Xbox 360/XAudio evidence exists; GodStix links XAudio2 | Hardware is for XMA-family content, not proof of Opus/AAC support. It does not solve cloud audio unless the relay intentionally transcodes to an XMA-compatible, licensed pipeline. |
| H.264 hardware decode callable by a title | Not found in GodStix imports or supplied XDK material | **UNKNOWN — BLOCKED until a licensed, documented API is demonstrated on target hardware.** Do not assume a GPU/video playback block is exposed to homebrew XEX code. |
| H.264 software decode | Technically portable in principle | Needs a PowerPC/Xenon benchmark, careful licensing, NEON-free/PPC work, and a low-latency implementation. Not assumed viable at 720p60. |
| OpenH264 | PowerPC support exists upstream | Candidate only; no Xbox 360/XDK build or performance result has been demonstrated. Licensing and toolchain work remain. |
| FFmpeg/libavcodec | General software solution | Large integration and licensing/performance surface. No Xbox 360 target result is in scope. |
| Opus software decode | Technically small/portable | Candidate after the cloud audio codec is confirmed; cannot be selected as a fact yet. |
| AAC software decode | Technically possible | Candidate only; no evidence xCloud negotiates it here. |

## Native frame path required

```text
Authenticated relay packets
  -> authenticated decrypt/reorder
  -> video payload reassembly (codec-aware)
  -> H.264 decoder [unproven selection]
  -> YUV/decoded frame
  -> color conversion/upload
  -> D3D9 texture/surface
  -> D3D9 composition + present
```

The output side is realistic and aligns with the existing application. The decoder is the gating component. Avoid copying decoded pixels through multiple CPU buffers: use a bounded frame pool, an explicit render ownership handoff, and drop late frames rather than increasing latency.

For audio:

```text
Relay audio packet
  -> decrypt/reorder/depacketize
  -> confirmed codec decoder
  -> 48 kHz PCM (format selected after testing)
  -> bounded audio jitter queue
  -> XAudio2 source voice
  -> TV
```

XMA acceleration cannot be presumed usable for arbitrary web-stream audio. PCM output via XAudio2 is feasible; codec selection and CPU cost are open.

## Performance envelope and test criteria

The console has a three-core PowerPC Xenon CPU and 512 MiB physical RAM, but usable title memory depends on the runtime and allocations. No safe memory budget can be derived from the 1 MiB PE heap-reserve field. The XEX itself maps roughly 5.6 MiB before dynamic assets/buffers.

Targets are hypotheses to measure, not promised performance:

| Mode | Initial target | Why | Pass criterion |
| --- | --- | --- | --- |
| 480p30 | Decoder bring-up | Establish parser/surfaces/timing cheaply | 10-minute playback, no unbounded queue growth. |
| 720p30 | First useful relay demo | Lower CPU and bandwidth risk | Stable visual output with measured frame loss/latency. |
| 720p60 | Desired interactive target | Cloud gaming normally benefits from 60 fps | Sustained on retail hardware with decoder time below frame budget. |
| 1080p30/60 | Feasibility experiment | Not a first release target | Only retain if measured decoder, memory, and network headroom exist. |

At 60 fps the total per-frame budget is 16.67 ms, shared by packet work, decoding, upload, UI, and presentation. A software H.264 decoder that regularly exceeds this budget will create queueing and controller-to-photon latency. Therefore, test 720p first and reject 1080p rather than reporting a fake quality option.

## Required benchmark harness

Before any Microsoft-facing work, run a local server test with known H.264 samples. Measure on physical hardware:

* decode time per frame (min/median/p95/p99), CPU-core use, and peak resident memory;
* compressed bitrate, packet loss/reorder recovery, and frame drops;
* NAL-to-present timestamp delay and audio queue depth;
* D3D9 upload/present cost at 480p, 720p, then 1080p;
* 10-minute soak and reconnect behavior.

Expose values only when measured. A valid overlay reports `N/A` until a metric has an instrumentation source.

## Decision

The native D3D9 and XAudio2 ends of a media pipeline are viable. Hardware H.264 decode is **not established** for this XEX/XDK context, and direct cloud codecs are not yet known. The next practical gate is a legal local 480p/720p H.264 + PCM test, followed by a measured decoder decision. If no target-hardware decoder sustains 720p with acceptable latency, the relay must transcode to a codec/path that the Xbox 360 can demonstrably decode, or the cloud-client plan stops at the UI/control layer.

