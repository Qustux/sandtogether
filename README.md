# SandTogether — Co-op Multiplayer mod for Sandustry

> ### ⚠ Unofficial fork — updated for Sandustry 0.5.6
>
> This is an **unofficial fork** of [SandTogether by Kamil Padula](https://github.com/IronBamBam1990/sandtogether).
> All credit for the mod belongs to the original author. Same MIT licence. Not affiliated with the
> author or the Sandustry team. If an official 0.5.6 release appears upstream, use that one instead.
>
> **⚠ Both players must run the same build.** This fork reports version `0.9.165-beta`, so pairing it
> with the Workshop build shows a MOD VERSION MISMATCH warning. That is intentional — the two builds
> speak slightly different protocols.

## What's different in this fork

### Fixed: the mod works on Sandustry 0.5.6 again

The 0.5.6 update broke most of the mod. Co-op still connected, so it looked like it worked, but
21 of the mod's 29 game hooks no longer matched — building, demolishing, the vacuum, the grabber,
the flamethrower, the volcanizer, the caulk blaster, the tech tree and creatures all stopped syncing.
Placing something and having it silently not appear for the other player was this.

### Fixed: joining no longer hangs for minutes

Under some conditions the world transfer restarted over and over instead of finishing — in one
recorded case 41 times in a row. The joining player would end up walking around a stale copy of the
world: things wouldn't place, digging was invisible to the person doing it, items behaved oddly.

### Fixed: the world is smooth for the joining player

The host was misreading a healthy connection as congestion and throttling itself down to a fraction
of its normal rate, then recovering, over and over. For the other player that looked like a low tick
rate — sand, falling resources and conveyors moved in jerks. Now steady.

### Fixed: Ctrl+Z is personal

Undo used to be shared: pressing Ctrl+Z would often undo *the other player's* last action instead of
your own. Each player now has their own undo history. Undoing a drag-placed row is one press instead
of one press per block, and undoing a large area no longer takes many seconds.

### New: you can see what other players are doing

Previously another player was just a cursor with the same small square under every action — you
couldn't tell whether they were building, digging or grabbing. Now the preview shows the real thing:

* **Building** — the actual structure graphics at the exact spot it will be placed, with the correct
  build mode (single / line / rectangle), correct snapping, angled lines and left/right variants, plus
  a direction arrow for conveyors.
* **Digging** — the exact area the shovel is about to dig, in the right place (next to the player,
  where the game itself computes it) and only the cells that actually contain something.
* **Grabber** — the tool's real area, and the material it is carrying, drawn in that material's own
  colours in the same layout the carrier sees.

### New: names above what people are holding

The item or material in someone's hands is now labelled above their cursor — "Gold ×14", "Sediment",
"Collector". It works for other players and for yourself, so you can tell at a glance what your
friends are using, and what you just picked up. The label follows the game's language.

### Known issue

Undoing a **very large** demolished area can leave red blocks on the other player's screen for about
ten seconds before they resolve. Small and medium areas recover in about a second. This is known and
will be fixed — testing needs two people and my co-tester is away at the moment.

**Author: Kamil Padula** · Contributors: **dotNine**, **Knight-HD**, **DwoaC**, **Cr0ss0vr**, **TCentraL**, **UwUDev** · [Steam Workshop page](https://steamcommunity.com/sharedfiles/filedetails/?id=3784750764)

Play [Sandustry](https://store.steampowered.com/app/2764460/Sandustry/) together over the internet — no server, no port forwarding. Steam friend invites (or LAN), up to 4 players, one shared live world: digging, fluids, building, tools, resources and story progression synchronized. Steam achievements keep working.

> ⚠️ Early Access game with no official mod loader — this mod patches the game files. Expect breakage after game updates; we re-anchor quickly (see `src/patches.json`).

## For players

> **Installing this fork:** download the ZIP from
> [Releases](../../releases/latest), unpack it anywhere, and run
> `dist-package/install.bat` (Windows), `dist-package/install.command` (macOS)
> or `dist-package/install-linux.sh` (Linux). Do **not** subscribe to the
> Workshop item for this build - the Workshop copy is the original 0.9.164,
> and the mod's auto-updater only replaces the installed copy when the
> Workshop version is *newer*, so a fork install stays put until the author
> publishes 0.9.166 or later. Both players must install the same build.

Subscribe on the Workshop, then run `install.bat` (Windows), `install.command` (macOS) or `install-linux.sh` (Linux, experimental) from the mod folder **once** — since v0.9.39 the mod auto-updates itself from the Workshop folder at every game launch. Full instructions: [README (EN)](dist-package/README.md) / [INSTRUKCJA (PL)](dist-package/INSTRUKCJA.md). macOS support is community-contributed by **DwoaC** (LAN co-op verified on two Apple Silicon Macs; the Steam-invite callback fix from PR #3 awaits a live test).

## Game versions (Steam branches)

Pick the game branch in Steam: **Sandustry → Properties → Betas**. All players must run the **same game version and the same mod version**.

| Mod version | Game version / Steam branch |
|-------------|-----------------------------|
| **v0.9.165-beta** (this fork) | **0.5.6** (default public version) |
| **v0.9.162** (current, Workshop) | **0.5.5** (branch `0.5.5 with mod support`, same build as the default public version) |
| [v0.9.161-beta](https://github.com/IronBamBam1990/sandtogether/archive/refs/tags/v0.9.161-beta.zip) | **0.5.2** (beta branch `mods`) — unzip, run `install.bat` from `dist-package` |

Old branch (`mods`, 0.5.2):

![Steam branch: mods](docs/img/steam-branch-mods.png)

New branch (`0.5.5 with mod support`):

![Steam branch: 0.5.5](docs/img/steam-branch-055.png)

## Architecture (for contributors)

The game is an Electron app; the simulation is non-deterministic (83× `Math.random` in physics, work-stealing scheduler), so lockstep is impossible. SandTogether is **host-authoritative**:

- **Host** runs the only real simulation and streams the world to clients: dirty 40×40 chunks of `mapData` (RGBA) + `wallData` + `shadowMap` + `authorization` + `sim.cellIds` (collision) + element types, 12 B/cell, **row-delta encoded** (per-row FNV hashes → only changed 40-cell rows are sent, protocol v5), deflate-compressed, prioritized around player positions (fast lane) with a starvation-free FIFO for the rest; fully fogged chunks are skipped until revealed.
- **Client** simulation is paused (manager opcode `SetPaused`); rendering stays alive and reads the mirrored buffers every frame. A re-pause heartbeat protects against the game's own unpause paths (ESC menu).
- **Client actions** (dig, build, demolish, move, vacuum, grabber, flamethrower, cryoblaster, spray, guns…) are captured via small string-patches in `bundle.js` (see `src/patches.json`, multi-version anchor variants) plus game event hooks, forwarded to the host, replayed there authoritatively, and confirmed back through the world stream.
- **Transports**: Steam P2P (lobbies, invites, `+connect_lobby`, lobby-ID clipboard join) and a dependency-free WebSocket (LAN), both with auto-reconnect. Networking lives in the Electron main process (`src/st-main.js`) because the renderer reloads between scenes.
- **Shared progression**: research/upgrade pool, tech tree, story steps, critter collection and factory-process counters are host-authoritative and synced at 1 Hz; client purchases forward the real cost (resource diff) for the host to deduct.
- **Auto-update**: at every game launch `st-main.js` compares the mod version in the Steam Workshop folder with the installed one; a newer Workshop copy is installed (files + bundle patches) and the game relaunches once. The author's newer local build is never downgraded.

### Repo layout

| Path | What |
|------|------|
| `src/sandtogether.js` | The mod (renderer side): HUD, world sync, action forwarding/replay, player models, i18n EN/PL |
| `src/st-main.js` | Electron main-process side: Steam P2P + WebSocket transports, invites, relays |
| `src/patches.json` | Anchor/patched string pairs applied to the game's `bundle.js` (+ per-game-version variants) |
| `src/st-preload-append.js` | Preload bridge (`sandtogetherNet`) |
| `src/patch.js` | Node-based patcher (dev convenience) |
| `dist-package/` | What players get: pure-PowerShell installer (no Node needed) + docs |
| `src/publish-workshop.js` | Steam Workshop publisher (uses the game's bundled steamworks.js) |
| `BUNDLE_MAP.md`, `WORKERS_MAP.md`, `COOP_PLAN.md`, `RECON.md`, `CHANGELOG.md` | Reverse-engineering notes, architecture plan & full changelog |

### Dev loop

1. Install the mod into your game once (`dist-package/install.bat`; macOS: `dist-package/install.command`; Linux: `dist-package/install-linux.sh` — the Unix installers need no Node, they run on the game's own Electron via `ELECTRON_RUN_AS_NODE`).
2. Edit `src/sandtogether.js`, then copy it to `<game>/resources/app/dist/js/sandtogether.js` and restart the game (bundle patches only need re-applying when `patches.json` changes).
3. Two-instance local testing: launch a second copy with `--st-userdata=<dir>` (bypasses the single-instance lock; any `--st-*` arg does) and use `Host LAN` / `Join LAN` on `127.0.0.1`.
4. Logs: `%APPDATA%\Sandustry\logs\main.log` (macOS: `~/Library/Logs/Sandustry/main.log`) — everything the mod does is tagged `[SandTogether]`.

### Contributing

PRs welcome. Keep changes host-authoritative (clients must never mutate the shared world locally except through the confirmed-mirror pattern), keep `patches.json` anchors unique-in-bundle, and note game-build compatibility in your PR. Bug reports: attach both players' `main.log`.

## License

[MIT](LICENSE) © Kamil Padula
