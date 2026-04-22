---
name: matrix-expert
description: |
  Expert guidance on Matrix protocol, matrix-rust-sdk, E2EE, MAS OAuth, sliding sync, and
  bot/client architecture. Use when: working on Matrix client/bot code, debugging token refresh
  races, implementing E2EE, designing multi-client architecture, dealing with large accounts
  (3000+ rooms), or any question involving matrix-sdk, MAS, sliding sync, or Matrix auth.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - WebFetch
---

# Matrix Protocol & matrix-rust-sdk Expert

Deep knowledge of the Matrix ecosystem: protocol, matrix-rust-sdk, MAS OAuth, E2EE, sliding
sync, and the architectural patterns that make or break real-world apps.

> **Currency**: knowledge current as of 2026-04. `matrix-rust-sdk` API evolves quickly —
> verify specific symbol names against the current docs for newer SDK versions.

## When to use

- Building or debugging a Matrix client / bot / bridge
- Token refresh failing with `invalid_grant` or `M_UNKNOWN_TOKEN`
- Initial sync hanging or timing out on large accounts
- E2EE messages showing "UTD" (Unable To Decrypt)
- Verification, cross-signing, key backup issues
- Designing multi-process architectures (bot + TUI, multiple workers)
- Understanding MSC4186 (simplified sliding sync) vs legacy `/sync`
- Recovery key / secret storage questions

## Core Concepts Cheat Sheet

### Matrix Authentication Server (MAS) vs legacy auth

- **MAS = OAuth 2.0 + OIDC**. It replaces `/login` with the device code / authorization code
  flow. Access tokens are short-lived (typically **5 minutes**); refresh tokens are **single-use**.
- Detect MAS via `.well-known/matrix/client` → `org.matrix.msc2965.authentication.issuer` →
  fetch `{issuer}/.well-known/openid-configuration` for endpoints.
- Token format: `mat_...` (access) and `mar_...` (refresh) on MAS. Legacy Synapse issues
  opaque `syt_...` tokens without refresh.
- MAS also supports the legacy `/_matrix/client/v3/refresh` endpoint — both work, the SDK uses
  the v3 endpoint for matrix-auth sessions.

### The single-use refresh token trap

**Every time you refresh, the old refresh token is INVALIDATED.** If two processes (or two
Client instances in one process!) both try to refresh using the same on-disk refresh token,
one wins and the other gets `invalid_grant` forever. This is the #1 source of bugs in
multi-client Matrix apps.

Rules:
1. **One refresh in flight at a time.** Use a file lock for multi-process, a Mutex for single.
2. **Always reload tokens from disk before refreshing** — someone else may have already done it.
3. **Atomic writes** (temp file + rename) so a crash can't leave partial tokens.
4. **Never cache the refresh token in memory long-term** — re-read from disk on every use.

### `matrix-rust-sdk` auth model

- `Client::builder().handle_refresh_tokens()` → SDK auto-refreshes on 401. Without it, you
  must call `client.refresh_access_token()` yourself.
- `restore_session()` takes a `MatrixSession` that wraps `SessionTokens { access_token,
  refresh_token }` plus identity fields (`user_id`, `device_id`). `refresh_token` is
  `Option<String>` — if `None`, auto-refresh is disabled for that session even if the
  builder enabled it.
- `Client::set_session_callbacks(reload_cb, save_cb)` — `save_cb` fires after refresh so you
  can persist new tokens to disk. `reload_cb` is **only used for OAuth cross-process mode**,
  NOT for matrix-auth refresh. So you can't rely on it to re-read disk before refresh in the
  matrix-auth path.
- `Client` is internally `Arc<ClientInner>` — cloning is cheap; pass it around freely.
- Two separate `Client` instances in one process = two separate `auth_ctx` = two refresh
  locks = race. **Always share ONE Client per process.**

### MSC4186: Simplified Sliding Sync

- Traditional `/sync` with a large account (3000+ rooms) can take minutes or time out. A full
  initial sync downloads all room state, all timelines, all keys.
- Sliding sync fetches rooms in **windows**. You define lists with `SlidingSyncMode::Growing {
  batch_size, max }` and the server streams room info in batches.
- Use `required_state` to tell the server which state events you need (`m.room.name`,
  `m.room.create`, `m.room.canonical_alias`) — otherwise `Room::name()` returns None and
  `display_name()` falls back to member names.
- `no_timeline_limit()` = don't fetch any timeline events (fast, good for room list only).
- Server must advertise support: look for `"org.matrix.simplified_msc3575"` in
  `/_matrix/client/versions` → `unstable_features`. MAS-fronted servers (Synapse MAS,
  Conduit) support it.
- Sliding sync is the right choice for **initial state** / room list on large accounts. For
  live sync, either sliding sync or traditional works — traditional is simpler.

### `RoomListService` — the high-level API

`matrix_sdk_ui::room_list_service::RoomListService` is the recommended high-level API built
on top of sliding sync. It handles list lifecycle, state hydration, filters, and
room-subscription windowing for you. Use it for anything UI-shaped: room lists, dashboards,
chat clients. Drop to raw `SlidingSync` (skeleton at the bottom of this doc) only when you
need custom list windows, bespoke `required_state`, or non-UI patterns. Paired with
`RoomListService`, the `EncryptionSyncService` (also in `matrix_sdk_ui`) runs a dedicated
sync loop for to-device messages, key requests, and backup — see the E2EE section below.

### E2EE: what breaks, how to fix

- **New device → new device keys**. Other devices won't share session keys until yours is
  verified (SAS) or trusts cross-signing from your recovery key.
- **Recovery key** (Element calls it "Security Key") unlocks `m.cross_signing.*` and
  `m.megolm_backup.v1` secrets from server-side secret storage. Apply it with
  `client.encryption().recovery().recover(&key)`.
- **UTD (Unable To Decrypt)**: the device doesn't have the megolm session for that message.
  Solutions, in order:
  1. Enable key backup: `EncryptionSettings { auto_enable_backups: true, ... }`
  2. Enter recovery key → pulls signing + backup keys from server
  3. Request keys from other devices (manual verification)
- **`backup_download_strategy: BackupDownloadStrategy::AfterDecryptionFailure`** = lazy
  backup fetch (downloads a key only when you try to decrypt a message and fail).
  Alternative is `BackupDownloadStrategy::OneShot` which downloads everything at startup —
  slow on large accounts.
- **`auto_enable_cross_signing: true`** + `auto_enable_backups: true` in `EncryptionSettings`
  makes new logins automatically bootstrap cross-signing and turn on key backup.
- **`EncryptionSyncService`** (`matrix_sdk_ui`) runs a dedicated sync loop for to-device
  messages, key requests, and backup — independent of your main `client.sync()`. Bots that
  only run `client.sync()` sometimes miss room keys; running the encryption sync service
  alongside fixes it. Element uses it by default via the RoomListService integration.
- **Crypto store = SQLite file** (`matrix-sdk-state.sqlite3` etc). One writer at a time.
  Sharing between processes will corrupt it. Separate processes need separate stores.
- **Separate state/crypto store from event cache.** Modern SDKs expose
  `sqlite_store_with_cache_path(state_path, cache_path, passphrase)` so the volatile event
  cache can live on fast or tmpfs storage while durable state + crypto stay on persistent
  storage. Recommended for large accounts.
- **Crypto store corruption** on SDK upgrade: error looks like
  `"Failed to deserialize RoomInfo"` or `"could not deserialize ... sqlite"`. Only fix is
  wipe + re-sync. **Always back up the store before wiping** — it contains your private keys.

### Room version 12 quirks

- Room IDs in v12 drop the `:server.tld` suffix. A v12 room ID looks like `!abc...xyz`
  (no colon, no server). Don't assume `:` is always there when parsing.
- The create event's event ID becomes the room ID. This makes room IDs globally unique
  without federation lookup.

## Common Bug Patterns → Fixes

### "token refresh failed: invalid_grant"

**Diagnosis**: Another process/Client already consumed the refresh token.

**Fix order**:
1. Is there really more than one Client in this process? Consolidate to one.
2. Multiple processes? Add file lock + "reload from disk before refresh" pattern.
3. Tokens genuinely revoked (user re-logged in on another device)? Only fix is re-auth.

### Initial sync hangs on large account

**Diagnosis**: `sync_once()` on traditional `/sync` with no timeout OR 30s timeout is too
short to return full state for 3000+ rooms.

**Fix**: Replace `sync_once()` with sliding sync. Batch of 100 rooms, loop until
`summary.rooms.is_empty()` or you hit `maximum_number_of_rooms`.

### Bot doesn't respond to commands

**Diagnosis order**:
1. Is `client.sync()` actually running? Check logs for sync activity.
2. Is the handler filtering by sender? Many bots reject non-owner messages.
3. Is the room E2EE and the bot missing keys? Look for UTD logs.
4. Is the bot filtering by room_id and the actual room_id doesn't match (e.g. v12 ID format)?

### "whoami failed HTTP 401"

**Diagnosis**: Access token expired. If it happens at startup, SDK's auto-refresh wasn't
enabled, or the refresh token was also dead.

**Fix**: Build client with `.handle_refresh_tokens()` AND pass `refresh_token` in
`SessionTokens` AND set `save_session_callback` to persist new tokens to disk.

### E2EE works in Element but not in your bot

**Diagnosis**: Bot's device was never verified. Element treats it as an untrusted device and
won't share room keys with it.

**Fix**: After login, call `client.encryption().recovery().recover(&recovery_key).await?`.
This downloads cross-signing keys from server secret storage. Now the bot's device can be
cross-signed, and Element will trust it.

## Architectural Rules (learned the hard way)

1. **One Matrix `Client` per process, and prefer one process.** `Client` is
   `Arc<ClientInner>` internally — clone is free, pass it around. The TUI, the bot, the
   background indexer all share ONE client. Separate processes mean separate crypto
   stores, separate refresh locks, separate bugs; spawn tasks on the shared client instead.

2. **Token persistence: disk is truth, memory is cache.** Before any refresh, reload from
   disk. After any refresh, atomic write to disk.

3. **Don't trust `sync_once()` on large accounts.** Always use sliding sync (or
   `RoomListService`) for the initial population.

4. **Provide a clear re-auth path.** When a refresh token is truly revoked, the user must
   be able to run OAuth again without wiping the crypto store. The crypto store survives
   token changes — only the session is dead.

5. **PID lockfile** to prevent two instances racing on the same config dir. `fs4::FileExt`
   advisory lock is sufficient. (`fs4` is the maintained fork of the deprecated `fs2` —
   crates.io search still surfaces `fs2` first; don't pick that one.)

6. **Device verification is a one-time setup**. On first login, auto-bootstrap cross-signing
   and auto-enable key backup. Tell the user "enter your recovery key" once, then never
   again.

7. **Crypto store corruption is not the user's fault.** Detect it by error string
   (`"deserialize"` + `"RoomInfo"`/`"sqlite"`), back it up (don't delete), tell the user
   to re-auth.

## Quick reference: builder settings for a robust client

```rust
Client::builder()
    .homeserver_url(url)
    .sqlite_store(path, None)
    .handle_refresh_tokens()          // auto-refresh on 401
    .with_encryption_settings(EncryptionSettings {
        auto_enable_cross_signing: true,
        backup_download_strategy: BackupDownloadStrategy::AfterDecryptionFailure,
        auto_enable_backups: true,
    })
    .build()
    .await?;

// After restore_session — clone `path` / `base_config` into each closure separately,
// otherwise the first closure moves them and the second won't compile.
client.set_session_callbacks(
    Box::new({
        let path = path.clone();
        move |_client| {
            // Reload session from disk.
            // Note: NOT called for matrix-auth refresh (only OAuth cross-process).
            // Keep it correct anyway.
            let cfg = Config::load(&path)?;
            Ok(SessionTokens {
                access_token: cfg.access_token,
                refresh_token: cfg.refresh_token, // Option<String>
            })
        }
    }),
    Box::new({
        let path = path.clone();
        let base_config = base_config.clone();
        move |client| {
            // Persist refreshed tokens to disk. CALLED BY SDK after refresh.
            let tokens = client.session_tokens().ok_or("no tokens")?;
            let mut cfg = base_config.clone();
            cfg.access_token = tokens.access_token;
            cfg.refresh_token = tokens.refresh_token;
            cfg.save(&path)?;
            Ok(())
        }
    }),
)?;
```

## Sliding sync skeleton

```rust
use matrix_sdk::sliding_sync::{SlidingSyncList, SlidingSyncMode, Version};
use matrix_sdk::ruma::events::StateEventType;

let list = SlidingSyncList::builder("rooms")
    .sync_mode(SlidingSyncMode::Growing {
        batch_size: 100,
        maximum_number_of_rooms_to_fetch: None,
    })
    .no_timeline_limit()
    .required_state(vec![
        (StateEventType::RoomName, "".to_owned()),
        (StateEventType::RoomCreate, "".to_owned()),
        (StateEventType::RoomCanonicalAlias, "".to_owned()),
    ]);

let ss = client.sliding_sync("init")?.version(Version::Native).add_list(list).build().await?;
let stream = ss.sync();
futures::pin_mut!(stream);

loop {
    match tokio::time::timeout(Duration::from_secs(60), stream.next()).await {
        Ok(Some(Ok(summary))) => {
            // Check progress
            if let Some(max) = ss.on_list("init", |l| async move { l.maximum_number_of_rooms() }).await.flatten() {
                if client.joined_rooms().len() as u32 >= max { break; }
            }
            if summary.rooms.is_empty() { break; }
        }
        Ok(Some(Err(e))) => bail!("sliding sync error: {e}"),
        Ok(None) => break,
        Err(_) => bail!("sliding sync timeout"),
    }
}
```

## Other traps worth knowing

- **Push rules for bot @-mentions.** Bots that only want to respond to direct mentions
  should filter on `m.push_rules` (override rules `.m.rule.contains_user_name` /
  `.m.rule.contains_display_name`) rather than substring-matching the body. Intentional
  mentions (`m.mentions` in event content, MSC3952) are the modern path — prefer those when
  the server supports them.
- **Never add `ruma` as a direct dep.** Always use `matrix_sdk::ruma` re-exports. Pulling
  `ruma` directly pins a different version than the SDK expects and produces
  "expected `ruma::X`, found `ruma::X`" type errors that look identical but are two
  different types.
- **OAuth redirect URIs for local apps.** MAS device-code flow works everywhere. The
  auth-code flow needs a redirect URI the browser can reach — for CLI / bot apps, bind a
  local HTTP listener on `http://127.0.0.1:<port>/callback` and register that exact URI
  with MAS. `http://localhost` sometimes fails where `127.0.0.1` succeeds, depending on
  MAS client config.

## Debugging checklist for new Matrix issues

1. What's the SDK version? (`matrix-rust-sdk` moves fast — pin it in any bug report.)
2. What's the server? (Synapse, Dendrite, Conduit, MAS-fronted?)
3. What's the auth? (legacy /login, MAS device-code, MAS auth-code)
4. Access token TTL — check `expires_in` in token response
5. Refresh token behavior — single-use? check token endpoint response
6. How many `Client` instances? (grep for `Client::builder`, `restore_session`)
7. How many processes touch the same config/crypto store?
8. Is the crypto store writable and uncorrupted?
9. Are `required_state` hints set for sliding sync?
10. Is E2EE bootstrapped (`recover()` called)? Is `EncryptionSyncService` running?
11. Does the server advertise `org.matrix.simplified_msc3575` in `unstable_features`?

## References

- matrix-rust-sdk source: https://github.com/matrix-org/matrix-rust-sdk
- MSC4186 — Simplified Sliding Sync: https://github.com/matrix-org/matrix-spec-proposals/pull/4186
- MSC3575 — Sliding Sync (original): https://github.com/matrix-org/matrix-spec-proposals/pull/3575
- MSC2965 — OIDC discovery: https://github.com/matrix-org/matrix-spec-proposals/pull/2965
- MSC3952 — Intentional mentions: https://github.com/matrix-org/matrix-spec-proposals/pull/3952
- MSC4291 — Room v12 (create event as room ID): https://github.com/matrix-org/matrix-spec-proposals/pull/4291
- MAS: https://github.com/element-hq/matrix-authentication-service
- Matrix E2EE spec: https://spec.matrix.org/latest/client-server-api/#end-to-end-encryption
