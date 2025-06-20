# macOS Guide: Offline Meeting Transcripts with Whisper & WhisperX

Need a fast, offline transcript of your Zoom, Microsoft Teams, or Google Meet recordings on an Apple Silicon Mac (M1/M2/M3)?

This guide outlines the process to:
1.  **Convert** video recordings to a clean WAV audio file.
2.  Generate a **plain text transcript** using OpenAI Whisper.
3.  Optionally, add **speaker diarization** (identifying who spoke when) with WhisperX.

> All processing is done **entirely offline** on your machine.

---

## 1. Install FFmpeg (Audio Extraction)

First, ensure you have FFmpeg installed. FFmpeg is a powerful tool for handling multimedia files and will be used here to extract audio from your video recordings.

```bash
brew install ffmpeg
```

---

## 2. Convert Video Recording to Whisper-Friendly Audio

Next, convert your meeting recording (e.g., `meeting.mp4`) into a WAV audio file format that Whisper is optimized for.

```bash
ffmpeg -i meeting.mp4 -acodec pcm_s16le -ac 1 -ar 16000 meeting.wav
```

> **Why 16 kHz Mono?**
> Whisper is trained on 16 kHz single-channel (mono) speech. Matching this input format generally yields the best accuracy and processing speed.

---

## 3. Generate a Quick Transcript with OpenAI Whisper

With your audio file ready, you can now use OpenAI Whisper to generate a transcript.

**One-time Installation:**
```bash
pip install openai-whisper
```

**Run Whisper:**
```bash
whisper meeting.wav \
  --model medium \                 # Swap to small/large as desired
  --language en \                  # Force English decoding
  --output_format txt \            # Plain text
  --output_format srt \            # Subtitles
  --output_format json \           # Structured segments
  --verbose True
```

**Output Files:**
The transcribed files will be saved in the same directory as your `meeting.wav` file:
*   `meeting.txt`
*   `meeting.srt`
*   `meeting.json`

> **Performance Note:**
> The `medium` model typically fits within 4 GB of VRAM and can transcribe approximately 1 hour of audio in about 10 minutes on an M2 Mac.

---

## 4. Add Speaker Labels with WhisperX

For transcripts where knowing *who* spoke is important, WhisperX can add speaker diarization.

### 4.1 Create a Clean Python Environment (Recommended)

It's good practice to use a virtual environment for Python projects to manage dependencies.

```bash
python3 -m venv ~/venvs/whisperx && source ~/venvs/whisperx/bin/activate
pip install --extra-index-url https://download.pytorch.org/whl/cpu \
            torch==2.5.1 torchvision==0.20.1 torchaudio==2.5.1 \
            numpy==2.0.2 pandas==2.2.3 pyarrow==16.1.0 \
            whisperx==3.3.4 "pyannote-audio>=3.3.2"
```
*(To deactivate this environment later, simply run `deactivate`)*

### 4.2 Authenticate for the Diarization Model (One-Time)

WhisperX uses a diarization model from Hugging Face. You'll need to authenticate once to download it.

```bash
huggingface-cli login     # Paste a free User Access Token with “Read” scope
```
*You can create a token on the [Hugging Face website](https://huggingface.co/settings/tokens).*

### 4.3 Run the Full WhisperX Pipeline

Now, run WhisperX to transcribe and diarize your audio.

```bash
whisperx meeting.wav \
  --model medium \
  --language en \
  --diarize \                     # Enable speaker separation
  --hf_token "$HF_TOKEN" \        # Must match the login token (or ensure you're logged in via huggingface-cli)
  --device cpu \                  # Fastest on M-series with int8 quantization
  --compute_type int8 \
  --min_speakers 2 --max_speakers 8 \ # Adjust as needed
  --output_dir out \
  --output_format vtt```

> **Note:** The first time you run this, WhisperX will download the Pyannote diarization model checkpoint (approximately 140 MB).

**Result:**
The output, including speaker labels, will be saved to `out/meeting.vtt`. It will look something like this:

```vtt
00:00:02.292 --> 00:00:12.384  SPEAKER_00: Welcome everyone…
00:00:12.556 --> 00:00:18.987  SPEAKER_01: Thanks for joining…
```

### 4.4 Troubleshooting Diarization Issues

If you encounter an error during diarization, such as `AttributeError: 'NoneType' object has no attribute 'to'` or messages about failing to download `pyannote/speaker-diarization-3.1`, it's likely due to model access restrictions on Hugging Face.

**Primary Fix: Accept Model User Conditions**

The `pyannote/speaker-diarization-3.1` model (and its dependencies like `pyannote/segmentation-3.0`) are "gated." This means you must accept their terms of use on the Hugging Face website before you can download them, even with a valid token.

1.  **Log in** to your Hugging Face account.
2.  **Accept conditions for `pyannote/speaker-diarization-3.1`**:
    Go to [https://huggingface.co/pyannote/speaker-diarization-3.1](https://huggingface.co/pyannote/speaker-diarization-3.1) and agree to the terms.
3.  **Accept conditions for `pyannote/segmentation-3.0`** (a common dependency):
    Go to [https://huggingface.co/pyannote/segmentation-3.0](https://huggingface.co/pyannote/segmentation-3.0) and agree to the terms.

**Secondary Checks:**

*   **Verify `$HF_TOKEN`:** Ensure the environment variable `$HF_TOKEN` is correctly set in your terminal and matches the token you used with `huggingface-cli login`. You can check it with `echo $HF_TOKEN`.
*   **Retry `huggingface-cli login`:** If in doubt, run `huggingface-cli login` again and re-paste your token.
*   **Version Mismatches (Warnings):** You might see warnings like `Model was trained with pyannote.audio 0.0.1, yours is 3.3.2` or about `torch` versions. While these can sometimes cause issues, the diarization model download failure is typically due to not accepting the user conditions on Hugging Face. Address the model access issue first. The versions specified in the installation command (Section 4.1) are generally chosen for compatibility with WhisperX.

After completing these steps, try running the `whisperx` command again.
