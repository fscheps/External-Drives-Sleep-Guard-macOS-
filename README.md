# External Drives Sleep Guard (macOS)

Stop the dreaded **“Disk Not Ejected Properly”** when you close your Mac’s lid **on battery** with external drives attached.
On **AC power**, drives stay mounted so long jobs (LLMs, copies, servers) keep running.

[![macOS](https://img.shields.io/badge/macOS-12%2B-blue)](#) [![Shell-zsh](https://img.shields.io/badge/Shell-zsh-333)](#) [![License-MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## What it does

* Clean **unmount + eject (whole disk)** right before battery sleep
* **Auto‑remount by Volume UUID** after wake (with a retry window)
* Targets **Samsung T7 only** by default; can switch to **all removable** or **specific serials**
* Works on Apple Silicon and Intel

It uses the tiny background agent **SleepWatcher** to run two user hooks: `~/.sleep` (before sleep) and `~/.wakeup` (after wake).

---

## Quick install (one command)

This one‑liner installs SleepWatcher, writes the helper at `~/.local/bin/sleep-guard.sh`, creates the hooks, starts the service, and runs a self‑test. **Default mode:** Samsung **T7 only**. You can change it later.

```zsh
if ! command -v brew >/dev/null 2>&1; then echo "Install Homebrew first: https://brew.sh"; exit 1; fi
brew list sleepwatcher >/dev/null 2>&1 || brew install sleepwatcher
mkdir -p ~/.local/bin ~/.cache/extdisk-park
cat > ~/.local/bin/sleep-guard.sh <<'SH'
#!/usr/bin/env zsh
# External Drives Sleep Guard — eject before battery sleep, remount on wake.
set -euo pipefail
log="$HOME/Library/Logs/sleep-guard.log"      # event log
state="$HOME/.cache/extdisk-park/uuids"       # cached Volume UUIDs for remount
# ---- Config knobs (edit later if needed) ----
HOLD_SECONDS=12      # hold system awake this long before sleep (finish unmount/eject)
WAKE_RETRY=30        # seconds to keep retrying remount on wake
MODE="t7"            # "t7" (Samsung T7 only) | "all" (all external removable)
SERIALS=()           # optional: ("SERIAL1" "SERIAL2") to target specific devices
# ---------------------------------------------
_ts(){ date "+%Y-%m-%d %H:%M:%S"; }
_onbat(){ pmset -g ps | grep -q "Battery Power"; }
# Discover WHOLE disks to manage (e.g., /dev/disk6 /dev/disk7)
_targets(){
  /usr/sbin/diskutil list external | /usr/bin/awk '/^\/dev\/disk[0-9]+ \(external/{print $1}' | while read -r dev; do
    p=$(/usr/sbin/diskutil info -plist "$dev") || continue
    name=$(echo "$p" | /usr/bin/plutil -extract MediaName   raw -o - - 2>/dev/null || :)
    proto=$(echo "$p" | /usr/bin/plutil -extract BusProtocol raw -o - - 2>/dev/null || :)
    ser=$(  echo "$p" | /usr/bin/plutil -extract SerialNumber raw -o - - 2>/dev/null || :)
    [[ "$MODE" = "t7" && ( "$name" != *T7* || "$proto" != *USB* ) ]] && continue
    if (( ${#SERIALS[@]} )); then
      keep=false; for s in "${SERIALS[@]}"; do [[ "$ser" == *"$s"* ]] && keep=true; done; $keep || continue
    fi
    echo "$dev"
  done
}
# Cache Volume UUIDs that live on those targets (so we can remount by UUID later)
_cache(){
  tmp="$state.tmp"; :>"$tmp"
  local t; t=($(_targets))
  for v in /Volumes/*; do
    info=$(/usr/sbin/diskutil info "$v" 2>/dev/null || :); [[ -z "$info" ]] && continue
    [[ "$(echo "$info" | /usr/bin/awk -F': *' '/Device Location/{print $2}')" != External ]] && continue
    uuid=$(echo "$info" | /usr/bin/awk -F': *' '/Volume UUID/{print $2}')
    leaf=$(echo "$info" | /usr/bin/awk -F': *' '/Device Node/{print $2}')
    [[ -z "$uuid" || -z "$leaf" ]] && continue
    whole=$(/usr/sbin/diskutil info -plist "$leaf" | /usr/bin/plutil -extract ParentWholeDisk raw -o - - 2>/dev/null || :)
    [[ -z "$whole" ]] && whole=$(echo "$leaf" | /usr/bin/sed -E 's#(disk[0-9]+)s[0-9]+#\1#')
    for d in "${t[@]}"; do [[ "/dev/$whole" == "$d" ]] && echo "$uuid" >>"$tmp"; done
  done
  if [[ -s "$tmp" ]]; then
    sort -u "$tmp" -o "$state"
    echo "$(_ts) register: $(tr '\n' ' ' <"$state")" >>"$log"
  else
    echo "$(_ts) register: no targets; keep previous" >>"$log"
  fi
  rm -f "$tmp"
}
_uuids(){ [[ -s "$state" ]] || _cache; cat "$state"; }
_do_sleep(){
  if ! _onbat; then echo "$(_ts) sleep: AC -> leave mounted" >>"$log"; return 0; fi
  echo "$(_ts) sleep: battery -> unmount+eject" >>"$log"
  /usr/bin/caffeinate -disu -t "$HOLD_SECONDS" &>/dev/null &  # hold sleep briefly
  /bin/sync                                                   # flush writes
  for dev in $(_targets); do
    /usr/sbin/diskutil unmountDisk force "$dev" &>/dev/null || true
    /usr/sbin/diskutil eject "$dev"            &>/dev/null || true
    echo "$(_ts) sleep: processed $dev" >>"$log"
  done
}
_do_wake(){
  echo "$(_ts) wake: remount" >>"$log"
  /bin/sleep 2
  for u in $(_uuids); do
    for i in $(/usr/bin/jot - 1 "$WAKE_RETRY"); do
      /usr/sbin/diskutil mount "volumeUUID:$u" &>/dev/null && { echo "$(_ts) wake: mounted $u" >>"$log"; break; }
      /bin/sleep 1
    done
  done
}
case "${1:-}" in
  register) _cache ;;
  sleep)    _do_sleep ;;
  wake)     _do_wake ;;
  test)     _cache; _do_sleep; /bin/sleep 3; _do_wake ;;
  *) echo "usage: $0 {register|sleep|wake|test}"; exit 1 ;;
esac
SH
chmod +x ~/.local/bin/sleep-guard.sh
printf '%s\n' '$HOME/.local/bin/sleep-guard.sh sleep' > ~/.sleep
printf '%s\n' '$HOME/.local/bin/sleep-guard.sh wake'  > ~/.wakeup
chmod +x ~/.sleep ~/.wakeup
brew services restart sleepwatcher
~/.local/bin/sleep-guard.sh test
echo "Log: $HOME/Library/Logs/sleep-guard.log"
```

**Check the log:**

```zsh
tail -n 60 "$HOME/Library/Logs/sleep-guard.log"
```

You should see `sleep: processed /dev/disk…` followed by `wake: mounted …`.

---

## Configure what gets managed

### Option A — All removable drives

Switch from T7‑only to all external removable disks.

```zsh
nano ~/.local/bin/sleep-guard.sh   # or your editor
# set: MODE="all"
brew services restart sleepwatcher
~/.local/bin/sleep-guard.sh register
```

### Option B — Specific devices by serial

Pin the behavior to one or more device serials.

```zsh
# Find serials for external disks (not all bridges expose one)
for d in $(diskutil list external | awk '/^\/dev\//{print $1}'); do
  echo "== $d =="; diskutil info -plist "$d" | \
    plutil -extract SerialNumber raw -o - - 2>/dev/null || echo "(no serial field)"
done
```

Then set inside the script and restart:

```zsh
SERIALS=("S6QJNA0R123456" "S6QJNA0R654321")
```

---

## Verify it works

1. Unplug AC so you’re on **battery**
2. Close the lid for ~15–20 s
3. Reopen and check:

```zsh
tail -n 120 "$HOME/Library/Logs/sleep-guard.log"
```

Expected: `sleep: processed /dev/diskX` then `wake: mounted <UUID>` and **no** macOS eject toast.

---

## Tuning & tips

* `HOLD_SECONDS` → increase if your Mac falls asleep too quickly (try `18`)
* `WAKE_RETRY` → increase if USB devices re‑enumerate slowly after wake (try `45`)
* Reduce background churn on target volumes:

```zsh
sudo mdutil -i off /Volumes/<YourVolumeName>
```

* System Settings → Privacy & Security → **Allow accessories to connect** → **Always**
* Advanced (optional) if deep hibernate races the hook on battery:

```zsh
sudo pmset -b hibernatemode 3
# revert later:
# sudo pmset -b hibernatemode 25
```

---

## Uninstall

Stops the agent and removes the helper + cache. Your disks/data are untouched.

```zsh
brew services stop sleepwatcher
rm -f ~/.sleep ~/.wakeup ~/.local/bin/sleep-guard.sh ~/.cache/extdisk-park/uuids
```

---

## FAQ

**Does this format or change my disks?**
No. It only unmounts/ejects and remounts.

**Thunderbolt enclosures?**
Yes. Set `MODE="all"`.

**Do I need Homebrew?**
It’s the easiest way to install SleepWatcher. You can install SleepWatcher manually and keep the same script.

**Where are logs?**
`~/Library/Logs/sleep-guard.log`

---

## License

MIT — see [LICENSE](LICENSE).
