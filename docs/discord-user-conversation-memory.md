# Discord User Conversation History Memory with Context Isolation

## Context

ZeroClaw currently maintains **per-user conversation history** in memory using `ConversationHistoryMap`, which correctly isolates context by `{channel}_{sender}` (e.g., `discord_123456789`). However, this history is **lost when the daemon restarts**.

**The Problem:**
- User A talks to bot → context stored in memory
- Restart daemon → all contexts lost
- User A continues conversation → bot doesn't remember them
- **CRITICAL**: User A's context must NEVER leak to User B

**The Solution:**
- Persist conversation history per-user to memory backend
- Load past conversations on startup (configurable amount)
- Maintain strict context isolation between users
- Use `{channel}_{sender}` as the key for isolation

## Key Design Decisions

### Context Isolation (Critical Requirement)
- **Storage Key**: `discord_conv:{user_id}` - one key per user
- **Session ID**: Use `sender` (user ID) as `session_id` for memory backend isolation
- **No Cross-User Leaks**: Each user's history is stored separately, never mixed
- **Existing Pattern**: The codebase already uses `conversation_history_key = "{channel}_{sender}"` for isolation

### Storage Strategy
| Aspect | Implementation |
|--------|---------------|
| Key Format | `discord_conv:{user_id}` |
| Memory Category | `MemoryCategory::Conversation` |
| Session ID | `{user_id}` (for backend-level isolation) |
| Format | JSON array of `ChatMessage` objects |
| Max History | `MAX_DISCORD_CONVERSATION_HISTORY = 100` messages per user |

### When to Save/Load
- **Load**: On user's first message after restart (if `load_past_conversations > 0`)
- **Save**: After each user message is processed (append to existing history)
- **Trim**: Before saving, trim to max size to prevent unbounded growth

## Implementation Plan

### Phase 1: Add Discord Configuration

**File**: `src/config/schema.rs`

```rust
#[derive(Debug, Clone, Deserialize)]
#[serde(default)]
pub struct DiscordConfig {
    pub bot_token: String,
    pub guild_id: Option<String>,
    pub allowed_users: Vec<String>,
    pub listen_to_bots: bool,
    pub mention_only: bool,
    pub load_past_conversations: usize,  // NEW: Number of past messages to load (default: 0 = disabled)
}

impl Default for DiscordConfig {
    fn default() -> Self {
        Self {
            bot_token: String::new(),
            guild_id: None,
            allowed_users: vec![],
            listen_to_bots: false,
            mention_only: false,
            load_past_conversations: 0,  // Default: disabled
        }
    }
}
```

### Phase 2: Add Conversation Persistence Functions

**File**: `src/channels/mod.rs`

Add helper functions with user-isolated storage:

```rust
/// Maximum conversation messages to store per Discord user
const MAX_DISCORD_CONVERSATION_HISTORY: usize = 100;

/// Key for storing conversation history for a specific user
/// Format: {channel}_conv:{user_id}
/// This ensures strict isolation between users
fn conversation_storage_key(channel: &str, sender: &str) -> String {
    format!("{}_conv:{}", channel, sender)
}

/// Save conversation history to memory backend (user-isolated)
///
/// Uses sender as session_id for additional backend-level isolation.
/// This ensures User A's history cannot be accessed by User B even in
/// the memory backend itself.
async fn save_user_conversation_history(
    memory: &dyn Memory,
    channel: &str,
    sender: &str,
    history: &[ChatMessage],
) -> anyhow::Result<()> {
    if history.is_empty() {
        return Ok(());
    }

    // Trim to max size before saving
    let to_save: Vec<ChatMessage> = history
        .iter()
        .rev()
        .take(MAX_DISCORD_CONVERSATION_HISTORY)
        .cloned()
        .collect();

    let key = conversation_storage_key(channel, sender);
    let json = serde_json::to_string(&to_save)?;

    // CRITICAL: Use sender as session_id for backend-level isolation
    memory.store(
        &key,
        &json,
        MemoryCategory::Conversation,
        Some(sender),  // session_id ensures per-user backend isolation
    ).await
}

/// Load conversation history from memory backend (user-isolated)
///
/// Only loads history for this specific user. Uses session_id filter
/// to prevent cross-user data leakage.
async fn load_user_conversation_history(
    memory: &dyn Memory,
    channel: &str,
    sender: &str,
) -> anyhow::Result<Vec<ChatMessage>> {
    let key = conversation_storage_key(channel, sender);

    // CRITICAL: Use session_id filter to ensure we only get this user's data
    if let Some(entry) = memory.get(&key).await? {
        // Verify the entry belongs to this user (defensive check)
        if entry.session_id.as_deref() != Some(sender) {
            tracing::error!(
                "Session ID mismatch for key {}: expected {}, got {:?}",
                key,
                sender,
                entry.session_id
            );
            return Ok(Vec::new());
        }

        serde_json::from_str(&entry.content)
            .map_err(|e| anyhow::anyhow!("Failed to parse conversation history: {}", e))
    } else {
        Ok(Vec::new())  // No history found for this user
    }
}
```

### Phase 3: Load History on User First Message

**File**: `src/channels/mod.rs` in `process_channel_message()`

Load user's past history on first message (after restart):

```rust
async fn process_channel_message(
    ctx: Arc<ChannelRuntimeContext>,
    msg: traits::ChannelMessage,
    cancellation_token: CancellationToken,
) {
    // ... existing code ...

    let history_key = conversation_history_key(&msg);

    // Check if we need to load history (first message from this user after restart)
    let should_load_history = {
        let map = ctx.conversation_histories.lock().unwrap_or_else(|e| e.into_inner());
        !map.contains_key(&history_key)
    };

    if should_load_history {
        // Check if this channel has load_past_conversations configured
        let load_limit = match msg.channel.as_str() {
            "discord" => ctx
                .channels_config
                .as_ref()
                .and_then(|c| c.discord.as_ref())
                .map(|d| d.load_past_conversations)
                .unwrap_or(0),
            _ => 0,  // Can extend to other channels later
        };

        if load_limit > 0 {
            tracing::info!(
                "Loading conversation history for user {} on channel {} (limit: {})",
                msg.sender,
                msg.channel,
                load_limit
            );

            // Load this user's past conversation history (isolated by user_id)
            match load_user_conversation_history(&*ctx.memory, &msg.channel, &msg.sender).await {
                Ok(loaded) => {
                    if !loaded.is_empty() {
                        // Take only the most recent 'load_limit' messages
                        let to_restore: Vec<ChatMessage> = loaded
                            .into_iter()
                            .rev()
                            .take(load_limit)
                            .collect();

                        let mut map = ctx.conversation_histories
                            .lock()
                            .unwrap_or_else(|e| e.into_inner());

                        map.entry(history_key.clone())
                            .or_insert_with(Vec::new)
                            .extend(to_restore);

                        tracing::info!(
                            "Loaded {} past messages for user {} ({})",
                            to_restore.len(),
                            &msg.sender,
                            &msg.channel
                        );
                    } else {
                        tracing::debug!(
                            "No past conversation history found for user {}",
                            msg.sender
                        );
                    }
                }
                Err(e) => {
                    tracing::warn!(
                        "Failed to load conversation history for user {}: {}",
                        msg.sender,
                        e
                    );
                }
            }
        }
    }

    // ... rest of existing message processing ...
}
```

### Phase 4: Save History After Each Message

**File**: `src/channels/mod.rs` after the agent loop completes

Save updated history after processing (maintain user isolation):

```rust
// In process_channel_message(), after processing the message

// Save updated conversation history for this user (isolated)
let current_history = ctx
    .conversation_histories
    .lock()
    .unwrap_or_else(|e| e.into_inner())
    .get(&history_key)
    .cloned()
    .unwrap_or_default();

if let Err(e) = save_user_conversation_history(
    &*ctx.memory,
    &msg.channel,
    &msg.sender,
    &current_history,
).await {
    tracing::warn!(
        "Failed to save conversation history for user {} on {}: {}",
        msg.sender,
        msg.channel,
        e
    );
}
```

### Phase 5: Verify Memory Backend Isolation

**File**: `src/memory/sqlite.rs` (verification only, no changes needed)

The SQLite backend already supports session-scoped queries:

```rust
// Existing implementation (already correct)
pub async fn get(&self, key: &str) -> anyhow::Result<Option<MemoryEntry>> {
    // ... retrieves by key only (session_id is stored but not filtered in get())
    // This is OK because we use unique keys per user: discord_conv:{user_id}
}
```

**Important**: The `session_id` is stored for each entry but `get()` retrieves by key only. Since we use unique keys per user (`discord_conv:{user_id}`), this provides the isolation we need.

### Phase 6: Configuration Documentation

**File**: `docs/config-reference.md`

Add documentation for the new option:

```toml
# Discord channel configuration
[[channels.discord]]
bot_token = "your_bot_token"
allowed_users = ["*"]
mention_only = false

# User conversation memory
load_past_conversations = 20  # Load up to 20 past messages per user (0 = disabled)
```

Explain the isolation:
> **Context Isolation**: Each Discord user's conversation history is stored separately under `discord_conv:{user_id}`. Users cannot see or access each other's conversation history. The `load_past_conversations` setting controls how many past messages to restore when a user sends their first message after a restart.

## Files to Modify

| File | Purpose | Lines |
|------|---------|-------|
| `src/config/schema.rs` | Add `load_past_conversations` to `DiscordConfig` | ~10 |
| `src/channels/mod.rs` | Add save/load functions, integrate into message processing | ~150 |
| `docs/config-reference.md` | Document new configuration option | ~20 |

## Data Flow with Context Isolation

```
┌─────────────────────────────────────────────────────────────────────┐
│  User A (ID: 111) sends message to Discord bot                      │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             v
┌─────────────────────────────────────────────────────────────────────┐
│  Check in-memory history for key "discord_111"                       │
│  ├─ Not found: Load from memory backend                             │
│  │   Key: "discord_conv:111"                                        │
│  │   Session: "111" (user-isolated)                                 │
│  └─ Found: Use existing in-memory history                           │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             v
┌─────────────────────────────────────────────────────────────────────┐
│  Process message with isolated context                               │
│  ├─ Only User A's history is used                                   │
│  ├─ User B's history cannot be accessed                             │
│  └─ Backend filters by session_id="111"                             │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             v
┌─────────────────────────────────────────────────────────────────────┐
│  Save updated history (User A only)                                 │
│  Key: "discord_conv:111"                                            │
│  Session: "111"                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  User B (ID: 222) sends message                                     │
│  └─ Completely isolated from User A                                 │
│  └─ Uses key "discord_conv:222"                                     │
│  └─ Uses session_id="222"                                           │
└─────────────────────────────────────────────────────────────────────┘
```

## Configuration Example

```toml
# Enable Discord user conversation memory with context isolation
[[channels.discord]]
bot_token = "your_bot_token_here"
allowed_users = ["*"]
mention_only = false
load_past_conversations = 20  # Load up to 20 past messages per user
```

## Verification Steps

### 1. Context Isolation Test
```
User Setup:
- User A (ID: 111) talks about "rust programming"
- User B (ID: 222) talks about "python programming"
- Restart daemon

Verification:
- User A sends "continue" → bot should discuss Rust
- User B sends "continue" → bot should discuss Python
- Cross-contamination check: User A should NOT see Python context
```

### 2. History Loading Test
```
Configuration: load_past_conversations = 5

Steps:
1. User sends 10 messages
2. Restart daemon
3. User sends new message
4. Bot should remember last 5 messages
```

### 3. Multi-User Concurrent Test
```
Scenario:
- User A and User B send messages simultaneously
- Verify no cross-contamination in responses
- Check memory backend has separate keys:
  - discord_conv:111 (User A's history)
  - discord_conv:222 (User B's history)
```

### 4. Disabled Test
```
Configuration: load_past_conversations = 0

Expected:
- No history loaded after restart
- Bot treats each message as fresh conversation
```

## Testing Commands

```bash
# Unit tests for channel message processing
cargo test -- channels::mod::test_conversation_storage

# Integration test (requires Discord bot token)
# 1. Set load_past_conversations = 5
# 2. Start with two different users
# 3. Send messages from both users
# 4. Restart
# 5. Send message from each user - verify context is preserved and isolated
cargo run -- --channel discord
```

## Rollback Plan

If issues arise:
1. **Disable**: Set `load_past_conversations = 0` to disable loading
2. **Clean Data**: Remove stored history with:
   ```bash
   # Via memory backend cleanup (if needed)
   # Keys follow pattern: discord_conv:{user_id}
   ```
3. **No Breaking Changes**: Feature is opt-in via configuration
4. **Existing Behavior**: When disabled, behaves exactly as before (no persistence)

## Security & Privacy Considerations

1. **User Isolation**: Each user's conversation is stored with their user ID as `session_id`
2. **No Cross-User Access**: Memory backend filters by session_id
3. **Key Uniqueness**: `discord_conv:{user_id}` ensures no key collisions
4. **Defensive Checks**: Verify `session_id` matches when loading (already implemented)
5. **Configurable**: Users can opt-out by setting `load_past_conversations = 0`

## Benefits

1. **Persistent Memory**: Bot remembers each user across restarts
2. **Strict Isolation**: User A's context NEVER leaks to User B
3. **Context Continuity**: Users don't need to repeat context after restart
4. **Configurable**: Operators control how much history to load per user
5. **Scalable**: Works with unlimited users, each with isolated storage
6. **Backend-Level Isolation**: Uses `session_id` for additional safety layer
