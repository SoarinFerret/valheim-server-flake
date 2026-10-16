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

Save the response into this session's scratchpad directory rather than `/tmp`.
Substitute its real path for `<scratch>` below — there is no `$SCRATCH` variable set,
so an unsubstituted placeholder would redirect to `/app.json` and fail:

```bash
curl -s https://api.steamcmd.net/v1/info/896660 > <scratch>/app.json
nix run nixpkgs#jq -- '.data["896660"].depots["896661"].manifests.public,
                       .data["896660"].depots.branches.public' <scratch>/app.json
```

The `gid` from `manifests.public` is the new `manifestId`. If it already matches the
manifestId in `default.nix`, the package is current — say so and stop.

Keep the branch's **`timeupdated`** (a Unix timestamp); step 2 uses it. The branch
object also contains `timebuildupdated`, which is a *different, earlier* timestamp —
these can be days apart (seen: `timebuildupdated` Oct 13 vs `timeupdated` Oct 15) and
matching on the wrong one silently lands you on an older patch's version number. Use
`timeupdated`.

Ignore the `default_*` branches — they are pinned legacy builds (pre-Mistlands,
pre-Ashlands, etc.), not the current release.

## 2. Determine the version number

The depot metadata carries no version string, so match the build against the patch
notes by timestamp:

```bash
curl -s "https://api.steampowered.com/ISteamNews/GetNewsForApp/v2/?appid=892970&count=8&maxlength=300&format=json" \
  | nix run nixpkgs#jq -- -r '.appnews.newsitems[] | "\(.date) | \(.title)"'
```

Compare the raw Unix timestamps numerically — the matching news item is typically
within a minute of `timeupdated` (seen: a 30-second gap, news post *after* the build).
`date -u -d @<timestamp>` renders either one readably if you want to sanity-check the
dates by eye.

The matching item names the version (e.g. "Patch 1.0.20" → `1.0.20`, or
"Hotfix 1.0.10 & 1.0.12" → `1.0.12`). If a hotfix post covers several versions, use
the highest. If no news item lines up with the build timestamp, say which version you
inferred and that it is unconfirmed rather than guessing silently.

The version may jump several releases at once (seen: 1.0.15 → 1.0.20), and some builds
get no news post at all. That is expected: the depot only ever carries the current
public build, so there are no intermediate versions to step through and nothing is
missing if the news feed skips numbers.

## 3. Edit the package

In `pkgs/valheim-server/default.nix` set the new `version` and `manifestId`, and
replace `hash` with a placeholder so Nix reports the real one:

```nix
hash = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
```

## 4. Build to get the hash, then build for real

The first build is expected to fail with the real hash. **Grep for it — do not pipe
this to `tail`:** Nix prints the `hash mismatch` block early and then buries it under a
cascade of "Cannot build … 1 dependency failed" errors for every dependent derivation,
so the tail of the output does not contain the hash.

```bash
NIXPKGS_ALLOW_UNFREE=1 nix build .#valheim-server --impure --no-link 2>&1 \
  | grep -E "hash mismatch|specified:|got:"
```

A failed fixed-output derivation caches nothing, so losing the hash from the output
costs a full re-download, not a cheap retry — capture it on the first attempt.

Copy the `got:` value into `hash`, then run the build again (without `--no-link`) to
confirm it completes. The depot download is ~2 GB, so allow a generous timeout
(10-15 minutes) on both builds.

Do not report the update as done until the second build succeeds.

## 5. Wrap up

Run the formatter check if you touched formatting:

```bash
nix flake check --impure
```

Changing only the three string values does not touch formatting, so this is usually
skippable — `git diff` should show exactly three changed lines. The `result` symlink
left behind by the verifying build is gitignored and does not need cleaning up.

Commit only if asked. A one-line message naming the version is the house style
(recent history: "valheim 1.0.15", "new valheim build", "update manifest id").
