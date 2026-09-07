# SandTogether — co-op multiplayer mod for Sandustry (v0.9.165-beta, unofficial fork)

> ### ⚠ Unofficial fork — updated for Sandustry 0.5.6
>
> This build is an **unofficial fork** of SandTogether by Kamil Padula, updated for game
> version **0.5.6**. All credit for the mod belongs to the original author. Same MIT licence.
>
> **Both players must run this same build.** Pairing it with the Workshop version shows a
> MOD VERSION MISMATCH warning — that is intentional, the two builds speak slightly
> different protocols.
>
> Fork: https://github.com/Qustux/sandtogether · Original: https://github.com/IronBamBam1990/sandtogether

**Author: Kamil Padula** · Contributors: **dotNine**, **Knight-HD**, **DwoaC**, **Cr0ss0vr**, **TCentraL**

SandTogether adds full co-op multiplayer to Sandustry — one shared live world over
Steam friend invites (or LAN), up to 4 players. Steam achievements keep working.

Polska instrukcja: zobacz `INSTRUKCJA.md`.

## Installation — ONCE, ever

### Windows

1. Have Sandustry installed from Steam (launch it once normally).
2. Right-click `install.bat` → **Run** (or `install.ps1` → Run with PowerShell;
   if Windows blocks it: `powershell -ExecutionPolicy Bypass -File install.ps1`).
3. Launch the game — the **SandTogether** panel appears in the top-right corner.

### macOS (community-contributed by DwoaC, tested on Apple Silicon)

1. Have Sandustry installed from Steam (launch it once normally).
2. Double-click `install.command` (or run it in Terminal; pass the path to
   `Sandustry.app` as an argument if your Steam library is somewhere unusual).
   No Node.js needed — it runs on the game's own Electron runtime.
3. Launch the game with `SandTogether-Launch.command` — it re-installs the mod
   automatically if a Steam update reverted it, then starts the game through
   Steam.

> **macOS note:** LAN co-op is fully verified (`ip:27777`; same network or a
> VPN like Tailscale). Steam friend invites got a fix in v0.9.41 (the macOS
> Steam library reports callback fields differently) — please report whether
> they work for you now.

### Linux (experimental — testers welcome!)

1. Have Sandustry installed from Steam (launch it once normally; the game has
   a native Linux build).
2. In a terminal: `bash install-linux.sh` (pass the game folder as an argument
   if it is not found automatically:
   `bash install-linux.sh /path/to/steamapps/common/Sandustry`).
   No Node.js needed — it runs on the game's own Electron runtime.
3. Launch the game from Steam — the **SandTogether** panel appears in the
   top-right corner. If a **game** update from Steam reverts the mod, just
   re-run `install-linux.sh` (mod updates are still automatic).

> **Linux note:** untested by the author (no Linux box) — the mod code itself is
> fully cross-platform and macOS works the same way, so it is expected to run.
> Please report success or failure (with `~/.config/Sandustry/logs/main.log`).

**That's it.** The mod auto-updates itself at every game launch from your Workshop
subscription — but only when the Workshop copy is **newer** than the installed one.
This fork is `0.9.165-beta` and the Workshop item is currently `0.9.164-beta`, so your
install stays put. When the original author publishes a build newer than this fork, it
will replace the fork automatically, which is the correct behaviour: use the official
version once it supports your game build.

If you are not subscribed to the Workshop item at all, nothing auto-updates and the
mod simply stays as installed.

## How to play (over the internet, via Steam — no network setup)

**Host:**
1. Panel → **Host (Steam)** → **Invite** (pick your friend).
2. Load/start a game — the world is sent to the joiner automatically.

**Joining player:**
1. Accept the Steam invite (works with the game open or closed).
2. After "World imported!": **Load Game** → load the received world.
3. You now share one live world (the panel shows "host mirror").

**LAN:** Host LAN / Join LAN (type `ip` or `ip:port`, default 27777).
**Chat:** type in the panel's message box, press Enter.
**Hide/show panel:** click its header or Ctrl+Shift+H. **Resync** forces a full world refresh.

## What works (v0.9.39 — full co-op)

- One authoritative live world: sand, fluids, digging, terrain, unlocked zones
  (row-delta streaming + fog-of-war skipping = low bandwidth, fast joins)
- Every tool for every player: shovel, spray, firearms & rockets, vacuum, grabber,
  flamethrower, cryoblaster, demolisher
- One shared factory: build, demolish, move, copy-paste blueprints, pipes,
  signal wiring & buttons, machine settings — on both sides
- Shared team progression: research/upgrade pool, tech tree, story steps,
  objectives, critter collection, factory processes
- Item pickups with full effects; creatures, drones, projectiles, world sounds
- Real player models with equipped tools, build ghosts, grabber crosshairs,
  off-screen arrows; team chat
- Per-player memory: rejoin a world and you're back where you left off, with
  your inventory
- Auto-reconnect on both transports; clear warnings for host-pause, version
  mismatch and different game builds

## What this fork adds on top

- **Works on Sandustry 0.5.6.** The 0.5.6 update silently broke most tool syncing —
  building, demolishing, the vacuum, grabber, flamethrower, volcanizer, caulk blaster,
  tech tree and creatures stopped syncing between players while co-op still connected.
- **Joining no longer hangs.** The world transfer could restart in a loop and leave the
  joining player on a stale copy of the world.
- **The world is smooth for the joining player.** The host used to mistake a healthy
  connection for congestion and throttle itself, which looked like stuttering sand and
  conveyors.
- **Ctrl+Z is personal.** Undo used to hit the *other* player's last action.
- **You can see what other players are doing:** the real building preview at the exact
  spot it will be placed, with the correct build mode and a direction arrow for
  conveyors; the shovel's dig area; the grabber's area and what it is carrying.
- **Names above held items** — "Gold ×14", "Sediment", "Collector" — for other players
  and for yourself, in your game's language.

**Known issue:** undoing a *very large* demolished area can leave red blocks on the
other player's screen for about ten seconds before they resolve. Small and medium areas
recover in about a second. It is known and will be fixed in a later update.

## Important note for the joining player

Don't rely on saving the game while connected as a client — your save captures
the world from the moment you joined. The host's save is the authoritative one.

After a **Steam game update** the mod may be reverted. Re-run the installer from this
folder to put the fork back (macOS: or launch via `SandTogether-Launch.command`).
Note that a game update may also break the mod's anchors again — if tools stop syncing
between players after an update, that is what happened.

## Uninstall

Steam → Sandustry → Properties → Installed Files → Verify integrity of game files,
then delete the `resources\app` folder
(macOS: `Sandustry.app/Contents/Resources/app`).

---
SandTogether by **Kamil Padula** · source: https://github.com/IronBamBam1990/sandtogether (MIT)
Unofficial 0.5.6 fork: https://github.com/Qustux/sandtogether (MIT)
