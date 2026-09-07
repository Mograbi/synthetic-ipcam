# synthetic-ipcam

Camera-free test cameras: an RTSP server ([mediamtx](https://github.com/bluenviron/mediamtx))
plus ffmpeg publishers shaped like real consumer IP cameras — a **dual-stream
H.264 camera**, a **single-stream H.265 camera**, and a **forward-looking AV1
camera**, all with AAC-LC 16 kHz mono audio, served over both **RTSP and
RTSPS (TLS)**. Point any NVR, recorder, or RTSP client at them and test
ingest, recording, motion detection, events, playback, and live view without
touching hardware.

## Quick start

```sh
docker compose up -d
```

| Stream | URL | Codec | Resolution |
|---|---|---|---|
| Camera 1, main | `rtsp://<host>:8554/cam1-main` | H.264 High + AAC | 1280×720 @ 15 |
| Camera 1, sub | `rtsp://<host>:8554/cam1-sub` | H.264 High + AAC | 640×360 @ 15 |
| Camera 2 (single stream) | `rtsp://<host>:8554/cam2-main` | H.265 Main + AAC | 1280×720 @ 15 |
| Camera 3 (single stream) | `rtsp://<host>:8554/cam3-main` | AV1 Main + AAC | 1280×720 @ 15 |

Every stream is also served over TLS on port 8322 — same paths, `rtsps://`
scheme: e.g. `rtsps://<host>:8322/cam1-main`.

Override the published ports with `RTSP_PORT=8555 RTSPS_PORT=8323 docker
compose up -d`.

## RTSPS (TLS)

Encryption is *optional*, not strict: the same server answers plain RTSP on
8554 and RTSPS on 8322, so a consumer can be tested against both transports
from one rig. The certificate is self-signed (`CN=synthetic-ipcam`), generated
on first start by the one-shot `certs` service into a named volume — delete
the `certs` volume to rotate it. Clients must skip verification or trust the
cert, which mirrors real cameras: RTSPS-capable hardware practically always
ships a self-signed cert.

```sh
ffprobe -rtsp_transport tcp -tls_verify 0 -i rtsps://localhost:8322/cam1-sub
```

**Know what you are testing:** over RTSPS mediamtx advertises
`m=video 0 RTP/SAVP` — the media itself is **SRTP-encrypted**, keyed out of
band via MIKEY (RFC 4567) — where plain RTSP advertises `RTP/AVP`. Most real
RTSPS cameras do the opposite: plain RTP carried inside the TLS control
connection. So this rig is a *stricter* RTSPS test than typical hardware, and
a consumer that handles real cameras may still fail here. ffmpeg copes;
GStreamer's `rtspsrc` (1.22) negotiates SRTP, builds a decryptor, and then
receives nothing — no media, no error, just silence. If your stack goes quiet
against cam1 over 8322 but works over 8554, check the SDP profile before
suspecting your TLS setup.

(`-tls_verify 0` because modern ffmpeg verifies TLS certificates by default
and this one is self-signed — the same accommodation your NVR needs to make
for real RTSPS cameras.)

## AV1 (camera 3)

AV1 IP cameras barely exist yet — this stream is for finding the next codec
cliff before your users do, the way camera 2's CRA keyframes find HEVC bugs.
Encoded with SVT-AV1 (preset 10, 2 s keyframe interval). One sharp edge worth
knowing: the AV1 RTP payload format is still an AOM draft, and **ffmpeg gates
it behind `-strict experimental` on both the publish and the consume side** —
so to inspect it:

```sh
ffprobe -rtsp_transport tcp -strict experimental -i rtsp://localhost:8554/cam3-main
```

A consumer that refuses the stream outright (unsupported encoding) is giving
a *correct* answer — the test is whether it says so honestly instead of
failing silently.

## Why the picture looks the way it does

Every element of the frame encodes a lesson learned debugging real pipelines
against this rig:

- **Static SMPTE bars as the background.** A flat dark frame is
  indistinguishable from a frozen or broken decoder at a glance; color bars
  read as "alive picture, nothing moving" instantly.
- **A small always-moving inset, bottom-right.** Continuous proof of life: if
  that box animates, the whole chain — ingest, payloading, transport, decode —
  is working. It covers ~0.4% of the luma, far below typical motion triggers,
  so it never pollutes motion detection.
- **A large moving pattern for 20 s of every 60.** Motion runs must *close*
  for event-driven systems to emit anything: a perpetually-moving test source
  never ends a motion run, so recorders that post events on the falling edge
  post none — which looks like a bug and isn't. The burst inset is sized
  (~47% of the frame) to clear a 3%-of-luma motion trigger with margin at
  both resolutions.

## Why the encoding looks the way it does

- **No B-frames** (`-bf 0` / `bframes=0`): DTS == PTS, matching real IP
  cameras and keeping strict recorders (that assume it) happy.
- **2 s closed GOP** (`keyint=30` @ 15 fps): keyframe-driven chunk rotation
  and live-join both land promptly.
- **`repeat-headers=1` on x265**: VPS/SPS/PPS ride every keyframe, as real
  cameras deliver them.
- **H.265 keyframes are CRA NALs** (that's just what x265 emits in this
  configuration) — deliberately kept, because it's realistic and it bites
  lazy HEVC RTP payloaders: one in-the-wild WebRTC stack wrote valid FU
  headers only for IDR NAL types and shipped malformed fragments for CRA,
  which this rig exposed within minutes (connected PeerConnection, black
  video). If your stack survives cam2, it handles real HEVC cameras.
- **AAC-LC 16 kHz mono** on every stream — the audio shape consumer cameras
  actually send (and the case where a camera *advertises* audio it never
  delivers is worth testing separately by pointing at a stream and dropping
  the audio track).

## Extracted from

Built as the test rig for ain (an NVR for ARM64), where an H.264/H.265 copy
lives in-tree. This repo is the standalone, NVR-agnostic asset.

## License

MIT — see [LICENSE](LICENSE).
