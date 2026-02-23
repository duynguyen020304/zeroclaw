# AGENTS.md — src/gateway

**Parent**: `./AGENTS.md` (read first)

## OVERVIEW
Webhook server — HTTP endpoints, WebSocket, SSE, static files.

## STRUCTURE
```
src/gateway/
├── mod.rs           # Server setup + routing
├── api.rs           # REST endpoints
├── ws.rs            # WebSocket handling
├── sse.rs           # Server-Sent Events
└── static_files.rs  # Embedded web UI
```

## WHERE TO LOOK
| Task | Location |
|------|----------|
| Server setup | `mod.rs` — `Gateway::new()`, `run()` |
| Webhook endpoint | `api.rs` — `/webhook` POST |
| Pairing flow | `api.rs` — `/pair` POST |
| Health check | `api.rs` — `/health` GET |

## CONVENTIONS
- Bind: `127.0.0.1` by default (security-hardened)
- Auth: Bearer token via `Authorization` header
- Pairing: 6-digit OTP exchange
- Never expose without tunnel

## ANTI-PATTERNS
- **NEVER** bind to `0.0.0.0` without explicit `allow_public_bind = true`
- **NEVER** log raw bearer tokens
- **NEVER** skip pairing verification
- **NEVER** accept webhook requests without valid Authorization

## RISK: HIGH
Internet-adjacent module. Changes REQUIRE:
- Security review
- Failure-mode validation
- No silent permission broadening
