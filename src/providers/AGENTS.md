# AGENTS.md — src/providers

**Parent**: `./AGENTS.md` (read first)

## OVERVIEW
AI model provider integrations — OpenAI, Anthropic, Ollama, llama.cpp, vLLM, OpenRouter.

## STRUCTURE
```
src/providers/
├── traits.rs       # Provider trait definition
├── mod.rs         # Factory + register_provider! macro
├── openai.rs      # OpenAI API
├── anthropic.rs   # Anthropic API
├── ollama.rs      # Ollama (local + remote)
├── llama.rs       # llama.cpp server
├── vllm.rs        # vLLM server
├── openrouter.rs  # OpenRouter aggregator
└── *.rs           # 15+ provider implementations
```

## WHERE TO LOOK
| Task | Location |
|------|----------|
| Add new provider | Implement `Provider` trait → register in `mod.rs` |
| Provider trait | `traits.rs` — `chat()`, `chat_stream()`, `models()` |
| Resilient wrapper | `mod.rs` — retry, timeout, fallback logic |
| Model catalog | `models()` method returns available models |

## CONVENTIONS
- Provider impl: `struct FooProvider; impl Provider for FooProvider { ... }`
- API calls: Use `reqwest` with proper error handling
- Streaming: Implement `Stream` for `chat_stream()`
- Mock: Use `ScriptedProvider`, `RecordingProvider` for tests

## ANTI-PATTERNS
- **NEVER** hardcode API keys
- **NEVER** skip timeout handling
- **NEVER** leak raw request/response in logs

## RISK: MEDIUM
Provider I/O module. Changes require:
- Error propagation tests
- Mock-based unit tests
