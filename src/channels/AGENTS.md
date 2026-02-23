# AGENTS.md — src/channels

**Parent**: `./AGENTS.md` (read first)

## OVERVIEW
Messaging channel integrations — Telegram, Discord, Slack, WhatsApp, Matrix, Nostr, Signal.

## STRUCTURE
```
src/channels/
├── traits.rs       # Channel trait definition
├── mod.rs         # Factory + register_channel! macro
├── telegram.rs    # Telegram bot
├── discord.rs     # Discord bot
├── slack.rs       # Slack bot
├── whatsapp.rs    # WhatsApp (Web + Business API)
├── matrix.rs      # Matrix (with E2EE)
├── nostr.rs       # Nostr (NIP-04, NIP-17)
└── *.rs           # 20+ channel implementations
```

## WHERE TO LOOK
| Task | Location |
|------|----------|
| Add new channel | Implement `Channel` trait → register in `mod.rs` |
| Channel trait | `traits.rs` — `send()`, `listen()`, `health_check()` |
| Allowlist logic | `mod.rs` — deny-by-default policy |
| Media handling | Telegram attachments via `[IMAGE:]`, `[DOCUMENT:]` markers |

## CONVENTIONS
- Channel impl: `struct FooChannel; impl Channel for FooChannel { ... }`
- Methods: `send()`, `listen()`, `health_check()`, `typing()`
- Auth: `validate_sender()` for allowlist checking
- Error: Return `ChannelError` with context

## ANTI-PATTERNS
- **NEVER** expose gateway without tunnel when public
- **NEVER** log raw tokens/API keys
- **NEVER** skip allowlist validation

## RISK: MEDIUM
Internet-adjacent module. Changes require:
- Auth/allowlist behavior tests
- Health-check validation
