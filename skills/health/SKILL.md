---
description: "Check system dependencies and environment status for Nazar. Use when user says 'health check', 'sistem kontrol', 'check dependencies', 'ortam kontrol', 'nazar health'."
---

# Nazar Health Check

Run ALL of the following checks in parallel using the Bash tool. Each check must be executed independently:

| # | Component | Command | Expected |
|---|-----------|---------|----------|
| 1 | Python version | `python3 --version` | Version >= 3.9 |
| 2 | Nazar CLI | `nazar --version 2>/dev/null || echo "NOT_FOUND"` | Prints a version string |
| 3 | iOS Simulator | `xcrun simctl list devices booted 2>/dev/null || echo "NOT_FOUND"` | At least one booted device listed |
| 4 | Android Emulator | `adb devices 2>/dev/null || echo "NOT_FOUND"` | At least one device listed (beyond the header line) |
| 5 | ffmpeg | `ffmpeg -version 2>/dev/null || echo "NOT_FOUND"` | Prints version info |
| 6 | Pillow | `python3 -c "from PIL import Image; print('OK')" 2>/dev/null || echo "NOT_FOUND"` | Prints OK |
| 7 | pywebview | `python3 -c "import webview; print('OK')" 2>/dev/null || echo "NOT_FOUND"` | Prints OK |
| 8 | Git | `git --version` | Prints version info |
| 9 | PyPI token | `test -f ~/.pypirc && echo "exists" || echo "NOT_FOUND"` | Prints "exists" |
| 10 | Port 9998 | `lsof -i :9998 2>/dev/null && echo "IN_USE" || echo "FREE"` | Prints "FREE" |
| 11 | Port 9999 | `lsof -i :9999 2>/dev/null && echo "IN_USE" || echo "FREE"` | Prints "FREE" |

## Evaluation Rules

For each check, determine the status:

- **OK** -- The command succeeded and output matches the expected result.
- **MISSING** -- The tool/package is not installed or not found.
- **WARNING** -- Partially available or unexpected state. Examples:
  - Python is installed but version < 3.9
  - A port is in use (not free)
  - iOS Simulator exists but no device is booted
  - Android `adb` exists but no device is connected

## Output Format

Present results as a markdown table with three columns:

```
| Component        | Status  | Result                          |
|------------------|---------|---------------------------------|
| Python version   | OK      | Python 3.11.5                   |
| Nazar CLI        | MISSING | nazar komutu bulunamadi         |
| ...              | ...     | ...                             |
```

After the table, print a summary line:

```
X/11 checks passed.
```

Where X is the number of checks with OK status.

## --fix Argument

If the user invoked the skill with `--fix` (check the user's message for "--fix"), append a "Fix Suggestions" section after the summary. For each MISSING or WARNING item, suggest the appropriate install command:

| Component | Fix Command |
|-----------|-------------|
| ffmpeg | `brew install ffmpeg` |
| Pillow | `pip install pillow` |
| pywebview | `pip install pywebview` |
| Nazar CLI | `pip install nazar` |
| Git | `brew install git` |
| Python | `brew install python@3.11` |
| PyPI token | Create `~/.pypirc` with your PyPI API token |
| iOS Simulator | Open Xcode and boot a simulator device |
| Android Emulator | Start an Android emulator via Android Studio or `emulator -avd <name>` |
| Port 9998 | `lsof -ti :9998 \| xargs kill -9` |
| Port 9999 | `lsof -ti :9999 \| xargs kill -9` |

Only list fix suggestions for items that are MISSING or WARNING. Do not list fixes for items that are OK.
