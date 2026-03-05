# Moment

**Moment** is an original music track composed and produced by **Maksym Shymchenko**, paired with an audio-reactive lava lamp visualizer that can export full-resolution video in seconds.

---

## Visualizer — Technical Overview

### Architecture

The entire visualizer is a single `index.html` file with no build step. It uses the browser as a rendering engine and media codec suite.

---

### 1. Audio Analysis (Web Audio API)

When a track is loaded, the audio element is routed through a Web Audio graph:

```
AudioElement
  └── AnalyserNode (full spectrum, FFT 2048)
  └── BiquadFilter lowpass  200 Hz → AnalyserNode (bass)
  └── BiquadFilter bandpass 1 kHz  → AnalyserNode (mid)
  └── BiquadFilter highpass 3 kHz  → AnalyserNode (high)
  └── AudioContext.destination (speakers)
```

Each frame, the RMS (Root Mean Square) energy is computed for all four analysers:

```
RMS = sqrt( (1/N) * sum( (x[i]/255)^2 ) )
```

The raw RMS values are smoothed with a first-order IIR low-pass filter (alpha = 0.09) to avoid jerky response:

```
env = env + (raw - env) * 0.09
```

The resulting `envBass`, `envMid`, `envHigh`, `envFull` values drive all visual effects.

---

### 2. Metaball Field (Implicit Surfaces)

The blobs are rendered using the **metaball / potential field** technique. Each blob contributes a scalar field value at every pixel:

```
field(p) = sum_i( r_i^2 / |p - b_i|^2 )
```

Where `r_i` is the effective radius of blob `i` (scaled by audio energy), and `b_i` is its position.

A pixel is rendered as part of a blob if `field(p) >= threshold`. The threshold decreases with audio energy, making blobs merge more aggressively to the beat.

**Performance optimization:** The field is computed on a downsampled canvas (`ds = ceil(height / 380)` — typically 1/3 to 1/5 resolution), then upscaled with bilinear interpolation. This allows real-time rendering even at 4K export resolution.

---

### 3. Blob Physics

Each blob is a particle with position, velocity, heat, and a sway phase. Physics run at variable delta-time locked to requestAnimationFrame:

**Buoyancy** — hot blobs rise, cold blobs sink:
```
vy -= (heat - 0.45) * buoyancy_coefficient * dt
```

**Heat exchange** — blobs heat up near the bottom, cool near the top:
```
if y > H*0.85: heat += 1.8 * dt
if y < H*0.15: heat -= 1.4 * dt
```

**Sway** — each blob has a unique sinusoidal lateral drift with frequency in [0.3, 0.9] Hz:
```
vx += sin(phase) * 14 * dt + sin(phase * 1.7 + 1.2) * 5 * dt
```

**Audio kick** — bass energy pushes hot blobs upward and adds mild lateral randomness:
```
vy -= kick * 40 * dt * heat
vx += (random - 0.5) * kick * 10 * dt
```

**Pairwise repulsion** — prevents blobs from permanently merging by applying a spring force when centers are closer than `1.9 * (r_i + r_j)`.

**Damping** — exponential velocity decay each frame: `pow(0.88, dt * 60)`.

---

### 4. Coloring

Blob color is interpolated along a lava lamp palette based on per-pixel heat (weighted average of contributing blobs):

```
heat < 0.5: dark red → orange   (rgb: 100,12,5 → 230,80,10)
heat > 0.5: orange → white-pink (rgb: 230,80,10 → 255,240,150)
```

Transparency at each pixel is determined by smooth-step clamping of the field value near the iso-surface.

---

### 5. Post-Processing

After blob rendering, three passes are applied:

1. **Multi-layer glow** — the blob layer is drawn 3 times with increasing blur radii (8–70 px) and `globalCompositeOperation = 'lighter'`, controlled by the Glow slider.
2. **Radial vignette** — a radial gradient from white to black with `'multiply'` blending darkens the edges.
3. **Track name** — drawn at canvas center with three passes: halo (large blur, low alpha), glow (medium blur), and crisp top layer. Font size scales to `min(w * 0.11, h * 0.17)` and pulses with bass energy.

---

### 6. Offline Video Export (WebCodecs API)

Unlike traditional screen recording (which must play in real time), the export pipeline renders faster than real time:

**Step 1 — Audio decode**
The audio source is fetched as an ArrayBuffer and decoded to a raw `AudioBuffer` via `AudioContext.decodeAudioData()`.

**Step 2 — Resample**
The selected segment is resampled to 48 000 Hz (required by Opus) using `OfflineAudioContext`. The `OfflineAudioContext` renders the audio graph synchronously in memory without playing through speakers.

**Step 3 — Per-frame frequency analysis**
A custom Cooley-Tukey FFT (radix-2, in-place) is applied to a 2048-sample Hann-windowed chunk centered at each frame's timestamp. Frequency bins are grouped into bass/mid/high/full bands, and RMS energy is computed for each. This pre-computes all `totalFrames = ceil(duration * 30)` energy values before rendering begins.

**Step 4 — Frame rendering**
Blob physics and field rendering run in a tight loop, one frame at a time (dt = 1/30 s), completely decoupled from real time. Each frame is captured as a `VideoFrame` object from the offscreen canvas.

**Step 5 — Video encoding**
Frames are fed to a `VideoEncoder` (VP9 preferred, VP8 fallback). The encoder runs asynchronously; the queue is flushed every 30 frames to prevent backpressure.

**Step 6 — Audio encoding**
Audio planar float data is chunked and fed to an `AudioEncoder` (Opus codec).

**Step 7 — Muxing**
Video and audio chunks are muxed into a WebM container by `webm-muxer`. On finalization, the entire file is available as an `ArrayBuffer` in memory — no disk I/O during rendering.

**Step 8 — Download**
A temporary Blob URL is created and programmatically clicked to trigger browser download.

**Typical export speed:** a 3-minute Full HD video renders in ~15–30 seconds, versus 3 minutes with real-time recording.

---

### 7. Multi-resolution Rendering

Blobs live in "display canvas space" (the physical pixel dimensions of the visible canvas). When exporting at a different resolution or aspect ratio (e.g., 1080×1920 portrait), the field sampler maps each record canvas pixel back to display canvas space:

```
world_x = (pixel_x * ds / target_w) * display_w
world_y = (pixel_y * ds / target_h) * display_h
```

This ensures blobs always fill the frame correctly regardless of the export resolution or aspect ratio.

---

### Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| [webm-muxer](https://github.com/Vanilagy/webm-muxer) | latest | WebM container muxing |

All other functionality uses native browser APIs: Web Audio API, WebCodecs, Canvas 2D, OfflineAudioContext, MediaStream.

**Minimum browser requirement:** Chrome / Edge 94+ (for WebCodecs). Firefox is not supported for export (no VideoEncoder); playback works in all browsers.

---

## License

Copyright (c) 2026 Maksym Shymchenko. All rights reserved over authorship.

This track is released under a **custom free-use license** with the following terms:

### You are free to:
- Listen, share, stream, and distribute the track in any format
- Use it in personal or commercial projects — videos, films, games, podcasts, art installations, and any other media
- Remix, adapt, or build upon it
- Use it for free, without paying royalties

### Under these conditions:
- **Credit is required.** You must clearly credit the author as **Maksym Shymchenko** wherever the track is used or published. Do not omit or obscure the authorship.
- **No impersonation.** You may not claim, imply, or suggest that you created this track. You may not present it as your own work.
- **Good purposes only.** This track may not be used in content that promotes hatred, violence, discrimination, harassment, or any activity that causes harm to individuals or groups.

### Summary:
> Use it freely. Credit the creator. Never claim it as your own. Use it for good.

---

## About the Track

| Field      | Details                        |
|------------|--------------------------------|
| Title      | Moment                         |
| Artist     | Maksym Shymchenko              |
| Year       | 2026                           |
| Format     | MP3                            |

---

## Contact

For licensing questions, collaborations, or other inquiries, please reach out to Maksym Shymchenko directly.

---

*This license does not affect your statutory rights. The author retains full moral rights over the work at all times.*
