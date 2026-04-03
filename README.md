# A444 — Harmonic Pitch Processor

> A real-time audio pitch shifter built on a from-scratch Phase Vocoder engine.  
> Shifts all audio **+16 cents** — moving the concert pitch reference from A=440 Hz to **A=444 Hz**.

**[Live Demo](#https://github.com/jharnois444/a444-pitch-processor/actions)** · [How It Works](#algorithm) · [Usage](#usage)

---

## Why A=444 Hz?

Standard concert pitch is tuned to A=440 Hz. A=444 Hz is an alternative reference used in some orchestras, recording studios, and acoustic research contexts. The interval between them is exactly **+16.07 cents** — a microtonally small but perceptually meaningful shift.

```
A = 440 Hz  →  A = 444 Hz
Δ = 16.07 cents
Ratio = 2^(16/1200) ≈ 1.009257
```

---

## Features

- 🎙️ **Real-time microphone input** — live pitch shift with ScriptProcessor
- 📁 **File mode** — load any audio file, render offline, export as WAV
- 📊 **Live FFT spectrum analyzer** — 2048-bin, Hann-windowed
- 〰️ **Waveform oscilloscope** — real-time time-domain view
- 📥 **WAV export** — custom PCM encoder, no dependencies
- ⚡ **Zero dependencies** — vanilla JS, Web Audio API only

---

## Algorithm

### Phase Vocoder

The Phase Vocoder is the gold standard for high-quality pitch shifting without altering playback speed. It operates in the frequency domain, manipulating the *phase* of each spectral bin independently to achieve pitch transposition while preserving temporal structure.

```
Input Signal
     │
     ▼
┌─────────────────┐
│  Analysis STFT  │  ← Hann window, N=2048, hop=512 (75% overlap)
└────────┬────────┘
         │  Complex spectrum X[k] = mag[k] · e^(i·φ[k])
         ▼
┌─────────────────────────────────────────────────────┐
│  True Frequency Estimation (Phase Unwrapping)       │
│                                                     │
│  Δφ[k] = φ[k] - φ_prev[k] - 2π·k·H_a / N          │
│  Δφ[k] = wrap(Δφ[k])   ← constrain to [-π, π]      │
│  ω[k]  = 2π·k/N + Δφ[k]/H_a                        │
└────────┬────────────────────────────────────────────┘
         │  True instantaneous frequency ω[k]
         ▼
┌─────────────────────────────────────────────────────┐
│  Phase Accumulation (Synthesis)                     │
│                                                     │
│  H_s = round(H_a · pitchRatio)                      │
│  Φ[k] += H_s · ω[k]                                │
└────────┬────────────────────────────────────────────┘
         │  Synthesised phase Φ[k]
         ▼
┌──────────────────┐
│  Synthesis IFFT  │  ← Reconstruct from mag[k] · e^(i·Φ[k])
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────┐
│  Overlap-Add (OLA)               │  ← Hann window, normalised
└──────────────────────────────────┘
         │
         ▼
   Output Signal (pitch-shifted, tempo-preserved)
```

### Key Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| FFT size (N) | 2048 bins | Frequency resolution ≈ 21.5 Hz @ 44.1kHz |
| Overlap factor | 4× (75%) | H_a = 512 samples |
| Window | Hann | Minimises spectral leakage |
| Pitch ratio | 1.009257 | `2^(16/1200)` |
| Synthesis hop | H_s = round(H_a × ratio) | ≈ 517 samples |

### FFT Implementation

The FFT is implemented from scratch as an **iterative Cooley-Tukey Radix-2 DIT (Decimation In Time)** algorithm — no libraries, no WebAssembly.

```javascript
// Bit-reversal permutation + butterfly operations
function fft(re, im, inverse) {
  const N = re.length;
  // Bit-reversal
  let j = 0;
  for (let i = 1; i < N; i++) {
    let bit = N >> 1;
    for (; j & bit; bit >>= 1) j ^= bit;
    j ^= bit;
    if (i < j) { swap(re, i, j); swap(im, i, j); }
  }
  // Butterfly stages
  for (let len = 2; len <= N; len <<= 1) {
    const ang = 2π / len * (inverse ? 1 : -1);
    // ... twiddle factors and butterfly operations
  }
  if (inverse) scale(re, im, 1/N);
}
```

**Complexity:** O(N log N) — for N=2048, that's ~22,000 operations per frame vs ~4.2M for naïve DFT.

### Phase Unwrapping

The critical step that separates a Phase Vocoder from a naive frequency-bin stretch. Raw phase differences between frames contain **phase wrapping** — discontinuities of ±2π that must be removed to recover the true instantaneous frequency:

```
Δφ_raw[k]  = φ[k] - φ_prev[k] - expected_advance
expected   = 2π · k · H_a / N
Δφ_wrapped = Δφ_raw - 2π · round(Δφ_raw / 2π)   ← wrap to [-π, π]
ω_true[k]  = 2π·k/N + Δφ_wrapped / H_a
```

Without phase unwrapping, transients smear and the output sounds "phasey" or metallic.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Vanilla JavaScript (ES2020) |
| Audio I/O | Web Audio API (`AudioContext`, `ScriptProcessor`) |
| DSP | Custom FFT + Phase Vocoder (no libraries) |
| Visualisation | Canvas 2D API |
| Encoding | Custom PCM→WAV encoder |
| Dependencies | **None** |

---

## Usage

### Browser (recommended)

Just open `index.html` in any modern browser. No build step, no npm install.

```bash
# Optional: serve locally to avoid file:// restrictions on mic access
npx serve .
# or
python3 -m http.server 8080
```

### Microphone Mode

1. Click **◉ MICROPHONE**
2. Click **▶ ENGAGE**
3. Grant microphone permission — your mic input is pitch-shifted in real time and routed to output

> **Note:** disable system echo cancellation and monitor through headphones to avoid feedback.

### File Mode

1. Click **⊞ LOAD AUDIO FILE** — supports MP3, WAV, OGG, FLAC, AAC
2. Click **▶ ENGAGE** to render (offline, fast)
3. Click **↓ RENDER & DOWNLOAD** to export a 16-bit PCM WAV

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│                  Web Audio Graph                 │
│                                                  │
│  MediaStream    AnalyserNode    ScriptProcessor  │
│  (mic input) ──►  (FFT viz) ──► (phase vocoder) ──► Destination
│                                                  │
│  AudioBuffer                                     │
│  (file input) ──────────────────────────────────►│
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│               Phase Vocoder Engine               │
│                                                  │
│  makeHann()    →  Hann window precomputation     │
│  fft()         →  Radix-2 Cooley-Tukey (fwd+inv) │
│  processSingleFrame()  →  Real-time frame proc   │
│  phaseVocoder()        →  Offline full-buffer    │
│  encodeWav()           →  PCM WAV serialiser     │
└──────────────────────────────────────────────────┘
```

---

## Limitations & Future Work

- **ScriptProcessor** is deprecated in favour of `AudioWorklet` — a Worklet port would reduce latency and run off the main thread
- Pitch shifting introduces minor transient smearing inherent to Phase Vocoder; a **transient detection + preservation** layer (as in élastique or Rubber Band Library) would improve percussive audio
- **Polyphonic pitch detection** (YIN or CREPE) could display the detected fundamental in real time
- A **native Electron wrapper** with `naudiodon` (PortAudio) would enable system audio capture via BlackHole/SoundFlower for OS-wide pitch shifting

---

## References

- Laroche, J. & Dolson, M. (1999). *Improved Phase Vocoder Time-Scale Modification of Audio*. IEEE Transactions on Speech and Audio Processing.
- Zölzer, U. (2011). *DAFX: Digital Audio Effects* (2nd ed.). Wiley.
- Cooley, J.W. & Tukey, J.W. (1965). *An Algorithm for the Machine Calculation of Complex Fourier Series*. Mathematics of Computation.

---

## License

MIT — do whatever you want with it.
