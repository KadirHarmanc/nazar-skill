---
description: "Run YAML test files on iOS simulator or Android emulator. Prerequisite: /nazar:health. Use when user says 'live test', 'canli test', 'simulator test', 'run test on device', 'emulator test', 'UI test'."
---

# Nazar Live Test

Run YAML-defined UI tests on simulator/emulator.

## Prerequisites
Run `/nazar:health` first to verify simulator/emulator is available.

## How to run

1. Find or accept YAML test file path (look in `.nazar/tests/` or user-specified)
2. Validate YAML syntax - must have: apiVersion, name, platform, steps
3. Detect platform:
   - iOS: check `xcrun simctl list devices booted` via Bash
   - Android: check `adb devices` via Bash
4. For each step, execute via Bash:
   - `launchApp`: iOS `xcrun simctl launch booted <bundle_id>`, Android `adb shell am start -n <package>/<activity>`
   - `tapOn`: iOS `xcrun simctl io booted tap <x> <y>`, Android `adb shell input tap <x> <y>`
   - `inputText`: iOS `xcrun simctl io booted input text "<value>"`, Android `adb shell input text "<value>"`
   - `assertVisible`: take screenshot and analyze for expected element (use Read on screenshot)
   - `screenshot`: iOS `xcrun simctl io booted screenshot <path>`, Android `adb shell screencap -p > <path>`
   - `scrollDown`: iOS simctl, Android `adb shell input swipe 500 1500 500 500`
   - `goBack`: Android `adb shell input keyevent KEYCODE_BACK`
   - `wait`: sleep for specified duration
5. Report: pass/fail per step, total pass rate, screenshots captured

## YAML Format

```yaml
apiVersion: nazar/v1
name: Test Name
platform: ios | android | all
priority: high | medium | low
steps:
  - action: launchApp
    target: "com.example.app"
  - action: tapOn
    target: "Login Button"
  - action: inputText
    target: "email_input"
    value: "test@example.com"
  - action: assertVisible
    target: "Welcome"
  - action: screenshot
    target: "login-success"
  - action: wait
    duration: 2
```
