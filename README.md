# XIV-Lear

FFXIV / Dalamud notes — starting with the XFC Fight Tracker plugin and how to
distribute it as a custom Dalamud repository.

## Contents

| Note | Covers |
|---|---|
| [XIV-Lear.md](XIV-Lear.md) | XFC Fight Tracker plugin, installing it in Dalamud, the "entered address is invalid" error and its causes, why the repo must be public, repository file format gotchas, and open questions |

## Quick reference

**XFC Fight Tracker** — a Dalamud plugin that runs and tracks X-Fight Club
matches inside FFXIV.

- Plugin source: https://github.com/dhumphrey7/XFC-Fight-Tracker
- Add to Dalamud (`/xlsettings` → Experimental → Custom Plugin Repositories):

  ```
  https://raw.githubusercontent.com/dhumphrey7/XFC-Fight-Tracker/master/pluginmaster.json
  ```

  Then tick the checkbox to enable it, save, and install from `/xlplugins`.

- Type `/xfc` in game to open the setup window.

## Why this exists

Notes written while building the plugin, kept because the non-obvious parts are
easy to forget: the exact URL that Dalamud accepts, the fact that a custom
repository must be **public** (Dalamud fetches it unauthenticated, so a private
repo returns 404), and the repository JSON rules that silently break installs.
