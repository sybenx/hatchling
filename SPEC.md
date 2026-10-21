# Hatchling — Specification

A nostr client for discovering new, unknown-but-promising accounts, paired with
a discovery DVM (**Rookery**) in a separate repo.

This document is the durable reference. It describes the whole system. It is
**not** a build order — see [Milestones](#milestones) for what to build now.

---

## 1. Purpose

Existing nostr discovery is follow-graph proximity: you see accounts your
follows already follow. That surfaces the well-connected, not the good, and it
never surfaces the new.

Hatchling's premise is that a discovery ranker needs a feedback loop, and no
such loop exists on nostr today because engagement signal dies with the client
session. Hatchling is the client that captures it. Rookery is the ranker that
consumes it.

The first version of Rookery is global, not personalized: one model, aggregate
feedback, surfacing accounts broadly getting traction from discriminating
reactors. Freshness and quality are largely user-independent, and global is
dramatically cheaper in storage and privacy exposure.

---

## 2. Constraints

- **Single self-contained HTML file.** All CSS and JS inline. No build step,
  no bundler, no npm. Opens from `file://` or any static host.
- Rationale: it later gets wrapped as a mobile app, and a build pipeline is
  the thing that makes that annoying.
- `nostr-tools` imported from esm.sh as an ES module.
- No framework. Plain DOM.
- `localStorage` is used and expected (telemetry buffer, relay config, soft
  dismissals).

---

## 3. Architecture

Two abstractions must be right from the first commit. Everything else is
replaceable.

### 3.1 Signer

```js
interface Signer {
  getPublicKey(): Promise<string>          // hex pubkey
  signEvent(unsigned): Promise<SignedEvent>
}
```

Ship one implementation, `Nip07Signer`, wrapping `window.nostr`.

**Never touch `window.nostr` outside that implementation.** NIP-07 browser
extensions do not exist on mobile; the portable path is NIP-46 remote signing,
which gets added later as a second implementation. If the interface leaks,
mobile support becomes a rewrite instead of one new file.

All UI reads a single `signerState` — `disconnected | connecting | connected`.
Actions with `needsSigner: true` render disabled when not connected.

### 3.2 DVM client

```js
fetchCandidates({ limit }): Promise<Candidate[]>
  Candidate = { pubkey, reason, sampleEventIds }

reportOutcomes(batch): Promise<void>
```

Backed by a `MockDvm` returning hardcoded pubkeys and logging outcomes to the
console. The real endpoint lives behind a single `DVM_BASE_URL` const.

Transport for v0 is plain HTTP POST, not nostr events. Cheap, debuggable, and
it can move onto NIP-90 job feedback once the payload shape stops changing.

### 3.3 Action registry

Actions are **data, not code**. One registry; menus render from it.

```js
{
  id: 'bookmark',
  label: 'Bookmark',
  icon: …,
  contexts: ['note'],        // 'note' | 'profile'
  needsSigner: true,
  tier: 1,                   // 2 = declared but not implemented
  handler: fn,
  telemetry: { field: 'bookmarked', polarity: 'strong_positive' }
}
```

Tier 2 entries render in a disabled "soon" state. Declaring them now means the
telemetry schema and the menu UI never need widening — adding zap-send later is
one registry entry, not a refactor.

---

## 4. Relays

Default read set, user-editable and persisted:

```
wss://relay.damus.io
wss://nos.lol
wss://relay.primal.net
```

Use `SimplePool`. One subscription per note for its engagement events; filter
client-side rather than opening three subscriptions.

---

## 5. Views

### Feed
Candidates from the DVM, resolved to kind 0 profiles and recent kind 1 notes.
Avatar, display name, truncated npub, content, relative timestamp.

Plain text rendering only for now: escape all user content, linkify bare URLs,
no media embeds, no markdown.

### Thread
A note plus its replies (kind 1 with NIP-10 `e` tags referencing it).

### Profile
kind 0 metadata (banner, picture, display_name, about, nip05) plus that
pubkey's recent notes using the same renderer.

Tabs: **notes** / **notes + replies** / **zapped-most**. The last requires the
author's full zap-receipt history, which is a heavy relay query — load it on
demand, never on profile open.

### Navigation
A **view stack**, not a single "back to feed". Feed → note → author profile →
one of their notes → its parent is a normal path and back must unwind it.

---

## 6. Actions

### Note — tier 1
reply · react (emoji picker, not just `+`) · view replies · view reactions ·
view zaps · view reposts & quotes · view note context (parent) · view profile ·
copy nevent · view raw event JSON · bookmark · mute author · not interested

### Note — tier 2
zap (send) · report (kind 1984) · share to njump

### Profile — tier 1
follow / unfollow · mute · not interested · copy npub · copy nprofile ·
nip05 verification status · add to watchlist · view raw kind 0

### Profile — tier 2
zap profile (lud16) · report

### Notes on specific actions

**React** — content goes in the kind 7 `content` field per NIP-25. Offer `+`,
`-`, and a few emoji.

**Copy nevent / nprofile** — encode *with relay hints and author pubkey*
(`nip19.neventEncode`, `nprofileEncode`). A bare `note1` id is close to useless
to whoever receives it.

**View note context** — resolve NIP-10 root and reply markers. Handle the
deprecated positional-tag form; plenty of clients still emit it.

**nip05** — resolve the `.well-known/nostr.json` and show verified / failed /
unresolved. Never show a name as verified without resolving.

### Engagement panel — build once, reuse

Reactions, zaps, and reposts are one component with different filters: a
collapsed count row on every note, expanding to a tabbed panel.

- **Reactions** — kind 7, grouped by content, each group listing reactors.
- **Zaps** — kind 9735 receipts. Parse the bolt11 amount and the embedded
  kind 9734 description for zapper pubkey and comment. Sort by amount.
- **Reposts** — kind 6, plus kind 1 events `q`-tagging this note.

### List editing (mute, bookmark, follow, watchlist)

NIP-51 lists are replaceable single events, so every edit is a
read-modify-write of the **full** tag list:

1. Fetch the latest version from all relays.
2. Merge by `created_at`.
3. Publish the whole list.

**Never publish a list built from cached state** — it silently drops entries
the user added in another client. If the fetch fails, disable the button rather
than publishing a truncated list.

| List | Kind |
|---|---|
| Mute | 10000 |
| Bookmarks | 10003 |
| Follows | 3 |
| Watchlist (follow set) | 30000 |

### Mute vs. not interested — keep separate

**Mute** is hard, permanent, and published. **Not interested** is a soft local
dismissal: hides the candidate, publishes nothing, reports a distinct negative
signal.

Most negative feedback is soft. Forcing it through mute loses resolution and
makes users reluctant to give any signal at all. This distinction is most of
why the telemetry is worth collecting.

**Mute is a hard client-side filter, independent of anything the DVM returns.**
A muted pubkey reappearing because Rookery didn't get the memo is the bug that
makes people close the app permanently.

---

## 7. Telemetry

The point of the whole application.

### Collection
Per candidate pubkey, tracked via `IntersectionObserver` and action handlers.
Buffered in memory, mirrored to `localStorage` so a refresh loses nothing.

### Flush
Batched through `reportOutcomes()` every 30s and on `visibilitychange`.

### Payload

```json
{
  "candidate_pubkey": "<hex>",
  "shown": true,
  "dwell_ms": 4200,
  "opened": false,
  "opened_profile": false,
  "profile_notes_scrolled": 0,
  "viewed_engagement": false,
  "viewed_context": false,
  "viewed_raw": false,
  "copied_nevent": false,
  "reacted": false,
  "replied": false,
  "bookmarked": false,
  "watchlisted": false,
  "followed": false,
  "unfollowed": false,
  "zapped": false,
  "not_interested": false,
  "muted": false
}
```

Every field is reserved now, including tier 2 (`zapped`), so the payload never
needs versioning.

### Polarity

| Polarity | Fields |
|---|---|
| strong positive | `zapped`, `followed`, `bookmarked`, `watchlisted` |
| positive | `reacted`, `replied`, `opened_profile`, `profile_notes_scrolled`, `viewed_engagement` |
| neutral | `copied_nevent`, `viewed_raw`, `viewed_context` |
| negative | `not_interested` |
| strong negative | `muted`, `unfollowed` |

Declared in the registry so Rookery isn't inferring intent from field names.

### Privacy

**Anonymous by default.** The user's pubkey is never included unless
`shareIdentity` is explicitly true. It defaults to false and there is no UI to
enable it in v0.

Never transmitted: note content, reply text, relay list, follow list,
timestamps finer than the batch.

The UI carries a plain-language statement of exactly what leaves the device.

`profile_notes_scrolled` is the field to reconsider if `shareIdentity` is ever
added — a browsing pattern is identifying in a way a boolean is not.

---

## 8. Milestones

Build in this order. Do not build ahead.

**M1 — Feed.** Signer, mock DVM, relay pool, feed rendering, thread view,
profile view, view stack, reply, react, mute, not interested, telemetry
capture and flush. The action registry exists and declares everything in
§6, with unimplemented entries at tier 2.

**M2 — Depth.** Engagement panel, remaining tier 1 actions, list editing for
bookmarks and watchlist, nip05 resolution, raw JSON views.

**M3 — Real backend.** Point `DVM_BASE_URL` at Rookery. Zap display (read-only
kind 9735 parsing) lands here.

**M4 — Signing and sending.** NIP-46 remote signer. Zap sending via NIP-47
(Nostr Wallet Connect) — WebLN is desktop-only and not the portable path.

### On zaps

Reading is trivial and high value: zap receipts are public, and zaps are the
only engagement metric with a real cost attached, which makes them far harder
to farm than reactions. Rookery should weight them heavily.

Sending costs a week — LNURL callback, kind 9734 request, invoice handling,
wallet connection flow, and its own error surface. Display in M3, send in M4.

---

## 9. Out of scope

DMs, image upload, service worker, settings panel beyond relay list,
long-form (kind 30023), communities, marketplace.

Prefer a working feed over a complete one.
