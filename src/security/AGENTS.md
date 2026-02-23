# AGENTS.md — src/security

**Parent**: `./AGENTS.md` (read first)

## OVERVIEW
Security subsystem — pairing, sandboxing, OTP, E-stop, policy enforcement.

## STRUCTURE
```
src/security/
├── pairing.rs     # Device pairing + OTP
├── sandbox.rs     # Landlock/firejail/bubblewrap
├── policy.rs      # SecurityPolicy enforcement
├── estop.rs       # Emergency stop levels
├── secrets.rs     # Encrypted secret store
└── *.rs           # Security modules
```

## WHERE TO LOOK
| Task | Location |
|------|----------|
| Pairing flow | `pairing.rs` — 6-digit OTP, bearer token |
| Sandbox setup | `sandbox.rs` — runtime isolation |
| Policy rules | `policy.rs` — allowlist, rate limits |
| E-stop levels | `estop.rs` — readonly/supervised/full |

## CONVENTIONS
- NEVER silently broaden permissions
- Deny-by-default for all access boundaries
- Fail-fast for unsupported/unsafe states
- No secrets in logs

## ANTI-PATTERNS
- **NEVER** weaken security policy without explicit justification
- **NEVER** log raw tokens, API keys, or credentials
- **NEVER** skip allowlist validation
- **NEVER** fallback to insecure defaults

## RISK: HIGH
Security-critical module. Changes REQUIRE:
- Threat/risk analysis in PR
- Failure-mode validation tests
- Rollback strategy documented
- Never bypass failing checks
