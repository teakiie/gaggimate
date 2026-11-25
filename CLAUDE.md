# Agent entry point

**This is a personal fork of `jniebuhr/gaggimate`, not the upstream project.**
Its purpose is to build and deliver the owner's own firmware over-the-air to his
machine.

## Read first

**[`FORK.md`](FORK.md)** — the fork operations manual: current state, the exact
fork delta, how OTA resolves a release, the versioning rule, how to ship, how to
merge upstream, and the open items. Read it before planning any change here.
`CONTRIBUTING.md` and `README.md` are upstream's and describe the upstream
project, not this pipeline.

## Invariants — breaking any of these bricks OTA delivery

1. `RELEASE_URL` in `src/display/plugins/WebUIPlugin.h` **must** point at
   `teakiie/gaggimate`. If an upstream merge reverts it, the machine starts
   pulling upstream's firmware instead of this fork's.
2. Release assets **must** be named `display-firmware.bin` and
   `board-firmware.bin` — the names the firmware fetches, declared as
   `OTA_DISPLAY_*` constants in `WebUIPlugin.h` and hardcoded for the board.
3. Tags are **`vX.Y.Z`**, major `11` (see `FORK.md` §5). No prerelease suffix, and
   never flag the GitHub Release as a prerelease — both make the device skip it.
4. Tag only a **clean** tree. `git describe --dirty` feeds the firmware version;
   a `-dirty` suffix breaks the version comparison on the device.
5. `./scripts/build_webui.sh` runs **before** the display build — the web UI is
   compiled into the display app image.
6. Leave `lib/OTA/certs/ca_roots.pem` alone. Firmware TLS trusts only those roots;
   dropping GitHub's breaks OTA downloads outright.

## Conventions

- Work on `dev`, merge to `master`, then tag. Pushing a `v*` tag is the ship
  signal — it triggers the release build. Do not push tags casually.
- Do not add an "official vs. custom firmware" switch. It was considered and
  rejected for a concrete reason (`FORK.md` §8).
- Formatting follows upstream: `.clang-format`, `./scripts/format.sh`.
