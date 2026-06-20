# StreamerBot_Converter
convert streamerbot exports to json, convert json to streamerbot. much easier for sending full actions/triggers/subactions in and out of Claude
# Streamer.bot Converter (SB ↔ JSON)

A single-file, dependency-free browser tool for converting between Streamer.bot's `.sb` export format and readable JSON — in both directions.

No install, no build step, no server. Open the HTML file in any modern browser and it works entirely client-side.

---

## What it does

Streamer.bot exports (actions, commands, timers, etc.) are distributed as an opaque base64 string. This tool:

- **Decodes** `.sb` exports / export strings → clean, readable, editable JSON
- **Encodes** JSON → a valid `.sb` import string, ready to paste into Streamer.bot

This lets you read, diff, version-control, hand-edit, or programmatically generate Streamer.bot actions instead of treating exports as a black box.

---

## Usage

Open `sb_converter.html` in a browser. Two tabs:

### ⬇ SB → JSON
- Drop a `.sb` file, or paste an export string directly
- Decodes automatically — no buttons to click
- Save the result as a `.json` file

### ⬆ JSON → SB
- Paste JSON into the editor, or drop a `.json` file
- Validates as you type (green border = valid, red = invalid)
- Click **Encode → .sb**
- Copy the result to clipboard or save as `.sb`

Files over 512 KB bypass the textarea and load straight into memory, since browsers can silently truncate large pastes/drops in a `<textarea>`.

---

## The `.sb` Export Format

```
.sb string = base64( "SBAE" + gzip(JSON) )
```

Breaking that down:

1. The JSON payload (actions, commands, timers, etc.) is serialized
2. It's compressed with **gzip**
3. A 4-byte ASCII magic header, `SBAE`, is prepended to the compressed bytes
4. The whole thing is **base64**-encoded into the string you copy/paste/export

Decoding reverses each step: base64 decode → strip the `SBAE` header → gzip decompress → `JSON.parse`.

---

## JSON Schema

This is what Streamer.bot actually expects on import. Get any of this wrong — especially the IDs — and the **Import button will not light up**, with no specific error message telling you why.

### Top level

```jsonc
{
  "version": 23,                      // export format version
  "exportedFrom": "1.0.4",            // Streamer.bot version string
  "minimumVersion": "1.0.0-alpha.1",
  "meta": { /* see below */ },
  "data": { /* see below */ }
}
```

### `meta`

```jsonc
{
  "name": "My Export",
  "author": "YourName",
  "version": "1.0.0",
  "description": "",
  "autoRunAction": null
}
```

### `data`

```jsonc
{
  "actions": [ /* Action[] */ ],
  "queues": [],
  "commands": [ /* Command[] */ ],
  "timers": [ /* Timer[] */ ],
  "websocketServers": [],
  "websocketClients": []
}
```

### Action

```jsonc
{
  "id": "<uuid-v4>",
  "name": "My Action",
  "group": "My Group",          // or null
  "enabled": true,
  "queue": "00000000-0000-0000-0000-000000000000",  // null UUID = no queue
  "alwaysRun": false,
  "concurrent": false,
  "randomAction": false,
  "excludeFromHistory": false,
  "excludeFromPending": false,
  "collapsedGroups": [],
  "triggers": [ /* Trigger[] */ ],
  "subActions": [ /* SubAction[] */ ]
}
```

### Triggers (by type code)

| Type | Meaning | Required fields |
|---|---|---|
| `401` | Chat command | `commandId` (must equal a Command's `id`) |
| `701` | Timer | `timerId` (must equal a Timer's `id`) |
| `102` | Follow/sub range | `min`, `max` |
| `124` | Stream went online | — |
| `133` | Stream went offline | — |
| `2003` | WebSocket custom event | — |

Every trigger also needs: `id` (UUID), `enabled` (bool), `exclusions` (`[]`).

### Sub-actions (by type code)

| Type | Meaning | Key fields |
|---|---|---|
| `99999` | Execute C# code | `byteCode` = `base64(UTF-8 C# source)`, `references` (DLL paths array) |
| `123` | Set variable | `variableName`, `value`, `autoType` |
| `1002` | Delay | `value` (ms, as string), `random` |
| `30` | OBS source visibility | `sceneName`, `sourceName`, `state` (0/1), `connectionId` |
| `1` / `7` | Send chat message | `message` |

Every sub-action also needs: `id` (UUID), `enabled` (bool), `index` (its position in the array), `weight` (`0`), `parentId` (`null` unless nested).

### Command

```jsonc
{
  "id": "<uuid-v4>",
  "name": "MY_COMMAND",
  "command": "!mycommand",       // use "\r\n" to separate multiple aliases
  "enabled": true,
  "mode": 0,                     // 0=contains, 1=exact, 2=startsWith
  "globalCooldown": 0,           // seconds
  "userCooldown": 0,
  "caseSensitive": false,
  "permittedUsers": [],
  "permittedGroups": []
}
```

### Timer

```jsonc
{
  "id": "<uuid-v4>",
  "name": "My Timer",
  "enabled": true,
  "repeat": true,
  "interval": 60                 // seconds
}
```

---

## ⚠️ The #1 cause of silent import failures: UUIDs

Streamer.bot validates **every** `id`, `commandId`, `timerId`, and `parentId` field strictly. It must match real UUID v4:

```
xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx
```

- The **third group** must start with `4`
- The **fourth group** must start with `8`, `9`, `a`, or `b`
- The rest are random hex digits

**This will fail:**
```
vc-cmd-0001-0000-0000-000000000001
```
It's UUID-*shaped* (right number of groups, right dash positions) but isn't a valid v4 UUID, and Streamer.bot rejects it — usually with no error message, just a grayed-out Import button.

**This will work:**
```
a1b2c3d4-e5f6-4a90-8bcd-ef1234567890
```

### Cross-references must match exactly

If a trigger has `"commandId": "X"`, there must be a command somewhere in the export with `"id": "X"` — identical string, no exceptions. Same for `timerId` → a timer's `id`. A real UUID that doesn't match anything is just as broken as a fake one.

### If you're asking an AI to generate this JSON for you

Be explicit about both constraints up front:

> All `id`, `commandId`, `timerId`, and `parentId` fields must be valid UUID v4 — randomly generated, matching `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx` (third group starts with `4`, fourth starts with `8/9/a/b`). Do not use shorthand or human-readable IDs. Cross-references between triggers and their target objects (commands, timers) must match exactly.

The encoder tab in this tool will auto-generate valid UUIDs for any object missing an `id`, but it **cannot fix** invalid IDs that are already present and miswired — those need to be corrected at the source.

---

## Do I need to rename it `index.html`?

**No, not for normal use.** Open the file directly, or host it on any static file server under whatever name you like — the file works standalone regardless of its name.

**Yes, if** you're deploying via **GitHub Pages** and want it to load automatically at your repo's root URL (`https://username.github.io/repo-name/`) without anyone needing to type a filename. GitHub Pages serves `index.html` by default for any directory. If you skip the rename, visitors would need the full path: `https://username.github.io/repo-name/sb_converter.html`.

To enable Pages: repo **Settings → Pages → Deploy from a branch → main / (root)**.

---

## Local use

No build, no install:

```bash
git clone https://github.com/yourname/sb-converter.git
cd sb-converter
open sb_converter.html   # or just double-click it
```

Everything runs client-side using native browser APIs (`CompressionStream` / `DecompressionStream` for gzip, `atob`/`btoa` for base64). Nothing is uploaded anywhere.

---

## Browser support

Requires `CompressionStream` / `DecompressionStream` (Chrome/Edge 80+, Firefox 113+, Safari 16.4+). If you're on an older browser, the gzip step will fail — update or switch browsers.

---

## License

MIT — do whatever you want with it.
