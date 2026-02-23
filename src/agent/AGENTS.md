# AGENTS.md — src/agent

**Parent**: `./AGENTS.md` (read first)

## OVERVIEW
Agent orchestration loop — prompt, classifier, dispatcher, memory loader.

## STRUCTURE
```
src/agent/
├── mod.rs           # Module exports
├── loop_.rs         # Main orchestration loop
├── agent.rs         # Agent state machine
├── prompt.rs        # Prompt construction
├── classifier.rs    # Intent/tool classification
├── dispatcher.rs    # Tool dispatch logic
├── memory_loader.rs # Memory recall integration
└── tests.rs         # Integration tests
```

## WHERE TO LOOK
| Task | Location |
|------|----------|
| Main loop | `loop_.rs` — `run_agent_loop()` |
| Tool selection | `classifier.rs` — intent classification |
| Execution | `dispatcher.rs` — tool execution |
| Memory recall | `memory_loader.rs` — context loading |

## CONVENTIONS
- Async-first: all major functions are async
- Message flow: channel → classifier → dispatcher → tools → response
- Error handling: propagate via `AgentError`

## ANTI-PATTERNS
- **NEVER** block on tool execution without timeout
- **NEVER** skip memory context loading for relevant queries
- **NEVER** expose raw tool errors to channels

## RISK: MEDIUM
Orchestration module. Changes require:
- Loop behavior validation
- Integration tests
