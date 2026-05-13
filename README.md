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
FSWATCH="/opt/homebrew/bin/fswatch"

# Debounce state
LAST_EVENT_TIME=0
DEBOUNCE_MS=700

"$FSWATCH" "$SCREENSHOT_DIR" | while IFS= read -r event
do
    # Ignore non-png files
    [[ "$event" != *.png ]] && continue

    filename=$(basename "$event")

    # Ignore hidden/temp files
    [[ "$filename" == .* ]] && continue

    # Record latest event timestamp (milliseconds)
    LAST_EVENT_TIME=$(python3 -c 'import time; print(int(time.time() * 1000))')

    (
        CURRENT_EVENT_TIME=$LAST_EVENT_TIME

        # Wait for additional screenshot events to settle
        sleep 0.7

        # Only continue if no newer events arrived
        if [[ "$CURRENT_EVENT_TIME" == "$LAST_EVENT_TIME" ]]; then

            # Grab newest screenshots (supports multi-monitor screenshots)
            mapfile -t newest_files < <(
                ls -t "$SCREENSHOT_DIR"/*.png 2>/dev/null | head -5
            )

            if (( ${#newest_files[@]} > 0 )); then
                echo "OPENING ${#newest_files[@]} SCREENSHOT(S)"

                for file in "${newest_files[@]}"
                do
                    echo "  $file"
                done

                # Open all screenshots together without stealing focus repeatedly
                open -g -a Preview "${newest_files[@]}"
            fi
        fi
    ) &
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


# Cleanup script to remove old screenshots after a day
# Screenshot Cleanup Job (macOS)

## 1. Create cleanup script

File:

```text
~/.cleanup-screenshots.sh
```

Contents:

```bash
#!/bin/bash

find "$HOME/screenshots" -type f -mtime +1 -delete
```

Make executable:

```bash
chmod +x ~/.cleanup-screenshots.sh
```

---

## 2. Create LaunchAgent

File:

```text
~/Library/LaunchAgents/com.iolo.screenshotcleanup.plist
```

Contents:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
"http://www.apple.com/DTDs/PropertyList-1.0.dtd">

<plist version="1.0">
<dict>

    <key>Label</key>
    <string>com.iolo.screenshotcleanup</string>

    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/YOUR_USERNAME/.cleanup-screenshots.sh</string>
    </array>

    <!-- Run daily at 3:00 AM -->
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>3</integer>

        <key>Minute</key>
        <integer>0</integer>
    </dict>

</dict>
</plist>
```

Replace:

```text
YOUR_USERNAME
```

with your mac username.

---

## 3. Load LaunchAgent

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.iolo.screenshotcleanup.plist
```

Verify:

```bash
launchctl list | grep screenshotcleanup
```

---

## 4. Reload after changes

Unload:

```bash
launchctl bootout gui/$(id -u)/com.iolo.screenshotcleanup
```

Reload:

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.iolo.screenshotcleanup.plist
```
