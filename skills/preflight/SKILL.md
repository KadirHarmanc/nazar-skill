---
description: "Run App Store and Play Store pre-submission checks. Use when user says 'preflight', 'store check', 'app store kontrol', 'play store kontrol', 'yayinlamadan once kontrol', 'store rejection'."
---

# Nazar Preflight Skill

Run comprehensive App Store and Play Store compliance checks before submission.

## Instructions

### Step 1: Detect Platform

Scan the project directory to determine the platform:

**iOS indicators** (check for ANY of these):
- `*.xcodeproj` or `*.xcworkspace`
- `Podfile` or `Podfile.lock`
- `Info.plist`
- `*.swift` source files
- `Package.swift` (Swift Package Manager)
- `Cartfile` (Carthage)

**Android indicators** (check for ANY of these):
- `build.gradle` or `build.gradle.kts`
- `AndroidManifest.xml`
- `*.kt` or `*.java` source files in standard Android paths
- `settings.gradle` or `settings.gradle.kts`
- `gradlew`

**Cross-platform (both)** (check for ANY of these):
- React Native: `package.json` with `react-native` dependency + `ios/` and `android/` directories
- Flutter: `pubspec.yaml` with `flutter` dependency + `ios/` and `android/` directories
- Kotlin Multiplatform: `shared/` directory with `commonMain`
- Capacitor/Ionic: `capacitor.config.ts` or `capacitor.config.json`
- .NET MAUI: `*.csproj` with MAUI references
- Xamarin: `*.sln` with iOS and Android projects

### Step 2: Dispatch Preflight Agent

Call the `nazar-skill:preflight` agent with:
- **platform**: `ios`, `android`, or `both`
- **project_path**: the root directory of the project

The agent will:
1. Load the appropriate rule files from `~/.claude/plugins/nazar-skill/skills/preflight/rules/`
2. Check each rule against the project files
3. Return pass/fail results per rule

### Step 3: Present Results

Format the output as follows:

```
=== NAZAR PREFLIGHT REPORT ===

Platform: [iOS / Android / Both]
Project:  [project path]
Date:     [current date]

--- READINESS SCORE ---

iOS:     XX/22 passed (XX%)
Android: XX/12 passed (XX%)

--- BLOCKING ISSUES (must fix before submission) ---

[FAIL] Rule X: Rule Name
       File: /path/to/file.ext (line XX)
       Issue: Description of what is wrong
       Fix: How to fix it

[FAIL] Rule Y: Rule Name
       ...

--- WARNINGS (recommended to fix) ---

[WARN] Rule Z: Rule Name
       ...

--- PASSED ---

[PASS] Rule A: Rule Name
[PASS] Rule B: Rule Name
...

--- FIX SUGGESTIONS (prioritized) ---

1. [BLOCKING] Fix description (Rule X)
2. [BLOCKING] Fix description (Rule Y)
3. [RECOMMENDED] Fix description (Rule Z)
```

### Step 4: Categorize Issues

**BLOCKING** (will cause rejection):
- Missing privacy manifest (iOS)
- Missing permission descriptions (iOS)
- UIWebView usage (iOS)
- targetSdkVersion too low (Android)
- debuggable=true in release (Android)
- Missing exported attribute (Android)
- Missing account deletion (Android, if accounts exist)

**WARNINGS** (may cause rejection or poor review):
- Force unwrap abuse (iOS)
- Placeholder content (both)
- ProGuard not enabled (Android)
- Content rating considerations (Android)
- ATS blanket exceptions (iOS)
- Unencrypted storage (Android)

### Platform-Specific File Paths

**React Native projects**:
- iOS files: `ios/[ProjectName]/Info.plist`, `ios/Podfile`, `ios/[ProjectName].xcodeproj`
- Android files: `android/app/build.gradle`, `android/app/src/main/AndroidManifest.xml`

**Flutter projects**:
- iOS files: `ios/Runner/Info.plist`, `ios/Podfile`, `ios/Runner.xcodeproj`
- Android files: `android/app/build.gradle`, `android/app/src/main/AndroidManifest.xml`

**Native iOS**: files at project root or in app target directory
**Native Android**: files in `app/` directory typically
