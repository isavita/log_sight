# macOS Guide – Offline Meeting Transcripts with Whisper & WhisperX

Need a fast transcript of your Zoom/Teams/Meet session?  
These steps show how to **convert video → clean WAV → text** on an Apple-Silicon Mac (M1/M2), entirely offline. Whisper gives plain transcripts; WhisperX adds **speaker diarization** so you know who spoke when.

---

## 1 · Install FFmpeg (audio extraction)

```bash
brew install ffmpeg
```

---

## 2 · Convert any recording to Whisper-friendly audio

```bash
ffmpeg -i meeting.mp4 -acodec pcm_s16le -ac 1 -ar 16000 meeting.wav
```

*Why 16 kHz mono?*
Whisper is trained on 16 kHz single-channel speech, so accuracy and speed are best when the input matches.

---

## 3 · Quick transcript with OpenAI Whisper

```bash
pip install openai-whisper         # one-time install

whisper meeting.wav \
  --model medium \                 # swap to small/large as desired
  --language en \                  # force English decoding
  --output_format txt \            # plain text
  --output_format srt \            # subtitles
  --output_format json \           # structured segments
  --verbose True
```

Outputs land next to your WAV (`meeting.txt`, `meeting.srt`, `meeting.json`).
Medium fits in <4 GB VRAM and usually transcribes 1 h audio in \~10 min on an M2.

---

## 4 · Add Speaker Labels with WhisperX

### 4.1 Create a clean Python env (recommended)

```bash
python3 -m venv ~/venvs/whisperx && source ~/venvs/whisperx/bin/activate
pip install --extra-index-url https://download.pytorch.org/whl/cpu \
            torch==2.5.1 torchvision==0.20.1 torchaudio==2.5.1 \
            numpy==2.0.2 pandas==2.2.3 pyarrow==16.1.0 \
            whisperx==3.3.4 "pyannote-audio>=3.3.2"
```

### 4.2 Authenticate once for the diarization model

```bash
huggingface-cli login     # paste a free User token with “Read” scope
```

### 4.3 Run full pipeline

```bash
whisperx meeting.wav \
  --model medium \
  --language en \
  --diarize \                     # enable speaker separation
  --hf_token "$HF_TOKEN" \        # must match the login token
  --device cpu \                  # fastest on M-series with int8
  --compute_type int8 \
  --min_speakers 2 --max_speakers 8 \
  --output_dir out \
  --output_format vtt
```

First run downloads the Pyannote diarization checkpoint (≈140 MB).
Result: `out/meeting.vtt` with cues like:

```
00:00:02.292 --> 00:00:12.384  SPEAKER_00: Welcome everyone…
00:00:12.556 --> 00:00:18.987  SPEAKER_01: Thanks for joining…
```
