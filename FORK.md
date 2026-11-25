# Fork operations manual — `teakiie/gaggimate`

**Read this before touching anything.** This file is the single source of truth
for *why this fork exists* and *how to ship from it*. It is written so that a
person or agent with **zero prior context** can pick the work up from here.

Upstream project: <https://github.com/jniebuhr/gaggimate> (docs: <https://gaggimate.eu/>).
Nothing here describes how to *use* a GaggiMate — it describes how **this fork's
firmware gets built, released and delivered over-the-air**.

Last verified against the repo: **2026-11-25**.

---

## 1. What this fork is for

The owner runs a Gaggia with a GaggiMate **display** installed. The goals, in
priority order:

1. Run **his own modified firmware** on the machine.
2. Keep receiving **upstream improvements**.
3. Deploy changes **over-the-air (OTA)** — no USB cable, no reflashing ritual.

The way those three combine: **this fork is always `upstream + my patches`, and
the machine's OTA endpoint permanently points at this fork's Releases.** There is
only ever one firmware source. See §8 for why there is no "official vs. custom"
toggle.

---

## 2. Current state

| Thing | Value |
|---|---|
| Fork default branch | `master` |
| Upstream default branch | `master` |
| Latest fork release | **`v11.8.1`** — built & published 2026-06-20, all assets present |
| Fork point (upstream commit this fork sits on) | `d941710` = `v1.8.1-121-gd9417101` |
| Upstream latest release | **`v1.10.0`** (2026-11-18) |
| How far behind | ~**308** commits; upstream shipped `v1.9.0` and `v1.10.0` since the fork point |
| Functional code changes vs. upstream | **none** — see §3 |
| Issues on this repo | **disabled** (this matters — see §9) |

The OTA pipeline is **proven end-to-end**: tag `v11.8.1` → `Release` workflow
succeeded → GitHub Release carries `display-firmware.bin` + `board-firmware.bin`
(plus USB-recovery assets) → the machine can pull it.

---

## 3. The complete fork delta

`git diff d941710..master` — apart from this documentation (`FORK.md`,
`CLAUDE.md`, `AGENTS.md`, the README banner), the fork is **four changes**,
nothing else:

| # | Change | File |
|---|---|---|
| 1 | `RELEASE_URL` repointed to this fork | `src/display/plugins/WebUIPlugin.h:20` |
| 2 | **Added** the release pipeline | `.github/workflows/release.yml` |
| 3 | **Deleted** inherited upstream tag-build workflow | `.github/workflows/build.yml` |
| 4 | **Deleted** inherited upstream nightly workflow | `.github/workflows/build-nightly.yml` |

Plus, **on branch `dev` only** (commit `19a857d`, not yet on `master`):

| 5 | **Added** upstream release watcher | `.github/workflows/upstream-watch.yml` |

There are **no modifications to firmware behaviour yet**. The fork so far is pure
delivery plumbing; any feature work is additive on top of this.

> **These are the patches that must survive every upstream merge.** When a merge
> conflicts: keep *ours* for the `RELEASE_URL` line, the workflow files and the
> README banner; take *theirs* for everything else.

Keep this file honest: when the delta changes, update the table above in the same
commit.

---

## 4. How GaggiMate OTA actually works

Facts established by reading the code, not assumptions. Re-verify the cited lines
before relying on them — they move as upstream refactors.

### Two chips

- **Display** (LilyGo T-RGB) — UI (LVGL/EEZ), WiFi, web server + WebSocket,
  plugins, BLE link, **the OTA client**, brew logic. The **web UI is compiled into
  the display app image** (upstream GM-106), not into the filesystem.
- **Controller / board** — real-time hardware only: PID/heater, pump, valve,
  sensors. No WiFi. It is updated **through** the display over **BLE OTA**.

### The endpoint is compile-time, not a setting

```cpp
// src/display/plugins/WebUIPlugin.h:20
const String RELEASE_URL = "https://github.com/teakiie/gaggimate/releases/";
```

The web UI only lets you choose a **channel**, never a repository:
`latest` (stable) or `nightly` → `WebUIPlugin.cpp:63` and `:494`.

### Update resolution (`lib/OTA/src/GitHubOTA.cpp`, `lib/OTA/src/common.cpp`)

- Stable: requests `<RELEASE_URL>latest`; GitHub 302-redirects to
  `/releases/tag/vX.Y.Z`; the firmware parses the version **out of the tag**,
  rewrites `tag` → `download`, and fetches assets from there.
- **Asset names are hardcoded** (`WebUIPlugin.cpp:72`): `display-firmware.bin`,
  `display-filesystem.bin`, `board-firmware.bin`.
  **Renaming a release asset breaks OTA silently.**
- The **filesystem image is NOT flashed over OTA** (`GitHubOTA.cpp:113-115`) — the
  web UI lives in the app image, so user data (profiles, shot history) survives
  updates. `display-filesystem.bin` exists for **fresh USB installs only**.
- Version comparison is strict semver, `new > current` (`common.cpp:101`).
- The `nightly` channel additionally needs a `version.txt` asset
  (`common.cpp:61`); stable does not.

### Device behaviour

The display **auto-checks** on a timer (`WebUIPlugin.cpp:126`) and surfaces
"update available", but only **flashes when the user clicks** "Update Display" /
"Update Controller" (`WebUIPlugin.cpp:114`). That manual click is deliberate —
nobody wants a reflash mid-shot. Do not "fix" it into an auto-flash.

### Version stamping

`scripts/auto_firmware_version.py` runs
`git describe --tags --dirty --exclude nightly --exclude db` and writes
`src/version.h`. **The git tag *is* the firmware version.** Build from a clean,
tagged commit — a dirty tree appends `-dirty` and poisons the comparator.

---

## 5. Versioning rule — read this twice

Upstream versions are `v1.x.y`. This fork tags **`v11.x.y`** — upstream's version
with a `1` glued onto the major.

| Upstream | This fork |
|---|---|
| `v1.8.1` | `v11.8.1` ← current |
| `v1.10.0` | `v11.10.0` ← next, after merging upstream `v1.10.0` |

Why: the device only offers an update when `new > current` in semver. A fork
version has to outrank **both** what is on the machine and anything it might ever
compare against. `11 > 1` guarantees that forever, with no per-release arithmetic.

**Traps:**

- **Never** use a prerelease suffix (`v1.8.1-mod`, `-rc1`). Prereleases sort
  **lower** than the plain version, so the machine will refuse the update.
- **Never** mark the GitHub Release as "prerelease" — the stable channel skips it.
- Tags must be `vX.Y.Z` with the leading `v`; the parser expects it.

---

## 6. Ship a release

Everything downstream of a pushed tag is automated by
`.github/workflows/release.yml`: build web UI → controller firmware → display
filesystem → display firmware, then publish a GitHub Release via
`ncipollo/release-action` with `out/*.bin`.

```bash
git clone https://github.com/teakiie/gaggimate.git
cd gaggimate
git remote add upstream https://github.com/jniebuhr/gaggimate.git

# 1. make your change and commit it on master (or merge dev)
# 2. pick the next version per §5 and tag a CLEAN tree:
git tag v11.10.0
git push origin master --tags
```

Then:

1. Watch **Actions → Release**. It is slow on a cold PlatformIO cache.
2. Confirm the Release carries `display-firmware.bin` **and**
   `board-firmware.bin` under exactly those names, and is **not** flagged
   prerelease.
3. On the machine: **System & Updates** shows "update available" → click
   **Update Display** (and **Update Controller** if the board firmware changed).
   The display reboots; the controller is flashed over BLE.

### What stays manual, on purpose

Your code edits · upstream-merge conflict resolution · deciding and creating the
version tag (that tag **is** the ship signal) · clicking Update on the machine.

---

## 7. Merge upstream

```bash
git fetch upstream --tags
git checkout -b merge/v1.10.0 master
git merge v1.10.0                 # upstream's tag, not a branch
# conflicts: keep OURS for the RELEASE_URL line + .github/workflows/*
#            keep THEIRS for everything else
#            if upstream re-adds build.yml / build-nightly.yml, delete them again
pio run -e display && pio run -e controller   # must compile before you tag
git checkout master && git merge --no-ff merge/v1.10.0
git tag v11.10.0 && git push origin master --tags
```

After every merge, re-verify §3: `git diff <new upstream base>..master` should
still show exactly the fork delta and nothing accidental.

---

## 8. Design decision: no "official vs. custom" toggle

A UI switch to flip `RELEASE_URL` between upstream and this fork was considered
and **rejected**. The toggle would be *code living in the app partition*;
selecting "official" overwrites that partition with upstream firmware, which does
not contain the toggle. You would be stranded on stock with no way back except
USB. NVS settings survive an OTA; **code cannot survive being overwritten by
firmware that lacks the feature.**

Hence: one source, always this fork. Going back to stock means deliberately
flashing upstream over USB.

---

## 9. Open items / next actions

1. **Merge upstream `v1.10.0`** (§7) and ship `v11.10.0`. The fork is ~5 months
   and two upstream releases behind; the longer it waits, the uglier the
   conflicts.
2. **`upstream-watch.yml` is stranded on `dev`.** GitHub runs `schedule:`
   triggers **only from the default branch**, so the daily watcher never fires
   today. Merging `dev` → `master` activates it.
3. **Issues are disabled on this repo** — and `upstream-watch.yml` works by
   *opening an issue*. Enable Issues in repo Settings, or that workflow fails on
   every run once it reaches `master`. (Repo-settings change — ask the owner.)
4. **The `nightly` OTA channel is dead on this fork.** `build-nightly.yml` was
   deleted (§3), so nothing publishes a `nightly` tag or the `version.txt` asset
   the nightly path needs. Keep the machine's channel on `latest`; wanting
   nightly means writing a nightly workflow first.
5. **Optional, not started:** a PR-check workflow beyond `check.yml`'s static
   analysis (compile both envs + run native tests on PRs).
6. **No feature work has happened yet.** The original motivation — "run my own
   modified firmware" — is still unexercised. The pipeline is ready for it.

---

## 10. Local build

Requires PlatformIO and Node (`.nvmrc` → `lts/jod`, i.e. Node 22).

```bash
./scripts/build_webui.sh      # MUST run before the display build (web UI is embedded)
pio run -e display            # display firmware -> .pio/build/display/firmware.bin
pio run -e controller         # board firmware   -> .pio/build/controller/firmware.bin
pio run -t buildfs -e display # filesystem image (fresh USB installs only)
```

PlatformIO envs: `display`, `display-headless`, `display-headless-8m`,
`controller`, `native_autotune`, `display-sim`.

### Fresh install / USB recovery

Each release also carries `display-bootloader.bin`, `display-partitions.bin`,
`display-filesystem.bin`, `board-bootloader.bin`, `board-partitions.bin`. Those
are for flashing a blank chip over USB — OTA never touches them.

---

## 11. CI inventory

| Workflow | Lives on | Trigger | What it does |
|---|---|---|---|
| `release.yml` | master, dev | push tag `v*` | Builds everything, publishes the GitHub Release. **The OTA source.** |
| `check.yml` | master, dev | every push & PR | PlatformIO static analysis (display + controller) |
| `pr-flash.yml` | master, dev | PRs by the repo owner | Publishes a flashable PR preview to gh-pages |
| `upstream-watch.yml` | **dev only** | daily 06:00 UTC + manual | Opens a tracking issue when upstream publishes a release. **Inactive** — see §9.2 and §9.3 |

Deleted from this fork on purpose: `build.yml`, `build-nightly.yml` — upstream's
own release machinery, which would publish under upstream's conventions.
