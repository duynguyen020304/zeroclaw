# AGENTS.md — src/tools

**Parent**: `./AGENTS.md` (read first)

## OVERVIEW
Tool execution surface — shell, file, memory, browser, cron, http_request, screenshot, hardware.

## STRUCTURE
```
src/tools/
├── traits.rs       # Tool trait definition
├── mod.rs         # Factory + register_tool! macro
├── shell.rs       # Shell execution
├── file.rs        # File read/write
├── memory.rs      # Memory operations
├── browser.rs     # Browser automation
├── http.rs        # HTTP requests
├── screenshot.rs  # Screenshot capture
├── cron.rs        # Scheduled tasks
└── *.rs           # 40+ tool implementations
```

## WHERE TO LOOK
| Task | Location |
|------|----------|
| Add new tool | Implement `Tool` trait → register in `mod.rs` |
| Tool schema | `traits.rs` — `parameter_schema()` method |
| Tool execution | `execute()` method in trait |
| Mock tools | Use `EchoTool`, `CountingTool`, `FailingTool` for tests |

## CONVENTIONS
- Tool impl: `struct FooTool; impl Tool for FooTool { ... }`
- Parameter validation: strict schema via `parameter_schema()`
- Return: `ToolResult` — never panic in runtime path
- Naming: `<Name>Tool` pattern (e.g., `ShellTool`, `FileTool`)

## ANTI-PATTERNS
- **NEVER** use `?` operator without proper error context in `execute()`
- **NEVER** log raw credentials or secrets
- **NEVER** skip input sanitization

## RISK: HIGH
Security-critical module. Changes require:
- Failure-mode validation
- No silent permission broadening
