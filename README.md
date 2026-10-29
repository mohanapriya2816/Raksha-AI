# Voice Veil – Live Deepfake Call Shield (Team AVENGERS, Dexathon 2026) — v3.1

## Run on Windows (PowerShell)
Run from the folder that contains `backend`, `frontend` and `run.bat` (if `dir` does not show them, `cd` into the inner `voice-veil` folder):

    .\run.bat

Then open http://localhost:8000 in Chrome or Edge. First start installs packages and trains the prototype model (~1–2 min).
No API keys, cloud services or paid APIs are used.

Manual start (Python 3.10–3.13 recommended; if pip wants a C++ compiler you are on a Python too new for some wheels – install 3.12):

    cd backend
    python -m pip install -r requirements.txt
    python -m uvicorn app.main:app --port 8000

## Tests
    cd backend
    python -m pytest -q tests          # 17 backend tests
    python bench.py                    # compute latency (features vs model)
    # with the server running in another window:
    python latency_e2e.py              # chunk->verdict and onset->warning latency over the real WebSocket
    cd ..\e2e
    npm install
    node --test unit.test.mjs          # resampler + worklet unit tests (Node 18+)
    cd ..\backend; python make_test_audio.py; cd ..\e2e
    $env:CHROME_PATH="C:\Program Files\Google\Chrome\Application\chrome.exe"   # or msedge.exe
    node run.mjs .\audio\natural_48000.wav    # drives the real dashboard in Chrome (server must be running on :8765,
                                              # start it with:  cd backend; python -m uvicorn app.main:app --port 8765)

## How it works
Browser (mic or shared-tab audio) -> AudioWorklet averages all channels to mono -> streaming resampler to 16 kHz -> Float32 chunks (~130 ms)
over a local WebSocket -> 1 s window, one decision per 0.5 s -> features (32 ms frames, 10 ms hop: MFCC, spectral, pitch/jitter/shimmer, pauses)
-> **RandomForest (scikit-learn, CPU)** -> fused with a pitch cue and optional Voice ID match -> rolling median -> HIGH after 3 consecutive low decisions.
States: Monitoring · Inconclusive (clipping, invalid audio, analysis failure) · Low risk indicators · Elevated · High. Never "safe".
The dashboard shows what the backend really receives (rate, format, level, clipping, browser capture rate/channels) and warns on a rate mismatch.

## What is VERIFIED and what is NOT
Verified by running (Linux sandbox, headless Chromium 153 with Chrome's *fake audio device* playing a WAV file as the microphone):
* Start/Stop, restart, rapid start/stop x4, device list, sensitivity (UI and server threshold), waveform + timeline drawing, demo mode labelling,
  clear-events (with confirm dialog), mobile width (390 px) without horizontal scroll, no console/page errors, no failed requests: 33/33 checks.
* Audio path: browser captured 44.1 kHz stereo -> backend measured 15.9–16.0 kHz, float32 mono, 0 dropped frames, non-silent, not clipped.
* Mic tracks are released on Stop and no frames are sent afterwards.
* 17 backend tests, 8 frontend unit tests.

NOT verified: a real microphone or sound card, Edge, Firefox/Safari, Windows, phones, real tab-audio capture (`getDisplayMedia`), real phone calls,
detection of real cloned voices (ElevenLabs etc.), and any accuracy on real speech.

## Latency (measured; sandbox, loopback — expect different numbers on your laptop)
| What | Result |
|---|---|
| Feature extraction | ~3–5 ms median |
| Model inference | ~10–18 ms median |
| Full compute per decision | 13.8 ms median, 20.3 ms p95 (bench.py, 300 windows) — your run: 24.6 / 29.8 ms |
| Chunk sent -> verdict received (WebSocket, loopback) | 19 ms median, 26 ms p95 (latency_e2e.py); 27–33 ms in the browser |
| Audio onset -> first Elevated | 0.7–1.7 s |
| Audio onset -> HIGH warning | 2.2–2.7 s (natural -> synthetic switch; 3 runs per setting) |
Not measured: microphone/OS audio-input latency. **A sub-200 ms end-to-end claim is NOT supported**; a warning needs a 1 s window plus 3 consecutive decisions.
At sensitivity 100 the natural audio produced false HIGH decisions in 2 of 3 runs — the sensitivity slider trades speed for false alarms.

## Honest model status
* The bundled model learns from **generated synthetic signals**. Its "0.89 accuracy" is on the same generator and says nothing about real clones.
  It is a labeled PROTOTYPE (banner in the UI). It is not PyTorch and not ONNX.
* A pretrained model was investigated: AASIST (MIT, github.com/clovaai/aasist) has weights, but no labeled real-vs-clone speech was reachable
  for evaluation, it needs PyTorch, and it was trained on ASVspoof 2019 and is known to generalise poorly to modern clones/phone audio. Not integrated.
* To train/evaluate on real data: put WAVs in `data/real/` and `data/fake/` (e.g. ASVspoof, asvspoof.org), then `cd backend; python train.py`.
  Train/test are split **by file** (a leakage bug in the random-window split was fixed). Then run `python evaluate.py <real_dir> <fake_dir>`
  on speakers/recordings NOT used for training; it reports accuracy and EER and warns if there are too few files.
* Voice ID is stable for the same voice but unreliable for similar-pitch voices (synthetic tests only).
* "Timing stability" = regularity of chunk arrival between browser and local server, not carrier network jitter.
* Browsers cannot capture normal phone calls; use speaker mode or tab audio for web calls.
* Privacy: audio stays in memory on this machine; only `{time, score}` of HIGH events is written to `logs/events.jsonl` (Clear button deletes it).
  The backend stores no device names. No raw audio is persisted.

## Suggested PowerPoint wording
* Slide 2/3 "scored in short frames (20 ms)": "Audio is analysed in 1-second windows (32 ms feature frames), with one decision every 0.5 s."
* "under 200 ms per decision": "About 15–25 ms of compute per decision on a laptop CPU; a warning appears about 2–3 s after a voice change because it needs several consecutive windows (measured)."
* "PyTorch classifier / ONNX Runtime": "A lightweight scikit-learn model runs on the CPU; ONNX export is planned."
* "Trained on public real vs cloned speech": "The current prototype is trained on synthetic signals; training and evaluation on ASVspoof with held-out speakers is the next step."
* "Network-jitter cues": "Timing-stability cue (audio chunk regularity)."
* "Runs on existing phones": "Runs in a laptop/phone browser; native phone-call capture needs a mobile app (future work)."
* Keep: on-device processing, no raw audio stored, only time + score logged, fused cues, rolling smoothing, adjustable sensitivity.
