# Audio Evaluation Pack (`audio`)

The **Audio Evaluation Pack** provides production-ready metric templates for evaluating speech, Text-to-Speech (TTS), voice acting, acoustic cleanliness, transcript accuracy, and hybrid DSP workflows using Gemini LLM-as-a-Judge via [Mizan](https://github.com/ghchinoy/mizan).

- **Namespace:** `audio` (template IDs are prefixed with `audio/`)
- **Maintainers:** `mizan-audio-team`
- **License:** Apache-2.0
- **Default Judge Model:** `gemini-2.5-flash` (or override with `--model gemini-2.5-pro` / `gemini-3.5-flash`)

---

## Template Catalog

| Template ID | Kind | Modalities | Description |
| :--- | :--- | :--- | :--- |
| `audio/speech-clarity-pointwise` | `pointwise` | `audio` | Rates speech intelligibility, acoustic cleanliness, and background noise on an explicit 1–5 scale. |
| `audio/transcript-fidelity-pointwise` | `pointwise` | `audio`, `text` | Compares spoken audio against a reference transcript to identify dropped words, hallucinations, and mispronunciations. |
| `audio/tts-naturalness-comparison` | `pairwise` | `audio` | Side-by-side comparison of two TTS or voice generation clips for prosody, breathing, natural cadence, and artifacts. |
| `audio/voice-talent-rubric` | `rubric` | `audio` | Multi-criterion scorecard evaluating Articulation, Pacing, Acoustic Cleanliness, and Expressiveness. |
| `audio/audio-acoustic-triage-schema` | `custom_schema` | `audio` | Extracts structured JSON including speaker count, noise classification, background noise level, and issue triage. |
| `audio/hybrid-dsp-diagnostics` | `pointwise` | `audio`, `text` | Interprets pre-computed DSP metrics (ViSQOL, SNR, PESQ) alongside listening to the audio to explain acoustic degradation. |

---

## Quickstart & Setup

### 1. Configure Mizan for Multimodal Audio

To evaluate local audio files (`.wav`, `.mp3`, `.m4a`, `.aac`, `.flac`, `.ogg`), configure your GCP project and Google Cloud Storage staging bucket:

```bash
# Set your GCP Project and Region
mizan config set project-id <your-gcp-project-id>
mizan config set location us-central1

# Configure GCS staging bucket (required for auto-staging local --file assets)
mizan config set staging-bucket gs://<your-staging-bucket>
```

### 2. Import this Pack

Import the audio templates from this repository into your local Mizan registry:

```bash
# Import all templates in the audio namespace
mizan registry import <path-to-mizan-templates> --namespace audio

# Verify the imported templates
mizan registry list --namespace audio
```

---

## Running the Audio Templates

### 1. Speech Clarity & Acoustic Quality (`pointwise`)

Evaluates the overall acoustic quality and intelligibility of an audio recording:

```bash
# Local audio file (automatically staged to GCS):
mizan eval run --metric audio/speech-clarity-pointwise \
  --file recording=/path/to/speech.wav

# Or using a pre-staged GCS asset:
mizan eval run --metric audio/speech-clarity-pointwise \
  --gcs recording=gs://cloud-samples-data/generative-ai/audio/coffee_order.wav
```

### 2. Audio Transcript Fidelity (`pointwise` audio + text)

Validates that spoken words accurately match the reference text without omissions or hallucinations:

```bash
mizan eval run --metric audio/transcript-fidelity-pointwise \
  --file recording=/path/to/narration.mp3 \
  --field reference_transcript="Welcome to our annual product keynote presentation."
```

### 3. TTS Voice Naturalness Comparison (`pairwise`)

Compares two synthesized audio voices side-by-side:

```bash
mizan eval pairwise --metric audio/tts-naturalness-comparison \
  --file baseline_audio=/path/to/tts_model_v1.wav \
  --file candidate_audio=/path/to/tts_model_v2.wav
```

*Tip:* Trust the returned `Choice` (`BASELINE`, `CANDIDATE`, or `TIE`) as the authoritative de-biased verdict.

### 4. Voice Talent Rubric Scorecard (`rubric`)

Evaluates narration across atomic criteria (articulation, pacing, acoustic cleanliness, expressiveness):

```bash
# Run with --rubric-detail for per-criterion scoring breakdown:
mizan eval run --metric audio/voice-talent-rubric \
  --file narration_audio=/path/to/podcast_intro.wav \
  --rubric-detail --rubric-scale 1-5
```

### 5. Structured Acoustic Triage (`custom_schema`)

Extracts machine-parseable JSON properties from an audio recording:

```bash
mizan eval run --metric audio/audio-acoustic-triage-schema \
  --file audio_file=/path/to/field_recording.m4a \
  -o json
```

---

## Using Mizan with Classic Audio Metrics (ViSQOL, PEAQ, PESQ, SNR)

### Classic DSP Metrics vs. LLM-as-a-Judge

| Evaluation Dimension | Classic DSP Metrics (ViSQOL, PEAQ, PESQ, SNR) | Mizan (Gemini LLM-as-a-Judge) |
| :--- | :--- | :--- |
| **Measurement Type** | Deterministic mathematical / psychoacoustic signal distance. | Semantic, perceptual, contextual reasoning. |
| **Reference Requirement** | Requires exact time-aligned reference waveform (except non-intrusive SNR). | No reference waveform needed; accepts transcripts, guidelines, or zero reference. |
| **Strengths** | Exact packet loss, bit-rate degradation, clipping detection, SNR (dB). | Intelligibility, natural prosody, emotional tone, accent/pronunciation, halluncinations, dropped words. |
| **Output** | Single scalar number (e.g., MOS-LQO 1.0–4.75). | Qualitative `Explanation` explaining *why* defects exist + overall score or structured triage. |

### Recommended Integration Patterns

#### Pattern A: Hybrid Contextual Diagnostics (The `audio/hybrid-dsp-diagnostics` template)

In this pattern, your data processing pipeline computes standard signal metrics (e.g. using `visqol` or `scipy.signal`) and passes both the **audio file** and the **numeric measurements** to Mizan. Gemini acts as an expert audio engineer correlating the numbers with what is audible.

```bash
# Example: Pass computed DSP measurements into Mizan
mizan eval run --metric audio/hybrid-dsp-diagnostics \
  --file audio_clip=/path/to/degraded_sample.wav \
  --field measured_visqol="2.8" \
  --field measured_snr_db="12.4" \
  --field codec_or_condition="Opus 12kbps with 5% simulated packet loss"
```

#### Pattern B: Two-Stage Automated Audio QA Pipeline

When evaluating thousands of audio generations, combining classic DSP metrics with Mizan provides optimal cost, speed, and qualitative depth:

```
                  ┌────────────────────────────────────────┐
                  │ 1,000 Generated Audio Clips (TTS / ASR)│
                  └───────────────────┬────────────────────┘
                                      │
                                      ▼
                  ┌────────────────────────────────────────┐
                  │ Stage 1: Fast Local DSP Gating Filter  │
                  │ (Calculate SNR / RMS Energy / Clipping)│
                  └───────────┬────────────────┬───────────┘
                              │                │
            SNR < 10 dB or    │                │ Passes basic
            Severe Clipping   │                │ signal checks
                              ▼                ▼
                  ┌────────────────┐   ┌────────────────────────────────────────┐
                  │ Auto-Fail      │   │ Stage 2: Mizan Perceptual & QA Eval    │
                  │ (Instant Reject│   │ - audio/transcript-fidelity-pointwise  │
                  │  no API cost)  │   │ - audio/voice-talent-rubric            │
                  └────────────────┘   │ - audio/tts-naturalness-comparison    │
                                       └───────────────────┬────────────────────┘
                                                           │
                                                           ▼
                                       ┌────────────────────────────────────────┐
                                       │ Comprehensive Human-Calibrated QA Store│
                                       │ (mizan results list / results show)    │
                                       └────────────────────────────────────────┘
```

#### Example Automation Script (Python)

```python
import subprocess
import json
import numpy as np
import soundfile as sf

def compute_snr(audio_path: str) -> float:
    """Simple signal-to-noise ratio estimation."""
    data, _ = sf.read(audio_path)
    signal_power = np.mean(data ** 2)
    noise_est = np.mean(np.abs(np.diff(data)) ** 2) / 2
    if noise_est == 0:
        return 100.0
    return float(10 * np.log10(signal_power / noise_est))

def evaluate_audio_clip(audio_path: str, transcript: str):
    # 1. Fast DSP check
    snr = compute_snr(audio_path)
    if snr < 5.0:
        print(f"[FAIL] {audio_path}: SNR too low ({snr:.1f} dB)")
        return

    # 2. Semantic and Perceptual QA with Mizan
    cmd = [
        "mizan", "eval", "run",
        "--metric", "audio/transcript-fidelity-pointwise",
        "--file", f"recording={audio_path}",
        "--field", f"reference_transcript={transcript}",
        "-o", "json"
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode == 0:
        eval_data = json.loads(result.stdout)
        print(f"[SUCCESS] Score: {eval_data.get('Score')} | Explanation: {eval_data.get('Explanation')}")
    else:
        print(f"[ERROR] Mizan evaluation failed: {result.stderr}")

if __name__ == "__main__":
    evaluate_audio_clip("sample.wav", "Hello, welcome to our service.")
```
