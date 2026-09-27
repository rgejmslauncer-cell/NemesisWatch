# Nemesis Watch — Anti-Cheat for Networked Games

An open-source, layered anti-cheat system for networked games: five
independent layers that can be adopted one at a time, plus a small
multi-tenant cloud backend so the same instance can serve more than one
project.

Built as a learning project exploring how real anti-cheat systems
(BattlEye, EAC, VAC) approach the problem at a user-mode level — server
authority, behavioral analytics, honeypots, and client-side integrity
checks — without a kernel-mode driver.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Your game (Unity, dedicated server)                       │
│                                                             │
│  Layer 1: Pre-launch checks    (local — must run on-machine)│
│  Layer 2: Runtime protection   (local — must run on-machine)│
│  Layer 3: Server authority     (local — needs the game state)│
│                     │                                       │
│                     │  NemesisWatchClient (SDK)              │
│                     ▼                                       │
└─────────────────────┼─────────────────────────────────────┘
                       │  HTTPS + X-Api-Key
                       ▼
┌─────────────────────────────────────────────────────────┐
│  Nemesis Watch Cloud (backend, one instance,                │
│  multi-tenant — every connected project isolated by key)   │
│                                                             │
│  Layer 4: Behavioral analytics                              │
│  Layer 5: Honeypot verification                             │
│  EscalationPolicy — turns findings into Kick/Ban/Flag,       │
│                      with score decay over time              │
│  Violation history — persisted to SQLite                     │
└─────────────────────────────────────────────────────────┘
```

**Why split it this way:** Layers 1–3 need direct access to the game's own
files, memory, and physics scene — they can only run inside the game
itself. Layers 4–5 are pure data/logic and don't care what engine sent the
numbers, so centralizing them means the same backend instance could serve
more than one project without duplicating the analytics logic per project.

## Repository layout

```
NemesisWatch/              Core library — Unity project scripts, organized by layer
  maket/                   Layer 1: pre-launch boot sequence + splash UI
  Client/                  Layer 1 & 2: integrity, process scan, hooks, runtime guard
  ServerAuthority.cs        Layer 3: movement/damage/hit validation
  VisibilityFilter.cs       Layer 3: server-side ESP mitigation
  EscalationPolicy.cs       Shared: violation scoring → action decisions, with decay
  Behavioral/               Layer 4: statistical anomaly detection
  Honeypot*.cs              Layer 5: bait opcodes and save-data fields
  Tests/                    Unity Test Framework unit + play mode tests

NemesisWatchApi/           The cloud backend (ASP.NET Core, .NET 8)
  Program.cs                All HTTP endpoints, API key auth, rate limiting
  Data/NemesisWatchDbContext.cs  EF Core + SQLite persistence
  TenantContext.cs          Per-project isolated anti-cheat state
  TenantRegistry.cs         API key → tenant lookup (SQLite-backed)
  Dtos.cs                   Request/response shapes

NemesisWatchSdk/            What a game project drops in to talk to the backend
  NemesisWatchClient.cs      Thin HTTP client calling the cloud backend
```

## Running the backend

Requires the .NET 8 SDK.

```bash
cd NemesisWatchApi
dotnet run
```

This creates a `nemesiswatch.db` SQLite file next to the app on first run
and seeds two throwaway test keys (`nw_test_key_alpha`, `nw_test_key_beta`)
if they don't already exist — see `TenantRegistry.cs`. Both the tenant
list and full violation history survive a restart.

**Rate limiting**: 50 requests/second per API key (sliding window),
configured in `Program.cs`.

**Score decay**: `EscalationPolicy` decays each player's accumulated score
with a configurable half-life (`DecayHalfLifeHours`, default 24) — a single
old flag fades back toward zero over time rather than following a player
forever. A lone Critical violation (teleport, impossible damage) still
triggers an immediate kick regardless of decay.

## Integrating into a game

1. Import `NemesisWatch/` (layers 1–3) into the Unity project as-is — this
   runs entirely on the game/server.
2. Import `NemesisWatchSdk/NemesisWatchClient.cs`, set `apiBaseUrl` to the
   hosted backend and `apiKey` to a key from `TenantRegistry`.
3. Wherever `ServerAuthority`, behavioral events, or honeypot checks fire
   locally, call the matching `NemesisWatchClient` method.
4. Wire `NemesisWatchClient.OnActionReceived` to the actual kick/ban
   implementation — Nemesis Watch decides the *verdict*, disconnecting a
   player stays framework-specific.

## API reference

All endpoints require header `X-Api-Key: <a registered key>`.

| Endpoint | Purpose |
|---|---|
| `POST /api/v1/violations/report` | Forward a hard-rule violation (teleport, speed hack, damage hack) found by `ServerAuthority` |
| `POST /api/v1/behavior/shot` | Record a shot outcome (hit/headshot) for accuracy tracking |
| `POST /api/v1/behavior/reaction` | Record reaction time (ms) since a target became visible |
| `POST /api/v1/behavior/aimsnap` | Record a view-rotation snap and whether it landed |
| `POST /api/v1/honeypot/opcode` | Check a network opcode against the bait list |
| `POST /api/v1/honeypot/savedata` | Validate a save-data upload against the bait field |
| `GET /api/v1/players/{playerId}/history` | Violation history for one player |
| `GET /api/v1/violations/recent?limit=50` | Most recent violations |

Every `POST` responds with `{ "action": "LogOnly" | "FlagForReview" | "Kick" | "Ban" }`.

## Testing

`NemesisWatch/Tests/` has Unity Test Framework tests covering
`EscalationPolicy` (including score decay), `BehaviorAnalyzer`, both
honeypots, and `ServerAuthority` (movement, fire-rate, wall-blocked hits).
Open **Window → General → Test Runner** in Unity, run EditMode then
PlayMode tests. The backend logic has additionally been verified live via
the running API (see commit history / project notes).

## What this is and isn't, honestly

**Closes:** DLL-injected cheats, memory-edited values, speed/teleport
hacks, fire-rate manipulation, wallhack shots, raw packet manipulation,
save-file editing, and the classic ESP problem via server-side visibility
filtering (enemies outside line of sight are never sent to the client).

**Doesn't close:** kernel-mode cheats (this is entirely user-mode),
manual-map DLL injection (loads code without appearing in the normal
module list — a meaningfully bigger detection problem than standard
injection), and a cheat built specifically against this exact system
rather than a generic public one.

**Behavioral flags are leads, not verdicts.** Severity is capped at Medium
specifically so a skilled human player's genuinely fast reactions don't get
auto-punished — pair a flag with an actual match replay before banning
anyone off statistics alone.

## Publishing this repository on GitHub

If you don't have a GitHub account yet: go to github.com, sign up (13+ is
allowed), verify your email.

Once you have a repository created on github.com (green "New" button on
your profile → name it, e.g. `nemesis-watch` → keep it Public → don't
initialize with a README since this project already has one):

```bash
cd NemesisWatch_Package
git init
git add .
git commit -m "Initial commit: Nemesis Watch anti-cheat system"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/nemesis-watch.git
git push -u origin main
```

You'll need Git installed (git-scm.com) if it isn't already. GitHub will
ask you to authenticate the first time — following its own prompts (a
personal access token, or signing in through the Git Credential Manager
it installs) is the simplest path.

## Contributing

Issues and pull requests are welcome — this started as a learning project
and grew from there, and outside eyes on the security-relevant parts
(honeypot design, hook detection, escalation thresholds) are especially
valuable.

## License

MIT — see `LICENSE`. Use it, modify it, ship it in your own game, just
keep the copyright notice.
