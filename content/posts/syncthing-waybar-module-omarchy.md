+++
date = '2026-06-09T22:21:41-04:00'
draft = false
title = 'Syncthing Status Module for Waybar (Omarchy)'
tags = ["omarchy", "waybar", "syncthing", "hyprland", "linux"]
categories = ["configuration"]
[cover]
image = "/images/syncthing-waybar-module-omarchy.png"
+++

# Syncthing Status Module for Waybar (Omarchy)

A lightweight custom Waybar module that polls Syncthing's local API and shows sync status as an icon. No tray app, no boost dependency, nothing to break on a rolling-release update. Click the icon to open the Syncthing Web UI.

Icon meanings:

| Icon | Class                                   | Meaning                                            |
| ---- | --------------------------------------- | -------------------------------------------------- |
| `󰴽` | `syncthing-idle`                        | All folders synced / idle                          |
| `󰓦` | `syncthing-syncing`                     | Actively syncing                                   |
| `󰅙` | `syncthing-error` / `syncthing-offline` | Folder error, bad key, or Syncthing not responding |

> [!note] Design choice
> The script fails LOUD. If the API key is missing or Syncthing rejects the request, it shows the error/offline icon instead of falsely reporting idle. A status indicator that shows green when it cannot actually see anything is worse than no indicator, since you would trust a sync that is not being checked.

## Prerequisites

- Omarchy / Arch with Hyprland and Waybar
- Syncthing running as a systemd user service and reachable at `https://127.0.0.1:8384`
- A Nerd Font in your Waybar config (Omarchy default has one)
- `jq` for validation: `sudo pacman -S jq`

If `syncthingtray` is installed and blocking updates, remove it first:

```bash
sudo pacman -R syncthingtray
```

## Step 1: Create the scripts folder

```bash
mkdir -p ~/.config/waybar/scripts
```

## Step 2: Store the API key in a locked-down file

Get the key from the Syncthing Web UI: **Actions > Settings > API Key**.

> [!warning] The key per device is unique
> Each system has its own key. Make sure you copy the key for THIS machine, and that you click **Save** in the Web UI after regenerating. A stale key returns `Forbidden` (HTTP 403).

Open a new file and paste the key as the only line:

```bash
nvim ~/.config/waybar/scripts/.syncthing-key
```

Lock it to owner read/write only:

```bash
chmod 600 ~/.config/waybar/scripts/.syncthing-key
```

> [!danger] Watch the chmod syntax
> Use `chmod 600` (sets permissions). A `chmod -600` (with a dash) REMOVES the owner bits and you get `Permission denied` even on your own file.

Verify you can read it as yourself, with no error:

```bash
cat ~/.config/waybar/scripts/.syncthing-key
```

## Step 3: Create the status script

```bash
nvim ~/.config/waybar/scripts/syncthing-status.sh
```

Paste in:

```bash
#!/usr/bin/env bash
# Polls local Syncthing API, outputs single-line JSON for a Waybar custom module.
# Fails LOUD: if the key is missing or the API rejects/ignores us, it shows the
# error/offline icon instead of pretending everything is idle.

API="https://127.0.0.1:8384"
KEYFILE="$HOME/.config/waybar/scripts/.syncthing-key"

emit() {
  # emit <icon> <class> <tooltip>
  # Escapes backslashes, double-quotes, and newlines so the JSON is always valid.
  local text="$1" class="$2" tip="$3"
  tip="${tip//\\/\\\\}"   # backslash first
  tip="${tip//\"/\\\"}"   # then double-quotes
  tip="${tip//$'\n'/\\n}" # then real newlines -> literal \n
  printf '{"text":"%s","class":"%s","tooltip":"%s"}\n' "$text" "$class" "$tip"
}

# --- Key must exist and be readable, or we fail loud --------------------------
if [ ! -r "$KEYFILE" ]; then
  emit "󰅙" "syncthing-error" "Cannot read API key file"
  exit 0
fi
KEY="$(cat "$KEYFILE")"
if [ -z "$KEY" ]; then
  emit "󰅙" "syncthing-error" "API key file is empty"
  exit 0
fi
HDR="X-API-Key: $KEY"

# --- Connections call: capture body AND http status separately ---------------
resp="$(curl -sk -H "$HDR" -w '\n%{http_code}' "$API/rest/system/connections")"
code="$(printf '%s' "$resp" | tail -n1)"
conns="$(printf '%s' "$resp" | sed '$d')"

if [ -z "$code" ] || [ "$code" = "000" ]; then
  emit "󰅙" "syncthing-offline" "Syncthing not responding"
  exit 0
fi
if [ "$code" != "200" ]; then
  emit "󰅙" "syncthing-error" "API error (HTTP $code) - check key"
  exit 0
fi

config="$(curl -sk -H "$HDR" "$API/rest/config")"

# Peer count: connected devices minus our own local entry
peers="$(printf '%s' "$conns" | grep -c '"connected": true')"
peers=$((peers - 1))
[ "$peers" -lt 0 ] && peers=0

# Walk folders, check each state
folder_ids="$(printf '%s' "$config" | grep -oP '"id": "\K[^"]+')"
syncing=0
errored=0
tooltip="Peers connected: $peers"

while read -r fid; do
  [ -z "$fid" ] && continue
  st="$(curl -sk -H "$HDR" "$API/rest/db/status?folder=$fid")"
  state="$(printf '%s' "$st" | grep -oP '"state": "\K[^"]+' | head -n1)"
  [ -z "$state" ] && state="unknown"
  case "$state" in
    syncing) syncing=1 ;;
    error)   errored=1 ;;
  esac
  tooltip="$tooltip"$'\n'"$fid: $state"
done <<< "$folder_ids"

if [ "$errored" -eq 1 ]; then
  emit "󰅙" "syncthing-error" "$tooltip"
elif [ "$syncing" -eq 1 ]; then
  emit "󰓦" "syncthing-syncing" "$tooltip"
else
  emit "󰴽" "syncthing-idle" "$tooltip"
fi
```

Make it executable:

```bash
chmod +x ~/.config/waybar/scripts/syncthing-status.sh
```

## Step 4: Test before touching Waybar

```bash
~/.config/waybar/scripts/syncthing-status.sh | jq .
```

You want clean JSON, no parse error, with peers and folder state in the tooltip:

```json
{
  "text": "󰴽",
  "class": "syncthing-idle",
  "tooltip": "Peers connected: 1\nmyfolder-abc12: idle"
}
```

> [!note] The `\n` in `jq` output is normal
> `jq` prints the escaped newline literally on one line. In the real Waybar tooltip popup it renders as separate lines.

## Step 5: Add the module to Waybar

```bash
nvim ~/.config/waybar/config.jsonc
```

Two edits are required.

**5a. Add the definition** alongside your other `custom/` blocks:

```jsonc
"custom/syncthing": {
    "exec": "~/.config/waybar/scripts/syncthing-status.sh",
    "return-type": "json",
    "interval": 30,
    "tooltip": true,
    "on-click": "xdg-open https://127.0.0.1:8384"
}
```

**5b. Place it on the bar** by adding `"custom/syncthing"` to `modules-right`
(or left/center, wherever you want it):

```jsonc
"modules-center": ["clock", "tray", "custom/syncthing"],
```

> [!warning] Both edits are needed
> The definition alone does nothing. Without the entry in a `modules-*` array nothing appears on the bar.

## Step 6: Add the CSS

```bash
nvim ~/.config/waybar/style.css
```

Append to the end of the file. Uses only `@foreground` / `@background` theme variables, so it survives Omarchy theme switches:

```css
#custom-syncthing {
    margin-right: 17px;
    color: @foreground;
}
#custom-syncthing.syncthing-syncing { color: @foreground; }
#custom-syncthing.syncthing-error   { color: @foreground; background: @background; }
#custom-syncthing.syncthing-offline { color: @foreground; }
```

## Step 7: Reload Waybar

```bash
pkill waybar && waybar &
```

The icon should appear. Hover it to confirm peers and folder state on separate lines.



---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Permission denied` on key file | Used `chmod -600` (dash) | `chmod 600 ~/.config/waybar/scripts/.syncthing-key` |
| `Forbidden` from curl | Key in file does not match Syncthing's live key | Compare file to live key, re-save in Web UI (see below) |
| `config.xml` not in `~/.config/syncthing/` | Newer Syncthing path | `find ~ -name config.xml -path '*syncthing*' 2>/dev/null` then grep that path |
| Waybar: `Error parsing JSON ... value, object or array expected` | Raw newline inside the JSON string | Use the hardened script above (escapes via `emit`) |
| `jq: control characters ... must be escaped` | Same raw-newline problem | Same fix |
| Icon shows idle but sync looks dead | Old un-hardened script lying | Use the hardened script; it returns error/offline on failure |
| Icon does not appear at all | Missing entry in `modules-*` array | Add `"custom/syncthing"` to `modules-right` |

### Verify the key against Syncthing's source of truth

Syncthing stores the live key in its config file. Find it, then compare:

```bash
find ~ -name config.xml -path '*syncthing*' 2>/dev/null
grep -i apikey /path/from/find/config.xml
```

Compare that value to your key file. They must match exactly:

```bash
cat -A ~/.config/waybar/scripts/.syncthing-key
```

`cat -A` shows a trailing newline as `$`. You want the key followed by a single `$` and nothing else. If they differ, copy the value from `config.xml` (or regenerate and **Save** in the Web UI), then re-run the Step 4 test.

---

## Adjustables

- **Poll interval:** change `"interval": 30` in the module definition (seconds).
- **Icon position:** reorder `"custom/syncthing"` in the `modules-*` array.
- **Web UI port:** if not `8384`, update `API` in the script and the `on-click` URL.