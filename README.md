# hatchling

A nostr client for discovering new, unknown-but-promising accounts, paired with
a discovery DVM (**Rookery**) that lives in a separate repo.

Existing nostr discovery is follow-graph proximity: you see accounts your
follows already follow. That surfaces the well-connected, not the good, and it
never surfaces the new. A discovery ranker needs a feedback loop, and no such
loop exists on nostr today because engagement signal dies with the client
session. **hatchling is the client that captures it. Rookery is the ranker that
consumes it.**

[`SPEC.md`](SPEC.md) is the durable reference for the whole system. This README
covers what is actually built.

---

## Status — M1 (feed)

The whole client is one file: [`index.html`](index.html). No build step, no
bundler, no npm. `nostr-tools` is imported from esm.sh as an ES module.

**Built:** signer, mock DVM, relay pool, feed, thread view, profile view, view
stack, reply, react, mute, not interested, telemetry capture and flush.

**Declared but not built:** everything else in SPEC §6. Those entries are in the
action registry at tier 2 and render as disabled "soon" rows, so the menu UI and
the telemetry schema are already the right shape for them.

## Running it

Open `index.html` in a browser. That is the whole procedure.

Chrome refuses ES module imports from `file://`, so from a checkout use any
static server:

```bash
python3 -m http.server 8765
```

then open <http://127.0.0.1:8765/index.html>. Firefox and Safari open the file
directly.

Signing needs a NIP-07 extension (Alby, nos2x, …). Without one the feed, thread
and profile views all work; the actions that publish render disabled.

---

## The two things that have to be right

Everything else in this codebase is disposable. These two are not.

### Signer (SPEC §3.1)

```js
interface Signer {
  getPublicKey(): Promise<string>          // hex pubkey
  signEvent(unsigned): Promise<SignedEvent>
}
```

One implementation ships: `Nip07Signer`, wrapping `window.nostr`.

**`window.nostr` is touched in exactly one place — inside `Nip07Signer`.** NIP-07
browser extensions do not exist on mobile; the portable path is NIP-46 remote
signing, which arrives in M4 as a *second implementation of this interface and
nothing else*. If the interface leaks, mobile support becomes a rewrite instead
of one new file. There is a single `signerState` (`disconnected | connecting |
connected`) that all UI reads, and actions declaring `needsSigner: true` render
disabled when it is not `connected`.

### Action registry (SPEC §3.3)

Actions are data, not code. There is one registry and every menu in the app
renders from it — there is no hand-written menu anywhere.

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

Every action in SPEC §6 is declared, including the ones that land in M2–M4.
Declaring them now means the telemetry schema and the menu UI never need
widening: adding zap-send later is one registry entry, not a refactor.

`telemetry` is `null` where an action sets no telemetry field. SPEC §7's
polarity table covers action-driven fields only — `shown`, `dwell_ms` and
`opened` are collected passively (observer and navigation) and carry no
polarity, and report / share / copy-npub have no field at all.

### DVM client (SPEC §3.2)

```js
fetchCandidates({ limit }): Promise<Candidate[]>   // { pubkey, reason, sampleEventIds }
reportOutcomes(batch): Promise<void>
```

M1 runs on `MockDvm`: hardcoded candidate pubkeys with plausible reason strings,
and `reportOutcomes` printing each batch to the console. The real endpoint sits
behind the single `DVM_BASE_URL` const; M3 points it at Rookery. Transport for
v0 is plain HTTP POST rather than nostr events — cheap, debuggable, and it can
move onto NIP-90 job feedback once the payload shape stops changing.

---

## Telemetry and privacy

Per candidate pubkey, hatchling records whether the account was on screen, for
roughly how long, and which actions you took on it. Buffered in memory, mirrored
to `localStorage` so a refresh loses nothing, flushed through `reportOutcomes()`
every 30 seconds and on `visibilitychange`. The full SPEC §7 payload shape is
sent, tier-2 fields included, so the payload never needs versioning.

Records are cumulative and idempotent: each flush sends a record's current full
value, booleans are monotonic and `dwell_ms` is a running total, so a dropped or
duplicated batch costs nothing. Only records that changed since the last
successful flush are sent, and only those survive a refresh as still-owed.

`dwell_ms` accrues only while a card is at least half on screen *and* the tab is
foregrounded. Time on a backgrounded tab is not attention.

**Anonymous by default.** The user's pubkey is never attached: `shareIdentity`
is `false` and there is deliberately no UI to enable it. Never transmitted: note
content, reply text, relay list, follow list, or timestamps finer than the
batch. The shield icon in the header opens a plain-language statement of exactly
what leaves the device, followed by the verbatim JSON currently buffered.

If `shareIdentity` is ever added, `profile_notes_scrolled` is the field to
reconsider first — a browsing pattern is identifying in a way a boolean is not.

---

## Behaviour worth knowing

**Mute vs. not interested are kept apart, on purpose.** Mute is hard, permanent
and published as a NIP-51 kind 10000 list. Not interested is a soft local
dismissal: it hides the candidate, publishes nothing, and reports a distinct
negative signal. Most negative feedback is soft; forcing it through mute loses
resolution and makes people reluctant to give any signal at all. That
distinction is most of why the telemetry is worth collecting.

Mute is also a **hard client-side filter**, independent of anything the DVM
returns. A muted pubkey reappearing because Rookery did not get the memo is the
bug that makes people close the app permanently.

**List edits are read-modify-write of the full tag list.** NIP-51 lists are
replaceable single events, so every edit fetches the latest version from all
relays, merges by `created_at`, and publishes the whole list. A list is never
built from cached state — that silently drops entries added in another client.
If no relay answers, the edit is refused rather than publishing a truncated
list, and the list event's `content` is carried over untouched because on a mute
list it may hold encrypted private entries.

**The feed is built not to shift.** Skeleton cards paint immediately; the first
real batch swaps in at relay EOSE or 1.4s, whichever comes first, in the
ranker's order; later arrivals append below the fold and never above what you
are reading. Avatars are fixed-size boxes and the display-name line has a
fixed-height placeholder, so a kind 0 resolving cross-fades without moving a
pixel. Long notes are clamped with an expander so one wall of text cannot
dominate the scroll.

**Navigation is a view stack, not a single back-to-feed.** Feed → note → author
profile → one of their notes is a normal path, and back unwinds it one step at a
time, in step with the browser/Android back gesture. Views below the top stay
mounted with their scroll position intact.

**Rendering is plain text only** — content is escaped structurally (it only ever
reaches the DOM as a text node), bare URLs are linkified, and there are no media
embeds and no markdown. Bare `nostr:` bech32 entities are elided to a skimmable
length with the full value on hover; they are not resolved or linked, which is
NIP-27 rendering and out of scope.

## Relays

Default read set, user-editable and persisted, edited via the antenna icon:

```
wss://relay.damus.io
wss://nos.lol
wss://relay.primal.net
```

`SimplePool` is used throughout. The feed issues one subscription carrying one
filter per candidate, so relays apply `limit` per author without opening a
subscription each.

## Working on the file

`index.html` is organised in the order SPEC.md is: config, signer, DVM client,
telemetry, relays and nostr helpers, UI primitives, the action registry,
tier-1 handlers, shared rendering, views, then boot. Section banners cite the
spec paragraph they implement.

To add an action, edit its registry entry: move `tier` to `1` and give it a
`handler`. Nothing else in the menu code needs to change.

## Not in M1

Engagement panel, remaining tier-1 actions, bookmark and watchlist list editing,
nip05 resolution and raw JSON views (M2). Real Rookery backend and zap display
(M3). NIP-46 remote signing and zap sending over NIP-47 (M4). Out of scope
entirely: DMs, image upload, service worker, settings beyond the relay list,
long-form, communities, marketplace.
