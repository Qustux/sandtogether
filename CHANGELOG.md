# Changelog

## 0.9.165-beta (unofficial fork)

Compatibility with Sandustry **0.5.6**, a set of sync fixes, and two new features: a full action
preview and name labels for held items.
All changes are in `src/sandtogether.js` and `src/patches.json`; installers and `st-main.js` are untouched.

---

### Sandustry 0.5.6 compatibility

The 0.5.6 bundle is structurally identical to 0.5.5 but was re-minified, so most literal anchors
stopped matching: **21 of 29 hooks missed**. None of them is marked `critical`, so the installer did
not fail — it printed the "many tools will NOT sync" warning and co-op ran half-broken.

* `supportedVersions` now includes `0.5.6`.
* 21 hooks got a new leading variant; hook count 29 → 32 (three new hooks, see below).

**How the anchors were regenerated** (reproducible next time the game is re-minified):
each old anchor was turned into a regex where every identifier of length ≤ 2 that is *not* preceded
by a dot (i.e. minified locals and webpack module aliases) became a capture group with
backreferences for repeats, while property names, string literals and longer identifiers were kept
literal. Matching that against the clean `dist/js/bundle.js` extracted from `app.asar` produced
**exactly one** hit for all 21, and the resulting old→new name mapping was applied to each `patched`
string.

Two anchors needed manual correction because their `patched` text references identifiers that do not
appear in the anchor:

* **demolish module exports** — in module `40443` the pipe-demolish alias is `m` in 0.5.6
  (`m=n(34142)`, called as `(0,m.Zn)(e,t,n)`), not `p`.
* **entities single-init export** — 0.5.6 renamed `dp`→`op`, `Rp`→`Np`, `Np`→`Dp`, `Ip`→`Lp` and
  added `Fp(t)` (pet-timer normalisation) to the init sequence.

### New hooks

| Hook | Purpose |
| --- | --- |
| `undo module export (_undoState)` | exports the undo module state so remote changes can be kept out of the local Ctrl+Z history |
| `build preview export (_bpPos/_drawStruct)` | exports the game's own build-preview positions and its structure draw function |
| `build preview export - single place (_bpPos/_drawStruct)` | same, for the single-placement branch |

---

### World transfer restart loop

Joining could hang for minutes: in one log the save transfer restarted **41 times over 2 m 43 s** and
the client never received a complete world, so it played on a stale copy — items would not place,
digging was invisible to its owner.

Cause: `scheduleRxCheck` sent `world-need` every 700 ms regardless of progress, so during a normal
transfer the host received a stream of them. On a stale `tid` the host did `ST._wtx = null;
sendWorld()`, and `sendWorld` is async (`FH.game.save` plus up to 10 s waiting for the file to
appear); for ~3 s it therefore sent nothing at all while the client kept asking with the old `tid`.
The first of those requests to arrive after the new `_wtx` was set restarted everything again.

Fixes:

* stale `tid` now re-announces `world-begin` for the **current** transfer and re-queues its parts —
  the chunks are still in `ST._wtx.parts`, no re-export is needed;
* `ST._wtxPreparing` guard around the async preparation, checked **outside** the `try/finally` so a
  rejected call cannot clear the flag of the running one;
* `world-need` is sent only when the transfer has actually stalled (no new part since the last check,
  or `world-end` already seen).

### Mirror pacing

`w.lag` is measured in batches and its thresholds (`>5`, `>8`, `>25`) were calibrated for a fixed
10 batches/s, but the cadence is adaptive and goes down to 16 ms (~60/s). At that rate the ack
granularity alone (client acks 10×/s) plus RTT is worth ~14 batches on a perfectly healthy link, so
the host classified it as congestion, dropped to 10 Hz, recovered, and oscillated — visible in the
log as alternating `63Hz` and `10Hz`, and to the client as stuttering sand and conveyors.

Lag is now expressed in milliseconds with the known overhead removed:

```js
w.lagMs = Math.max(0, w.lag * (w.gap || 33) - 100 - pingMs);
```

Thresholds are time-based (cadence 300/150 ms, byte budget 700/350 ms, hard stop 2500 ms), and
`SYNC-HOST` prints `lag N/Nms` so the effect is visible in logs.

### Per-player undo (Ctrl+Z)

The game's undo history is filled by `structures:removed` / `moved` / `pasted` and
`afterStructuresPlaced`. The mod applies remote changes through the same functions, so those events
fired locally too and each player's history ended up holding the **other** player's actions — Ctrl+Z
undid someone else's work.

* `_applyingNet` now mirrors into the undo module's `isUndoing`, which the game already uses to skip
  history writes; the previous value is restored rather than forced to `false`, so the game's own
  undo flag survives.
* The client's own actions are pushed to the history explicitly (they are intercepted before the game
  sees them) and consecutive placements within 700 ms are coalesced into one entry, matching how the
  game treats a drag.
* `FH.structures.removeAt` / `removeAtPositions` are forwarded to the host while an undo is running —
  that path was not intercepted before, so undoing a build on the client never reached the host and
  the mirror restored the structure. Removals are batched into a single `demolish` request; sending
  them one by one was slow and left tiles in the QUEUED (red) state.
* After an undo the host re-sends the structures inside the demolished rectangle and marks that
  terrain urgent — the terrain writes made by the undo do not always set `chunkShouldUpdate`, so the
  mirror skipped them and the client kept the stale (red) tiles.
* A client-side check reports foundation tiles that have no structure to the host, which answers with
  either the missing structures or permission to clean the tile.

### New feature: action preview

Previously there was no working action preview. `getBuildIntent` existed, but it fell back to the
hotbar slot and read `item.type`, which is the **category** enum (`X2`: Weapon / Building / Tool /
Mod), not a structure id. The result was one small square, identical for every action and every
structure, so a remote player looked exactly the same whether they were building, digging or
grabbing. The preview logic in this fork is essentially new code.

It now mirrors what the acting player sees:

* **Building** — exact positions and the structure's own graphics are taken from the game's build
  preview loop through two new hooks, so build mode (single / line / rectangle), snap grid, angled
  lines and left/right variants are all correct. A direction arrow is drawn for directional
  structures such as conveyors, constrained to the axes that structure's build mode allows.
* **Shovel** — the dig area is anchored at `session.action.point` (what the game itself uses, not the
  cursor) and only the cells that actually contain something are highlighted.
* **Grabber** — tool footprint from the game's own formula (`Math.sqrt(tool.data.size)`), and the
  tank contents drawn in real element colours from
  `session.colors.scheme.element[type].variants[0]`, in the same layout the carrier sees.

Selection comes from `FH.action.getSelected` instead of being guessed from the hotbar, which is what
makes the above possible at all.

### New feature: name labels above held items

The structure or material in a player's hands is now labelled above the preview — "Gold ×14",
"Sediment", "Collector". This applies to other players and to yourself, so you can tell at a glance
what everyone is holding and what you just picked up. Labels are resolved through
`FH.i18n.t(cfg.nameKey)` so they follow the game language, and variant ids (e.g. a mirrored
conveyor) are mapped back to their base config, which is where `nameKey` lives.

**A game-side gotcha worth calling out:** `FH.rendering.getDrawPos` returns a shared mutable object
(`ke=(e,t,n)=>(we.x=…,we.y=…,we)`), so two calls yield the same object and comparing them to derive
pixels-per-cell always gives `0`. That silently fed a hardcoded fallback of `6` instead of the real
`4`, and every overlay came out ~1.5× too large. Results are copied immediately now and the scale is
taken as `cellSize * session.view.zoom`.

Also fixed: the overlay canvas was sized to the game canvas backing store and stretched, which looked
blurry on HiDPI; a departed player's nametag stayed on screen forever because the canvas was only
cleared when at least one peer was left; peers that vanish without a clean disconnect are dropped
after 45 s of silence.

---

### Known issues

* Undoing a **very large** demolish can leave red, unregistered tiles on the client for 10+ seconds
  before they resolve. Small and medium areas recover in about a second. The cause is understood well
  enough to keep working on, but verifying a fix needs two players at once and my co-tester is away,
  so it is left as a known issue for now and will be addressed in a later update rather than holding
  back everything else.

---

Original mod by **Kamil Padula** — <https://github.com/IronBamBam1990/sandtogether> (MIT).
This fork keeps the same licence.
