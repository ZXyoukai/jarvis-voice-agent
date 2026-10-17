# jarvis-voice-agent

Local, open-source, voice-first AI assistant in the Jarvis style. It runs on your own machine: you speak, it transcribes locally (STT), asks OpenRouter using **free models only** with an ordered fallback sequence, and speaks the answer back (TTS). Session context is persisted in SQLite.

## Features (planned)

- Voice loop: push-to-talk or hands-free, live transcript, interrupt to stop speaking
- Local speech-to-text (Whisper) and text-to-speech (browser voices or Piper)
- OpenRouter with a configurable list of `:free` models, tried in order on timeout, rate limit or error
- Paid models rejected server side
- SQLite for conversation and session persistence
- Single `.env` configuration, one command to start, works on macOS, Linux and Windows

## Configuration

```env
OPENROUTER_API_KEY=
OPENROUTER_FREE_MODELS=model-a:free,model-b:free,model-c:free
DATABASE_PATH=./data/jarvis.db
```

## Status

Scaffold only. The application is being built and will land here.

## License

MIT
