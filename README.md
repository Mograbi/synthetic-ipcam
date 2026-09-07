# synthetic-ipcam

Camera-free test cameras: an RTSP server ([mediamtx](https://github.com/bluenviron/mediamtx))
plus ffmpeg publishers shaped like real consumer IP cameras — a **dual-stream
H.264 camera** and two **single-stream cameras** (one H.265), all with AAC-LC
16 kHz mono audio, served over both **RTSP and RTSPS (TLS)**. Point any NVR, recorder, or RTSP client at them and test
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
| Camera 3 (single stream) | `rtsp://<host>:8554/cam3-main` | H.264 High + AAC | 1280×720 @ 15 |

Every stream is also served over TLS, on two ports that differ in a way worth
understanding:

| Port | URL | What it is |
|---|---|---|
| 8322 | `rtsps://<host>:8322/cam1-main` | **Camera-shaped**: TLS control connection, plain `RTP/AVP` interleaved inside it |
| 8323 | `rtsps://<host>:8323/cam1-main` | mediamtx native: `RTP/SAVP` — the media itself is SRTP-encrypted |

Override the published ports with `RTSP_PORT=8555 RTSPS_PORT=8322
RTSPS_SRTP_PORT=8323 docker compose up -d`.

## RTSPS (TLS) — two flavours, on purpose

The certificate is self-signed (`CN=synthetic-ipcam`), generated on first start
by the one-shot `certs` service into a named volume — delete the `certs` volume
to rotate it. Clients must skip verification or trust it, which mirrors real
cameras: RTSPS-capable hardware practically always ships a self-signed cert.

```sh
ffprobe -rtsp_transport tcp -tls_verify 0 -i rtsps://localhost:8322/cam1-sub
```

**Port 8322 is what an IP camera means by `rtsps://`**: the *control connection*
is TLS, and RTP rides interleaved inside it as ordinary `RTP/AVP`. It is served
by TLS-terminating in front of the plain RTSP listener, which is exactly the
shape cameras produce — the SDP says `m=video 0 RTP/AVP`, and any client that
speaks RTSP-over-TLS just works.

**Port 8323 is mediamtx's own RTSPS**, which advertises `m=video 0 RTP/SAVP`:
the media is **SRTP-encrypted**, keyed out of band via MIKEY (RFC 4567). Few
cameras do this, and support is uneven — ffmpeg copes; GStreamer's `rtspsrc`
(1.22) negotiates SRTP, builds a decryptor, and then receives nothing at all:
no media, no error, just silence, which surfaces as a negotiation timeout
blaming the camera for advertising no video. That makes 8323 a genuinely
useful hostile case: a consumer should either decrypt it or *say* it cannot,
rather than going quiet.

So: test against **8322** for "does my NVR support RTSPS cameras", and against
**8323** for "does my NVR fail honestly when it meets SRTP".

(`-tls_verify 0` because modern ffmpeg verifies TLS certificates by default
and this one is self-signed — the same accommodation your NVR needs to make
for real RTSPS cameras.)

## Forcing the transport: `rtspt://`, `rtspst://`

Beyond `rtsp`/`rtsps`, RTSP clients commonly accept a family of schemes that
*pin the media transport* — the spelling is systematic: an `s` after `rtsp`
means TLS, and a trailing letter selects the transport (`t` interleaved TCP,
`u` UDP, `h` tunnelled over HTTP):

| Scheme | Means |
|---|---|
| `rtspt://<host>:8554/cam3-main` | plaintext, **TCP only** (no UDP fallback) |
| `rtspst://<host>:8322/cam3-main` | **TLS + TCP only** — the strict form for a camera behind a firewall |
| `rtspsu://<host>:8322/cam3-main` | TLS + UDP media |

Camera 3 exists to be consumed this way. Nothing changes server-side — these
are client schemes — but they are worth testing explicitly, because a consumer
that only pattern-matches `rtsps://` will reject `rtspst://` outright or, worse,
accept it and then quietly drop the TLS settings.

```sh
ffprobe -rtsp_transport tcp -tls_verify 0 -i rtsps://localhost:8322/cam3-main
```

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

Built as the test rig for testing.

## License

MIT — see [LICENSE](LICENSE).
