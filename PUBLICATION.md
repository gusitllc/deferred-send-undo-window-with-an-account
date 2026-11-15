# Deferred-send undo window with an account-bound encrypted offline action queue that is never written over before it is read

**Defensive Publication** - published to establish prior art.
**Publisher:** Gus IT LLC.

## Abstract

A mobile client applies user actions on objects such as archive, delete or complete by hiding the object immediately but sending nothing until a cancellable undo period ends, so undo needs no server-side restore operation. At most one action is held at a time. If sending finds no connection, the action enters an encrypted, per-account offline queue and only then receives a random idempotency key. Every asynchronous operation captures an account mark and discards its result if the signed-in account has changed. A user-initiated sign-out runs ordered pre-sign-out steps under one time-bounded abort signal, then reports changes that could not be sent and destroys the encryption key before deleting stored records. A queue or index on disk is never overwritten until it has been read for the current account.

## Technical Field

Mobile and web client software, specifically offline-capable user actions, local encrypted caching, and account isolation on shared devices.

## Background

Mobile applications commonly offer an undo for destructive actions. One approach sends the action immediately and implements undo as a second server call that restores the object, which requires a restore operation for every action type and leaves a window where other devices see the intermediate state. Another approach deletes locally and syncs later, which can lose changes silently when the app is closed or the user signs out. Offline outboxes typically store pending requests in local storage and replay them on reconnect, but they rarely consider that the user may sign out or switch accounts while requests are pending or in flight, so a late answer or a queued request can act on the wrong account. A further failure arises on a cold start: the app writes its in-memory queue or cache index to disk before it has read the copy the previous run left, silently overwriting pending work. Finally, sign-out is usually immediate, so any queued work is discarded without telling the user.

## Summary of the Disclosure

The disclosed client holds each action in a single pending slot for a configured undo period, hiding the object from its list immediately. Undo during the period restores the object without any request. When the period ends, or another action starts, the held action is sent. A missing connection or a cancelled send moves the action to an offline queue, minting its idempotency key at that moment. The queue is stored encrypted on the device; each sealed record carries a schema version and the owning account identifier inside the ciphertext. Writes to the stored queue or cache index happen only after the stored copy has been read and merged for the same account. Replay distinguishes permanent refusals from transient failures. Every action, read, prefetch and index write is fenced by an account mark. Sign-out runs ordered hooks under one abort signal with a configurable budget, counts unsent changes, including those on a disk copy never read in the session, reports them as lost, and then shreds the store by removing the key first.

## Detailed Description

Each object kind registers how to perform an action and how to label it. When the user acts on an object, the client builds a hold record containing the object kind and identifier, the action, a display title, a function that performs the request, a function that hides the object from its list, a function that restores it, and the payload needed to replay the request later. If another hold is already pending, it is committed (sent) at once and its undo notice is removed, so any undo notice on screen always refers to an action that can still be undone. The new object is hidden, an account mark is captured (a session epoch counter plus the signed-in user key), a timer is set for the configured undo period, and an undo notice is shown.

Undo during the period clears the timer, empties the pending slot, restores the object and optionally reopens it. It is ignored if the account mark no longer matches. No server restore operation is needed.

When the timer fires, the hold is committed. If the device is offline, the action is parked. Otherwise the perform function is called with an abort controller linked to any outer signal, and the running promise is recorded in a map of in-flight sends. On success a confirmation is shown, but only if the account is unchanged. On failure: if the account changed meanwhile, nothing at all happens (no queue, no restore, no notice); if the error indicates no connection, the action is parked; if the send was aborted, the action is queued; if the server rejected authentication, the object is restored silently; any other error restores the object and tells the user why the action did not happen. The first send carries no idempotency key, because it is sent exactly once in the user's presence.

Parking an action means minting a random version-4 UUID as its idempotency key and adding an entry (kind, identifier, action, key, payload, title, timestamp) to the queue. If local caching is disabled, nothing can wait and the action is treated as refused with the object restored. A queued action shows a notice that it will sync, and its undo removes the entry by key, again only for the same account.

The queue lives in memory and on disk. On disk it is one record in a browser-side database, encrypted with AES-GCM under a random 256-bit key stored in the platform's secure key store, device-only. Each sealed value is a JSON object containing the schema version, the user key and the payload. A read returns nothing for a missing record or for one whose decrypted version or user does not match, but throws a distinct unreadable error when a record exists and cannot be decrypted or its key is missing, so an unreadable store is never mistaken for an empty one. Binary attachments are sealed separately, with an owner tag sealed on its own so no user identifier is kept in clear.

Every write of the queue first awaits a one-time hydration for the current account. Hydration reads the disk queue and merges it with memory: disk entries not already present by key are placed ahead of the in-memory entries, preserving creation order. Two account fields gate writing: the account whose disk copy was read, and the account whose disk copy may be written. A successful read sets both. A read that fails leaves the disk copy untouched and is retried by the next write; only if the failure was the unreadable kind may the next write replace the disk copy. If the account changes while a read is outstanding, nothing from that read enters memory. The cache index (which objects are kept, and remembered list orders) uses the same read-before-write rule.

Replay runs on reconnect events and on demand, one replay at a time. It hydrates, then sends each entry in order with its key in the request body. A no-connection error or an abort stops the replay and leaves the entry waiting. A 4xx status other than request timeout and too-many-requests is a refusal: the entry is removed, kept in a bounded list of recent refusals, reported to error telemetry by kind, identifier, action and status only (no title), and announced to the user. Any other failure keeps the entry with its last error.

Sign-out runs registered pre-sign-out hooks in order under a single abort controller whose timer is the configured budget (which may be unbounded). The pending-action hook is registered first: it commits the held action and awaits every in-flight send, aborting them when the signal fires; an aborted send re-enters the queue. The cache hook then replays the queue if online. Each hook is awaited until it settles or until one event-loop turn after the abort, so a hook that ignores the signal cannot block sign-out, and a hook failure is swallowed.

Then the account reset clears memory. Changes still waiting, excluding read-marks, are announced as lost. If the disk queue was never read in this session, it is read now, before the key is destroyed, merged with memory, and its changes are announced as lost. Finally the store is shredded: the key is removed from the secure store first, then all records are cleared, falling back to per-record deletes; any record that cannot be removed causes a reported failure, though it is already unreadable without the key.

## Disclosed Features (each disclosed alone and in combination)

1. A method in which a user action on an object hides the object immediately and sends no request until a cancellable undo period ends, so that undo requires no server-side restore operation.
2. The method wherein at most one action is held at a time and starting a second action commits the first and removes the first action's undo notice.
3. The method wherein an idempotency key is minted only when an action enters the offline queue, after a send found no connection or was aborted.
4. The method wherein every asynchronous action, read, prefetch and stored-index write captures an account mark, and a result that settles after the signed-in account changed is discarded without any effect.
5. A system storing an offline action queue encrypted per device, each sealed record carrying a schema version and owning account identifier inside the ciphertext, a mismatched account or version reading as a miss and an undecryptable record raising a distinct unreadable error.
6. The system wherein a write of the stored queue or cache index is permitted only after the stored copy was read and merged for the same account, or was found unreadable, and a failed read leaves the stored copy untouched.
7. The method wherein replay of the queue stops at the first no-connection or abort, removes and reports an entry refused with a permanent client-error status, and keeps an entry that failed transiently together with its reason.
8. The method wherein sign-out runs ordered hooks under one time-bounded abort signal: commit the held action, await in-flight sends, replay the queue, and re-queue sends cancelled by the signal.
9. The method wherein changes that could not be sent at sign-out, including those in a stored queue never read in the session, are counted and reported to the user as lost, read-marks excluded.
10. The method wherein the local store is destroyed by removing the encryption key from the secure key store before deleting the encrypted records.

## Variations

- The undo period may be fixed per action type rather than configured globally.
- The pending slot may hold a small bounded set of actions instead of one, each with its own undo notice, committing the oldest when the bound is reached.
- The account mark may be a single monotonically increasing session counter without the user key, or a hash of the session token.
- The idempotency key may be carried in a request header rather than the body where cross-origin policy permits.
- The encrypted store may be a file-per-record store or a native key-value store instead of a browser-side database.
- The sign-out drain may run in the background after the user interface has already returned to the sign-in screen, with the shred deferred until it completes.
- Replay ordering may group entries by object so that later actions on the same object supersede earlier ones.

## Note on omitted parameters

The undo period, the duration of notices, the sign-out time budget, the number of recent refusals retained and the cache size limits are configuration values and implementation choices.
