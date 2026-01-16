# Discovery: What is the thread system and how does it persist conversations?

> Category: Follow-ups (Data Flow)
> Discovered: 2026-01-16
> Confidence: High

## Answer

The thread system is Loom's conversation persistence layer, implementing a **dual-storage architecture** with local file-based storage on the client and SQLite database storage on the server. A `Thread` represents a complete AI coding session including conversation messages, agent state, git context, and metadata. The system uses **background synchronization** to keep local and server copies in sync, with a pending queue for offline resilience.

Locally, threads are stored as JSON files in `~/.local/share/loom/threads/{thread-id}.json`. On the server, threads are persisted in SQLite with denormalized columns for efficient querying and an FTS5 virtual table for full-text search across message content, titles, tags, git branches, and commit SHAs.

The sync mechanism is **eventually consistent**: saves happen locally first (immediate), then sync to server in a background task. Failed syncs are queued in a `PendingSyncStore` for retry. Private threads (`is_private: true`) never sync to the server.

## Evidence

### Core Types (`loom-common-thread/src/model.rs`)

```rust
pub struct Thread {
    pub id: ThreadId,                    // "T-{uuid7}" format
    pub version: u64,                    // Optimistic concurrency
    pub created_at: String,              // RFC3339 timestamps
    pub updated_at: String,
    pub last_activity_at: String,
    
    // Git context
    pub git_branch: Option<String>,
    pub git_remote_url: Option<String>,
    pub git_commits: Vec<String>,        // All commits observed
    
    // Core data
    pub conversation: ConversationSnapshot,  // Vec<MessageSnapshot>
    pub agent_state: AgentStateSnapshot,
    pub metadata: ThreadMetadata,        // title, tags, is_pinned
    
    // Access control
    pub visibility: ThreadVisibility,    // Organization, Private, Public
    pub is_private: bool,                // Never syncs if true
}
```

### Local Storage (`loom-common-thread/src/store.rs`)

```rust
pub struct LocalThreadStore {
    threads_dir: PathBuf,  // ~/.local/share/loom/threads/
}

// Threads stored as: {threads_dir}/{thread_id}.json
// Atomic writes via tmp file + rename
```

### Server Storage (`loom-server-db/src/thread.rs`)

```rust
pub struct ThreadRepository {
    pool: SqlitePool,
}

// Key operations:
// - upsert() with optimistic concurrency (version checking)
// - search() with FTS5 or commit SHA prefix matching
// - Soft-delete via deleted_at timestamp
```

### Sync Architecture (`loom-common-thread/src/sync.rs`)

```rust
pub struct SyncingThreadStore {
    local: LocalThreadStore,
    sync_client: Option<ThreadSyncClient>,
    pending_store: Option<Arc<Mutex<PendingSyncStore>>>,
}

// save() -> local save + background sync spawn
// save_and_sync() -> local save + blocking sync (for CLI commands)
```

**Key Files:**
- `crates/loom-common-thread/src/model.rs` — Thread, ThreadSummary, MessageSnapshot types
- `crates/loom-common-thread/src/store.rs` — LocalThreadStore (file-based)
- `crates/loom-common-thread/src/sync.rs` — SyncingThreadStore, ThreadSyncClient
- `crates/loom-server-db/src/thread.rs` — ThreadRepository (SQLite)
- `crates/loom-server/src/routes/threads.rs` — HTTP API handlers
- `crates/loom-server/migrations/001_create_threads.sql` — Base schema
- `crates/loom-server/migrations/005_thread_fts.sql` — FTS5 search

### Database Schema

```sql
-- Main table with denormalized columns for querying
CREATE TABLE threads (
    id TEXT PRIMARY KEY,           -- "T-{uuid7}"
    version INTEGER DEFAULT 1,     -- Optimistic concurrency
    conversation JSON NOT NULL,    -- Messages array
    agent_state JSON NOT NULL,     -- Current state machine state
    full_json JSON NOT NULL,       -- Complete thread for schema evolution
    -- Denormalized for queries:
    title TEXT, tags TEXT, workspace_root TEXT,
    git_branch TEXT, git_remote_url TEXT,
    deleted_at TEXT                -- Soft delete
);

-- FTS5 for full-text search
CREATE VIRTUAL TABLE thread_fts USING fts5(
    thread_id UNINDEXED,
    title, body, git_branch, git_remote_url, git_commits, tags
);
-- Triggers auto-populate FTS on INSERT/UPDATE/DELETE
```

### HTTP API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| PUT | `/api/threads/{id}` | Upsert with If-Match for concurrency |
| GET | `/api/threads/{id}` | Get single thread |
| GET | `/api/threads` | List with workspace filter, pagination |
| GET | `/api/threads/search` | FTS5 search or commit SHA prefix |
| DELETE | `/api/threads/{id}` | Soft delete |
| POST | `/api/threads/{id}/visibility` | Update visibility |

## Implications

- **Offline-first**: Local saves are instant; sync failures don't block the user
- **Version conflicts**: Use `If-Match` header with version number for safe concurrent updates
- **Search is powerful**: FTS5 indexes message content, git metadata, tags - can find threads by commit SHA prefix
- **Private threads**: Set `is_private: true` to keep threads local-only (never syncs)
- **Schema evolution**: `full_json` column stores complete thread, allowing backward-compatible changes
- **Soft deletes**: Threads are never hard-deleted; `deleted_at` timestamp filters them out

## Follow-up Questions

- [ ] How does the pending sync queue handle conflicts when the same thread is modified offline and online?
- [ ] How does the thread visibility system integrate with organization/team-level access control?

## Related Discoveries

- [[013-llm-proxy-pattern]] — Threads integrate with proxy for conversation persistence
- [[002-state-machine-orchestration]] — AgentState stored in threads
