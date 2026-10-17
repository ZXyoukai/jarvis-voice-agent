# Runbook: running jarvis-voice-agent offline

This guide explains how to run Jarvis with no internet connection: a local model server instead of OpenRouter, local speech-to-text with Whisper and local text-to-speech with Piper. It is documentation only. Values marked "approximate" depend on quantization, context length and runtime.

## 1. Overview

| Layer | Online (default) | Offline |
| --- | --- | --- |
| LLM | OpenRouter, ordered list of `:free` models | Ollama or LM Studio on `localhost` |
| STT | Local Whisper (Web Speech API optional) | Local Whisper only |
| TTS | Browser voices | Piper (or browser voices installed on the OS) |
| Storage | SQLite | SQLite (unchanged) |

Both Ollama and LM Studio expose an OpenAI-compatible API, so the app talks to them the same way it talks to OpenRouter. Only the base URL, the key and the model list change.

## 2. Choose a local model

Recommended small models for a voice assistant (short, fast answers):

| Model | Ollama tag | Parameters | Download (Q4, approx.) | RAM recommended (approx.) | Notes |
| --- | --- | --- | --- | --- | --- |
| Llama 3.2 3B Instruct | `llama3.2:3b` | 3B | ~2 GB | 4 to 8 GB | Fastest, good for short spoken answers |
| Qwen2.5 7B Instruct | `qwen2.5:7b` | 7B | ~4.7 GB | 8 to 10 GB | Better reasoning and Portuguese, supports tool calling |
| Llama 3.1 8B Instruct | `llama3.1:8b` | 8B | ~4.9 GB | 8 to 12 GB | Alternative to Qwen 7B |

RAM by model size (Q4 quantization, short context, approximate):

| Model size | Minimum RAM | Comfortable RAM |
| --- | --- | --- |
| 1B to 3B | 4 GB | 8 GB |
| 7B to 8B | 8 GB | 16 GB |
| 13B to 14B | 16 GB | 24 to 32 GB |
| 30B to 34B | 32 GB | 48 GB |
| 70B | 48 GB | 64 GB or more |

Rules of thumb:
- Total free memory (RAM, or RAM plus VRAM) should exceed the model file size plus 1 to 2 GB for the runtime, KV cache and the app itself.
- Whisper and Piper also need memory (see sections 3 and 4). Budget them in.
- A GPU or Apple Silicon makes answers much faster. On CPU only, prefer 3B.

## 3. Install a local model server

### Option A: Ollama

1. Install from [ollama.com](https://ollama.com).
2. Pull the models while you still have internet:
   ```bash
   ollama pull llama3.2:3b
   ollama pull qwen2.5:7b
   ```
3. Ollama serves on `http://localhost:11434`. The OpenAI-compatible endpoint is `http://localhost:11434/v1`.
4. Check it:
   ```bash
   curl http://localhost:11434/v1/models
   ```

### Option B: LM Studio

1. Install from [lmstudio.ai](https://lmstudio.ai) and download a GGUF of one of the models above (Q4_K_M is a good default).
2. Start the local server in LM Studio (Developer tab). It serves on `http://localhost:1234/v1` by default.
3. Check it:
   ```bash
   curl http://localhost:1234/v1/models
   ```

## 4. Offline speech-to-text (Whisper)

Use Whisper locally. Two common setups:
- **In the browser** with transformers.js: the model files are cached by the browser after the first load. Load the app once while online so the files are cached, or serve the model files from the app's own `public/models` folder.
- **On the local server** with whisper.cpp or faster-whisper: download the model file ahead of time.

Whisper model sizes (approximate):

| Model | Disk | Notes |
| --- | --- | --- |
| tiny | ~75 MB | Fastest, lowest accuracy |
| base | ~142 MB | Good default for short commands |
| small | ~466 MB | Better accuracy for Portuguese |
| medium | ~1.5 GB | Slow on CPU |

Recommendation: `base` or `small` multilingual. Set the language explicitly (`pt`) for better results. Do not rely on the browser Web Speech API offline, because Chrome sends audio to an online service.

## 5. Offline text-to-speech (Piper)

1. Download the Piper binary for your OS from the [Piper releases](https://github.com/rhasspy/piper/releases).
2. Download a voice (an `.onnx` file plus its `.onnx.json`) from the [Piper voices list](https://huggingface.co/rhasspy/piper-voices). Portuguese options include `pt_PT` and `pt_BR` voices in the `medium` quality.
3. Test it:
   ```bash
   echo "Olá, eu sou o Jarvis." | ./piper --model pt_PT-<voice>-medium.onnx --output_file test.wav
   ```
4. Point the app to Piper (see section 6). Browser `speechSynthesis` also works offline when the OS has local voices installed, but quality varies.

Text stays the primary channel. If Piper is not installed, replies still appear as text.

## 6. Configure the app for local endpoints

Planned `.env` settings for offline mode (names may change with the implementation, keep this runbook in sync):

```env
LLM_PROVIDER=local
LLM_BASE_URL=http://localhost:11434/v1
LLM_API_KEY=ollama
LLM_MODELS=qwen2.5:7b,llama3.2:3b
LLM_TIMEOUT_MS=30000

STT_ENGINE=whisper
STT_MODEL=base
STT_LANGUAGE=pt

TTS_ENGINE=piper
PIPER_BIN=./bin/piper
PIPER_VOICE=./voices/pt_PT-<voice>-medium.onnx

DATABASE_PATH=./data/jarvis.db
```

For LM Studio, use `LLM_BASE_URL=http://localhost:1234/v1` and the model id LM Studio shows.

How the model sequence changes:
- Online, `OPENROUTER_FREE_MODELS` is an ordered fallback list of free models, and paid models are rejected.
- Offline, `LLM_MODELS` is an ordered list of models installed locally. The same fallback loop applies (timeout or error moves to the next model), but there are no rate limits and no cost checks.
- Put the larger model first for quality and the 3B model second as a fast fallback. On low memory, list only the 3B model, because loading two models can exhaust RAM.

Verification checklist:
1. Disconnect from the internet.
2. `curl $LLM_BASE_URL/models` lists your models.
3. Start the app and open `http://localhost:<port>`.
4. Speak a short question. The transcript appears, the reply appears as text, then it is spoken.
5. The UI shows which local model answered.

## 7. Hardware that is NOT sufficient

Do not use these models for a local Jarvis on a normal computer:

| Model | Why it is excluded |
| --- | --- |
| Muse Spark Open (Meta, 816B total, 42B active, MoE) | Needs about 256 to 315 GB of memory even at 2-bit, about 470 to 500 GB at 4-bit, and Meta recommends roughly 1 TB of unified memory (two Mac Studio M5 Ultra with 512 GB each) |
| Any 70B model | Needs 48 GB or more of RAM or VRAM |
| 13B to 34B models on a 16 GB laptop | Too slow or will not load with the STT and TTS running |

A machine with 8 GB of RAM should run `llama3.2:3b` plus Whisper `base` plus Piper. A machine with 16 GB can run `qwen2.5:7b` comfortably.

## 8. Troubleshooting

| Symptom | Fix |
| --- | --- |
| Very slow replies | Use a smaller model or quantization, shorten the context, close other apps |
| Out of memory | List only one model in `LLM_MODELS` |
| No microphone | Open the app on `localhost`, which browsers treat as a secure context, and allow the permission |
| Transcription in the wrong language | Set `STT_LANGUAGE=pt` |
| No voice | Check `PIPER_BIN` and `PIPER_VOICE` paths. Text replies still work |

## Sources

- [Muse Spark Open announcement (Meta)](https://research.meta.ai/blog/muse-spark-open)
- [Muse Spark Open hardware requirements (Meta)](https://dev.meta.ai/docs/muse-spark-open/deploy)
- [Ollama OpenAI compatibility](https://github.com/ollama/ollama/blob/main/docs/openai.md)
- [Piper](https://github.com/rhasspy/piper)
