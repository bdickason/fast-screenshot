# fast-screenshot
Screenshots open instantly in Preview so you can copy, edit, then delete.

# Instant Screenshot → Preview Workflow (macOS)

## 1. Create screenshot folder

```bash
mkdir -p ~/screenshots
```

---

## 2. Set macOS screenshots to save there

```bash
defaults write com.apple.screencapture location ~/screenshots
killall SystemUIServer
```

---

## 3. Install fswatch

```bash
brew install fswatch
```

Find fswatch path:

```bash
which fswatch
```

Usually:

```text
/opt/homebrew/bin/fswatch
```

---

## 4. Create watcher script

File:

```text
~/.watch-screenshots.sh
```

Contents:

```bash
#!/bin/bash

SCREENSHOT_DIR="$HOME/screenshots"
LAST_OPENED=""
LAST_TIME=0

FSWATCH="/opt/homebrew/bin/fswatch"

"$FSWATCH" "$SCREENSHOT_DIR" | while IFS= read -r event
do
    filename=$(basename "$event")

    [[ "$filename" == .* ]] && continue
    [[ "$filename" != *.png ]] && continue

    now=$(date +%s)

    if [[ "$event" == "$LAST_OPENED" ]] && (( now - LAST_TIME < 2 )); then
        continue
    fi

    LAST_OPENED="$event"
    LAST_TIME=$now

    sleep 0.2

    if [ -f "$event" ]; then
        open -a Preview "$event"
    fi
done
```

Make executable:

```bash
chmod +x ~/.watch-screenshots.sh
```

---

## 6. Create LaunchAgent

File:

```text
~/Library/LaunchAgents/com.iolo.screenshotwatcher.plist
```

Contents:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
"http://www.apple.com/DTDs/PropertyList-1.0.dtd">

<plist version="1.0">
<dict>

    <key>Label</key>
    <string>com.iolo.screenshotwatcher</string>

    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/YOUR_USERNAME/.watch-screenshots.sh</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/tmp/screenshotwatcher.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/screenshotwatcher.err</string>

</dict>
</plist>
```

Replace:

```text
YOUR_USERNAME
```

with your mac username.

---

## 7. Load LaunchAgent

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.iolo.screenshotwatcher.plist
```

Verify:

```bash
launchctl list | grep screenshot
```

---

## 8. Reload after changes

Unload:

```bash
launchctl bootout gui/$(id -u)/com.iolo.screenshotwatcher
```

Reload:

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.iolo.screenshotwatcher.plist
```

Restart:

```bash
launchctl kickstart -k gui/$(id -u)/com.iolo.screenshotwatcher
```
