---
name: update-valheim
description: Update the Valheim dedicated server package to the latest Steam build - looks up the current public depot manifest, determines the version number, updates pkgs/valheim-server/default.nix, and rebuilds to verify. Use when asked to "update valheim", bump the server version, or pick up a new Valheim patch/hotfix.
---

# Update the Valheim server package

Everything lives in `pkgs/valheim-server/default.nix`. Three fields change per update:
`version`, `src.manifestId`, and `src.hash`.

Steam IDs (constant): appId `896660` (Valheim Dedicated Server), depotId `896661`
(the Linux depot), and `892970` (the Valheim game app, whose news feed carries the
patch notes).

`jq` is not on PATH in this environment; run it as `nix run nixpkgs#jq -- <args>`.

## 1. Find the latest public manifest

```bash
curl -s https://api.steamcmd.net/v1/info/896660 > /tmp/app.json
nix run nixpkgs#jq -- '.data["896660"].depots["896661"].manifests.public' /tmp/app.json
nix run nixpkgs#jq -- '.data["896660"].depots.branches.public' /tmp/app.json
```

The `gid` from `manifests.public` is the new `manifestId`. If it already matches the
manifestId in `default.nix`, the package is current — say so and stop.

Keep the branch's `timeupdated` (a Unix timestamp); step 2 uses it. Ignore the
`default_*` branches — they are pinned legacy builds (pre-Mistlands, pre-Ashlands,
etc.), not the current release.

## 2. Determine the version number

The depot metadata carries no version string, so match the build against the patch
notes by timestamp:

```bash
curl -s "https://api.steampowered.com/ISteamNews/GetNewsForApp/v2/?appid=892970&count=8&maxlength=300&format=json" \
  | nix run nixpkgs#jq -- -r '.appnews.newsitems[] | "\(.date) | \(.title)"'
```

The patch-notes item whose `date` is within a few minutes of the branch's
`timeupdated` names the version (e.g. "Hotfix 1.0.10 & 1.0.12" → `1.0.12`). If a
hotfix post covers several versions, use the highest. If no news item lines up with
the build timestamp, say which version you inferred and that it is unconfirmed rather
than guessing silently.

## 3. Edit the package

In `pkgs/valheim-server/default.nix` set the new `version` and `manifestId`, and
replace `hash` with a placeholder so Nix reports the real one:

```nix
hash = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
```

## 4. Build to get the hash, then build for real

```bash
NIXPKGS_ALLOW_UNFREE=1 nix build .#valheim-server --impure --no-link
```

This fails with `hash mismatch in fixed-output derivation ... got: sha256-...`.
Copy the `got:` value into `hash`, then run the build again (without `--no-link`)
to confirm it completes. The depot download is ~2 GB, so allow a generous timeout
(10-15 minutes) on both builds.

Do not report the update as done until the second build succeeds.

## 5. Wrap up

Run the formatter check if you touched formatting:

```bash
nix flake check --impure
```

Commit only if asked. A one-line message naming the version is the house style
(recent history: "new valheim build", "update manifest id").
