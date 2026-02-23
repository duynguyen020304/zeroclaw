# AGENTS.md — src/runtime

**Parent**: `./AGENTS.md` (read first)

## OVERVIEW
Runtime adapters — native execution, Docker sandbox.

## STRUCTURE
```
src/runtime/
├── traits.rs       # RuntimeAdapter trait
├── mod.rs         # Factory + registration
├── native.rs      # Native (direct) execution
├── docker.rs      # Docker sandboxed execution
└── wasm.rs        # WASM runtime (planned)
```

## WHERE TO LOOK
| Task | Location |
|------|----------|
| Add runtime | Implement `RuntimeAdapter` trait → register in `mod.rs` |
| Native exec | `native.rs` — direct command execution |
| Docker sandbox | `docker.rs` — container isolation |
| Trait methods | `traits.rs` — `execute()`, `validate()` |

## CONVENTIONS
- Runtime impl: `struct FooRuntime; impl RuntimeAdapter for FooRuntime { ... }`
- Execute: `async fn execute(&self, cmd: &Command) -> Result<Output>`
- Validate: `fn validate(&self, policy: &SecurityPolicy) -> Result<()>`

## ANTI-PATTERNS
- **NEVER** execute without security policy validation
- **NEVER** skip resource limits (memory, CPU)
- **NEVER** use shell=True in command execution
- **NEVER** allow privilege escalation

## RISK: HIGH
Execution environment. Changes REQUIRE:
- Sandbox escape testing
- Resource limit validation
- Security review
