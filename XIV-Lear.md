# XIV-Lear

FFXIV / Dalamud notes — XFC Fight Tracker plugin and custom repository setup.

Created 2026-10-18. Last updated 2026-10-18.

---

## The plugin

**XFC Fight Tracker** — a Dalamud plugin that runs and tracks X-Fight Club matches inside FFXIV.

| | |
|---|---|
| Repository | https://github.com/dhumphrey7/XFC-Fight-Tracker |
| Visibility | **PUBLIC** (required — see below) |
| Local clone | `/home/hermes/.hermes/cache/scratch/XFC-Fight-Tracker` |
| InternalName | `XFC-Fight-Tracker` (permanent, cannot change) |
| Dalamud API level | 15 |
| AssemblyVersion | `0.1.0.0` |
| Default branch | `master` |

Covers all ten XFC match types from https://xfightclub.com/fights: Basic Rules,
Russian Roulette, Battle Royale, Gauntlet, Money in the Bank, Power Bottom Battle
Extravaganza, T.N.T Belt, Tag Team Championship, Iron Crown Belt, XFC World
Championship.

---

## Adding it to Dalamud (the working method)

**There is no JSON file to write.** `pluginmaster.json` already exists in the
repo — that IS the JSON file. You only point Dalamud at its URL.

1. Launch FFXIV **through XIVLauncher** (not the official launcher), Dalamud enabled.
2. In game type **`/xlsettings`** (or `Esc` → Dalamud Settings).
3. Go to the **Experimental** tab (the ⚠️ icon).
4. Find **Custom Plugin Repositories**. Paste:

```
https://raw.githubusercontent.com/dhumphrey7/XFC-Fight-Tracker/master/pluginmaster.json
```

5. Click the **+** button to add it.
6. **Tick the checkbox next to the new row to enable it.** ← easy to miss;
   an unticked repo is silently ignored.
7. Click **Save**.
8. Type **`/xlplugins`** → **All Plugins** tab → search **XFC Fight Tracker** → **Install**.

Then type `/xfc` in chat to open the setup window.

**Sanity check:** open the URL in a browser first. It should show raw JSON
starting with `[`. If it does, the URL is good and any problem is in the paste.

---

## The error: "The entered address is invalid. Did you mean to enter it as a DevPlugin..."

### What it actually means

Dalamud rejects the URL at validation time, before it ever tries to install.
The check is in
`Dalamud/Interface/Internal/Windows/Settings/Widgets/ThirdRepoSettingsEntry.cs`:

```csharp
private static bool ValidThirdPartyRepoUrl(string url)
{
    // Disallow inputting main repo as a ThirdPartyRepo
    if (string.Equals(url, PluginRepository.MainRepoUrl, StringComparison.OrdinalIgnoreCase))
    {
        return false;
    }

    return Uri.TryCreate(url, UriKind.Absolute, out var uriResult)
           && (uriResult.Scheme == Uri.UriSchemeHttps || uriResult.Scheme == Uri.UriSchemeHttp);
}
```

So it only checks: is it an absolute URI with an `http`/`https` scheme? Nothing
about the host, the path, or whether the file exists. If you get this error, the
string in the box is not a clean absolute http(s) URL.

It is **not** a network error and **not** a problem with the repository.

### Causes, tested against that exact function

| What you pasted | Result |
|---|---|
| `raw.githubusercontent.com/dhumphrey7/...` (no scheme) | **REJECTED** |
| `"https://..."` wrapped in double quotes | **REJECTED** |
| `'https://...'` / backticks / smart quotes `“ ”` | **REJECTED** |
| **zero-width space** before the URL (invisible) | **REJECTED** |
| `[repo](https://...)` markdown link form | **REJECTED** |
| `https://raw.githubusercontent.com/dhumphrey7/XFC-Fight-Tracker/master/pluginmaster.json` | accepted |

The zero-width space (`U+200B`) is the nasty one — completely invisible, and it
survives copy-paste from rendered web pages, chat clients and Discord. A
trailing zero-width space is also accepted, so the validator won't catch that
variant; it would fail later at fetch time instead.

### Fix

Click into the field, select all, delete, then paste the exact 87-character
ASCII string:

```
https://raw.githubusercontent.com/dhumphrey7/XFC-Fight-Tracker/master/pluginmaster.json
```

Expected: **length 87**, starts with `h`, ends with `n`, all ASCII.

If it still fails, **type `https://` by hand first**, then paste only
`raw.githubusercontent.com/dhumphrey7/XFC-Fight-Tracker/master/pluginmaster.json`
after it. That defeats any leading invisible character.

### Also check

- You are in **Custom Plugin Repositories**, not the **Dev Plugin Locations**
  fields above it. Dalamud's error text hints at this because both live on the
  Experimental tab. DevPlugin fields expect a **local folder path**, not a URL.
- The field has a 300-character limit — not an issue at 87, but relevant if a
  URL ever gets truncated.

---

## Why the repo must be PUBLIC

Dalamud fetches the repository file with a plain **unauthenticated `GET`**.
`raw.githubusercontent.com` returns **404 for private repos**, and release assets
in a private repo are unreachable too.

This is easy to miss because `gh api` **with a token succeeds** on a private
repo's release assets, so the release looks fine:

```
pluginmaster.json (unauthenticated)  -> 404   (when private)
latest.zip (unauthenticated)         -> 404   (when private)
latest.zip (gh api + token)          -> 200   (works either way)
```

Verified after making it public — all three return **200**:

| Step | Result |
|---|---|
| `pluginmaster.json` | 200 |
| `latest.zip` | 200 (302 → 200), 57,585 bytes |
| `images/icon.png` | 200, `image/png` |
| Validator on live file | 0 errors, 0 warnings |

**Verify the whole chain unauthenticated, not with `gh`:**

```bash
curl -sIL https://raw.githubusercontent.com/dhumphrey7/XFC-Fight-Tracker/master/pluginmaster.json | grep -i '^HTTP'
curl -sIL <DownloadLinkInstall from pluginmaster.json> | grep -i '^HTTP'
```

Both must be `200`.

---

## Bundling XFC into another repository

Append our store entry object to that repo's array. Get the current object
rather than copying a stale one — `LastUpdate` changes every release:

```bash
curl -sL https://raw.githubusercontent.com/dhumphrey7/XFC-Fight-Tracker/master/pluginmaster.json
```

Two things to watch:

- `InternalName` must stay `XFC-Fight-Tracker` — it is the plugin's permanent identity.
- Their file's `LastUpdate` for our entry must be refreshed on each of our
  releases, or their users never get updates.

---

## Releasing an update

Tags must be **four-part** — `AssemblyVersion` drives update detection and
cannot be derived from a three-part semver tag.

```bash
git tag v0.1.0.1 && git push origin v0.1.0.1
```

The `Release` workflow then: builds → asserts the zip contains both the DLL and
the manifest → publishes `latest.zip` → regenerates `pluginmaster.json` with a
fresh `LastUpdate` → validates it → commits it back. Users get the update
through `/xlplugins` automatically.

**Bump the version AND `LastUpdate`, or users never update.** A stale
`LastUpdate` means the installer sees nothing new. `LastUpdate: 0` is
effectively broken (Penumbra ships it and never prompts).

---

## Repository file format — the gotchas

A custom repository is a URL to a JSON **array of store entries**, fetched by
plain unauthenticated `GET`. A store entry = the build manifest + store-only keys.

Required: `Author`, `Name`, `Punchline`, `Description`, `InternalName`,
`AssemblyVersion`, `ApplicableVersion`, `DalamudApiLevel`,
`DownloadLinkInstall`, `DownloadLinkUpdate`, `LastUpdate`.

Things that actually bite:

- **`AssemblyVersion` must be 4 numeric parts** (`1.0.0.0`). It drives update
  comparison.
- **`CategoryTags` only accepts `jobs`, `utility`, `other`.** Anything else —
  including capitalised `Utility` — means the plugin appears under **no**
  category filter. (This was a real bug in our manifest, now fixed.)
- **`RepoUrl` is the source repo page, not the raw JSON.**
- **Omit `IconUrl`/`ImageUrls` rather than leaving them empty** — empty strings
  render broken images.
- Set `ApplicableVersion` (`any`) and `Changelog` explicitly; the SDK does not
  infer them, and the installer surfaces both. (Also fixed in our manifest.)
- `DownloadLinkTesting`/`TestingAssemblyVersion` are only needed to ship a
  separate test build.

### Type coercion — do not "fix" this

Dalamud parses repository files with **Newtonsoft.Json**, which coerces strings
to target types. Live repositories ship `"DalamudApiLevel": "15"` (puni.sh) and
`"LastUpdate": "1696378706"` (UnknownX7) and work fine. A validator that rejects
numeric strings will flag working plugins as broken. Separate genuine breakage
(missing download link) from convention deviations.

### Reference repositories

| Repo | Entries | Notable |
|---|---|---|
| UnknownX7/DalamudPluginRepo | 6 | ships `"LastUpdate"` as a string |
| NightmareXIV/MyDalamudPlugins | 16 | `MaskedCarnivale` has **no download link** — genuinely broken |
| xivdev/Penumbra | 1 | `LastUpdate: 0` — never prompts updates |
| Ottermandias/Glamourer | 1 | cleanest reference; adds `MinimumDalamudVersion` |
| love.puni.sh | 14 | ships `"DalamudApiLevel": "15"` as a string |

---

## Our tooling

- **`scripts/build-repo.py`** — generates `pluginmaster.json` from the build
  manifest so it cannot drift from what shipped. Also validates any repo file
  (`--check-only`, `--strict`).
- **`scripts/make-icon.py`** — writes the installer icon as a real PNG using
  only the stdlib (no Pillow/ImageMagick needed).
- **`docs/repository-compliance.md`** — the full compliance writeup in-repo.
- **Build workflow** — `build` (Windows) + `test` (Linux, 95 rules assertions)
  + `repo-file` (validates the JSON and icon on every push).
- **Release workflow** — triggered by a `v*` tag.

---

## Commands

| Command | What it does |
|---|---|
| `/xfc` | Open the setup window |
| `/xfc setup` | Open the setup window |
| `/xfc fight` | Open the live match window |
| `/xfc start` | Start the match that was set up |
| `/xfc roll <fighter> <value>` | Record a roll by hand |
| `/xfc next` | Resolve the current round |
| `/xfc status` | Print standings to chat |
| `/xfc history` | Open the match history |
| `/xfc config` | Open settings |
| `/xfc cancel` | Abandon the current match |

---

## Still unverified / open questions

- **Never loaded in-game.** The install chain is proven end-to-end and the rules
  engine is covered by 95 tests, but there is no FFXIV client here, so the ImGui
  rendering and the `/random` chat hook are unexercised. First thing to check at
  an event: whether it reads the group's actual roll lines.
- **Damage formula is a guess.** Rules say damage is "recorded" without
  specifying it. Defaults to the attacker's roll; configurable to the margin
  over the defender. Needs a decision.
- **Should it auto-post results to chat?** Currently read-only apart from its
  own tags.
- **Russian Roulette deathroll mechanics** ("roll off that number", and the
  announce-before-rerolling rule) are not fully modelled yet.
- **Power Bottom bracket UI** is scaffolded but has no bracket display yet.
- **Official repo option:** submitting to `goatcorp/DalamudPluginsD17` would
  remove the custom-repo step entirely — members would just find it in
  `/xlplugins`. Needs review/approval and compliance with the AI Usage Policy
  (entirely AI-generated submissions are rejected; undisclosed AI use can get
  you banned).

---

## Related

- Dalamud docs: https://dalamud.dev
- Custom repositories: https://dalamud.dev/plugin-publishing/custom-repositories/
- Plugin metadata: https://dalamud.dev/plugin-development/plugin-metadata
- XFC fight rules: https://xfightclub.com/fights
