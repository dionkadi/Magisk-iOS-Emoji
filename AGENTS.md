# AGENTS.md

Magisk module that systemlessly replaces Android's emoji font with Apple's iOS emoji font. No build system, no tests, no CI — the repo contents ARE the module. The release ZIP is created by zipping repo contents (META-INF + root files) with the name `Magisk_iOS_Emoji_<version>.zip`.

## Structure

- `system/fonts/NotoColorEmoji.ttf` — the payload: an actual iOS emoji font, **deliberately named NotoColorEmoji.ttf** so Android picks it up. 34MB binary; never read/edit as text. Updates come from upstream projects (recently mistu01/MEEMEmoji, samuelngs/apple-emoji-linux) via PRs.
- `customize.sh` — install-time logic: copies the font over device-specific variants (Samsung/LG/HTC/etc.), bind-mounts it to `/system/fonts`, mounts fonts into Facebook app dirs, clears caches, symlinks fonts.xml entries, and (if magisk_overlayfs is installed) sources its `util_functions.sh` and deletes `$MODPATH/system`.
- `service.sh` — runs at every boot (LATESTARTSERVICE): overwrites `*emoji*.ttf` files under `/data/data` and `/data/user/0` (Android 13+ storage isolation), locks Facebook apps' `FacebookEmoji.ttf` read-only + `chattr +i`, chmod-000's Messenger font dirs, force-stops FB apps, disables GMS font provider/updater per user, removes `/data/fonts` and GMS font dirs. Logs to `service.log` (5MB rotate, 3 files, 7 days).
- `action.sh` — re-runs `service.sh` on demand from the Magisk app.
- `module.prop` + `updater.json` — module metadata; `updater.json` changelog URL points at `Changelog.md`.

## Conventions

- All scripts are POSIX `sh` for Android (`/system/bin/sh`), Magisk-only utilities (`ui_print`, `set_perm_recursive`, `MODPATH`, `ZIPFILE`). No bashisms.
- Android package names are matched with `pm list packages | grep` — partial matches, so keep exact names.
- No local testing possible. Only verification is syntax checking: `sh -n customize.sh service.sh action.sh`.

## Release process

Bumping the emoji version requires editing **all** of these (they drift out of sync easily — e.g. `service.sh` header log still says "iOS Emoji 18.4" at v26.4):

1. `module.prop` — `version`, `versionCode` (strip dots, ×100: 26.4 → 26400, 26.4.1 → 264100)
2. `updater.json` — `version`, `versionCode`, `zipUrl` (must match the GitHub release asset name)
3. `customize.sh` — the `ui_print` banner string
4. `README.md` and `Changelog.md` — changelog entries (Changelog.md is what users see in-app)

## Device constraints

- Requires Magisk v24+ (installer bootstrap requires 20.4+); tested on Android 10+, Magisk 27005+ and OverlayFS both supported.
- KernelSU is supported since v26.4.1: same ZIP, ksud sources `customize.sh` with `KSU=true` and provides `ui_print`/`set_perm_recursive`; `META-INF` and the `LATESTARTSERVICE` presets are ignored; `service.sh`/`action.sh` work unchanged. Keep the `[ "$KSU" != "true" ]` guard on the magisk_overlayfs block in `customize.sh`.
- Behavior is only verifiable on-device; expect user issue reports for per-app emoji breakage (Messenger/GMS font handling in `service.sh` is the historically fragile part).
