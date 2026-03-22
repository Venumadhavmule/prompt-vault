# SKILL 03-C — How to Manage Offline State and Local Persistence in Mobile
Category: Mobile Development
Applies to: React Native, Expo; MMKV, WatermelonDB, SQLite, React Query

## What this skill covers
Mobile apps run on intermittent networks. An app that fails completely when offline is broken. This skill covers how to detect network state, persist data locally for offline access, queue pending mutations for sync when connectivity returns, and handle the synchronization of server state with local state correctly. It also covers the right storage layers for different data types: secure storage for credentials, key-value for preferences, and a local database for relational data.

## When to activate this skill
- When building any mobile app that users will use in unreliable networks
- When implementing "remember me" or session persistence across app restarts
- When data created offline needs to sync when online
- When choosing between MMKV, AsyncStorage, and SQLite

## Core principles
1. **Choose storage by data type.** Credentials → `expo-secure-store`. Preferences and small values → MMKV. Structured relational data → WatermelonDB or SQLite.
2. **Never use AsyncStorage for sensitive data.** It is unencrypted and accessible on rooted/jailbroken devices.
3. **Network state determines behavior, not just display.** Offline means some actions are queued, not blocked.
4. **React Query handles server state caching.** Use React Query's `persistQueryClient` plugin to persist caches across app restarts — this replaces manual offline caching for most use cases.
5. **Optimistic updates require rollback.** When queuing mutations for offline sync, implement rollback for the cases where sync fails.

## Step-by-step guide
1. **Detect network state:** use `@react-native-community/netinfo` — `NetInfo.fetch()` or `NetInfo.addEventListener()`. Expose via a `useNetworkStatus()` composable.
2. **Choose storage layer:**
   - `expo-secure-store`: auth tokens, PII — encrypted, limited to 2KB per key
   - `react-native-mmkv`: preferences, feature flags, small cached values — 30x faster than AsyncStorage
   - `expo-sqlite` or WatermelonDB: large structured data, offline-first apps with complex queries
3. **Persist React Query cache across restarts:**
   - Install `@tanstack/query-async-storage-persister` and `@tanstack/react-query-persist-client`
   - Configure with MMKV as the storage backend
   - Set `gcTime` long enough to survive app restarts (e.g. 24 hours)
4. **Queue offline mutations:**
   - Use React Query's `networkMode: 'offlineFirst'` on mutations
   - Detect failed mutations and store them in MMKV as a pending queue
   - On reconnect (`NetInfo` event), replay the queue
5. **Sync conflict resolution:** define a merge strategy before implementing sync. Last-write-wins is simplest; timestamp-based or user-choice is needed for collaborative features.
6. **Show offline state clearly in UI:** a subtle banner indicating "offline — changes will sync when connected" is sufficient for most apps.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| `AsyncStorage` for auth tokens | `expo-secure-store` for any sensitive data |
| Allow no actions while offline | Queue mutations locally, sync on reconnect |
| Blank screen on network error | Show cached data with "last updated at [time]" indicator |
| Manual offline cache management with `AsyncStorage` | `persistQueryClient` plugin with MMKV — automated caching |
| Optimistic update with no rollback | Always implement rollback when the server mutation fails |

## Stack-specific notes
**Expo Managed Workflow:** `expo-secure-store`, `expo-sqlite` are available without ejecting. MMKV requires `expo-dev-client` or bare workflow.
**Bare Workflow:** Full access to all React Native packages. Use `react-native-mmkv` directly.
**Node.js / Python (API):** API should support `If-Modified-Since` or ETags for efficient cache validation. Support idempotent mutations (with idempotency keys) so mobile clients can safely replay failed mutations.

## Common mistakes
1. **Using AsyncStorage for tokens.** It is unencrypted. Tokens in AsyncStorage on a jailbroken device are accessible to other apps.
2. **No offline indicator.** Users who trigger actions while offline assume the app is broken. A small indicator eliminates this confusion.
3. **Merging stale offline data over fresh server data.** When coming online after offline mutations, check server state first before assuming local state is the latest.
4. **Storing too much in secure storage.** `expo-secure-store` has a 2KB per-key limit. Store only tokens and small secrets — not full user profiles.

## Checklist
- [ ] `expo-secure-store` used for all credentials and tokens
- [ ] MMKV used for preferences and cached small values
- [ ] Network status detected via `NetInfo`
- [ ] `persistQueryClient` configured for React Query with MMKV storage
- [ ] Offline mutations queued and replayed on reconnect
- [ ] Offline state shown in UI with appropriate indicator
- [ ] Rollback implemented for failed optimistic updates
- [ ] No sensitive data in AsyncStorage or MMKV (only in SecureStore)
- [ ] Data sync tested by toggling airplane mode mid-session
- [ ] Conflict resolution strategy defined for collaborative features
