# Google Play Store Submission Rules

12 rules that Google reviews during Play Store submission. Each rule includes what to check, where to check, pass/fail conditions, and fix guidance.

---

## Rule 1: Target SDK Version

- **Rule Name**: targetSdkVersion Compliance
- **What File to Check**: `build.gradle` or `build.gradle.kts` (app-level), `AndroidManifest.xml`
- **What to Look For**: `targetSdkVersion` or `targetSdk` value in the `android > defaultConfig` block.
- **Pass Condition**: `targetSdkVersion >= 34` (Android 14). Google Play requires new apps and updates to target the latest major API level.
- **Fail Reason**: Apps targeting below the required SDK version are rejected from the Play Store. As of 2024, the requirement is API 34+.
- **Fix Guide**: Update `build.gradle`:
  ```groovy
  android {
      defaultConfig {
          targetSdkVersion 34  // or higher
      }
  }
  ```
  Test the app thoroughly against Android 14 behavior changes after updating.

---

## Rule 2: Data Safety Form Declarations

- **Rule Name**: Data Safety Declarations
- **What File to Check**: `AndroidManifest.xml` (permissions), source code (data collection patterns), `build.gradle` (SDKs)
- **What to Look For**: Identify all data the app collects, shares, or processes. Check for:
  - Location permissions (`ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`)
  - Contact access (`READ_CONTACTS`)
  - Camera/microphone usage
  - Network SDKs (analytics, advertising, crash reporting)
  - Storage access
  - Account/profile data handling
- **Pass Condition**: All data collection patterns found in the app code are documented (this is a pre-check reminder; the actual form is filled in Play Console). The check identifies what needs to be declared.
- **Fail Reason**: Inaccurate Data Safety declarations cause rejection or removal. If the app collects data not declared in the form, this is a policy violation.
- **Fix Guide**: Audit all permissions and SDK data flows. Document every data type collected. Fill the Data Safety Form in Play Console accurately. See `data-safety.md` for the full checklist.

---

## Rule 3: QUERY_ALL_PACKAGES Permission

- **Rule Name**: Package Visibility Restriction
- **What File to Check**: `AndroidManifest.xml`
- **What to Look For**: `<uses-permission android:name="android.permission.QUERY_ALL_PACKAGES" />`
- **Pass Condition**: `QUERY_ALL_PACKAGES` is NOT present. OR if present, the app is a launcher, accessibility tool, security app, device management, or file manager that legitimately needs it.
- **Fail Reason**: Google restricts `QUERY_ALL_PACKAGES` to specific app categories. Using it without qualification causes rejection.
- **Fix Guide**: Remove `QUERY_ALL_PACKAGES` and use `<queries>` element instead to declare specific packages or intents:
  ```xml
  <queries>
      <package android:name="com.specific.app" />
      <intent>
          <action android:name="android.intent.action.VIEW" />
          <data android:scheme="https" />
      </intent>
  </queries>
  ```

---

## Rule 4: Granular Media Permissions (Android 13+)

- **Rule Name**: Granular Permissions for Android 13+
- **What File to Check**: `AndroidManifest.xml`, source code
- **What to Look For**: Usage of deprecated broad permissions vs granular ones:
  - `READ_EXTERNAL_STORAGE` should be replaced with:
    - `READ_MEDIA_IMAGES` (for photos)
    - `READ_MEDIA_VIDEO` (for videos)
    - `READ_MEDIA_AUDIO` (for audio files)
  - `WRITE_EXTERNAL_STORAGE` is no longer needed for Android 10+ (scoped storage)
- **Pass Condition**: If targeting Android 13+ (API 33+), uses granular `READ_MEDIA_*` permissions instead of `READ_EXTERNAL_STORAGE`. May keep both with `maxSdkVersion` for backward compatibility.
- **Fail Reason**: Using broad storage permissions when targeting Android 13+ triggers policy warnings and may cause rejection.
- **Fix Guide**: Replace broad permissions:
  ```xml
  <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" android:maxSdkVersion="32" />
  <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
  <uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
  ```

---

## Rule 5: POST_NOTIFICATIONS Permission (Android 13+)

- **Rule Name**: Notification Permission Declaration
- **What File to Check**: `AndroidManifest.xml`, source code for notification usage
- **What to Look For**: If the app sends notifications (uses `NotificationManager`, `NotificationCompat`, Firebase Messaging, etc.), it must declare `POST_NOTIFICATIONS` permission and request it at runtime for Android 13+.
- **Pass Condition**: If notifications are used: `POST_NOTIFICATIONS` is declared in manifest AND runtime permission request exists in code. If no notifications, passes automatically.
- **Fail Reason**: On Android 13+, notifications are blocked by default without the permission. Not requesting it means users never see notifications, which is a functional issue that can cause rejection.
- **Fix Guide**: Add to `AndroidManifest.xml`:
  ```xml
  <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
  ```
  Request at runtime:
  ```kotlin
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
      requestPermissions(arrayOf(Manifest.permission.POST_NOTIFICATIONS), REQUEST_CODE)
  }
  ```

---

## Rule 6: Foreground Service Type

- **Rule Name**: Foreground Service Type Declaration
- **What File to Check**: `AndroidManifest.xml`, source code using `startForegroundService`
- **What to Look For**: All `<service>` elements using `startForeground()` must have `android:foregroundServiceType` declared. Valid types:
  - `camera`, `connectedDevice`, `dataSync`, `health`, `location`, `mediaPlayback`, `mediaProjection`, `microphone`, `phoneCall`, `remoteMessaging`, `shortService`, `specialUse`, `systemExempted`
- **Pass Condition**: Every foreground service has `android:foregroundServiceType` attribute. Corresponding permissions are declared (e.g., `FOREGROUND_SERVICE_LOCATION` for location type).
- **Fail Reason**: Android 14 (API 34) requires foreground service types. Missing type causes crash on Android 14+ and rejection.
- **Fix Guide**: Add type to each foreground service in manifest:
  ```xml
  <service
      android:name=".MyService"
      android:foregroundServiceType="location"
      android:exported="false" />
  ```
  Also add the corresponding permission: `<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />`

---

## Rule 7: Debug Mode Disabled in Release

- **Rule Name**: Release Build Security
- **What File to Check**: `AndroidManifest.xml`, `build.gradle` / `build.gradle.kts`
- **What to Look For**:
  - `android:debuggable="true"` in `AndroidManifest.xml` (must NOT be present or must be `false`)
  - `debuggable true` in release buildType in `build.gradle`
- **Pass Condition**: `debuggable` is NOT set to `true` in the release configuration. Ideally, it is not set at all (defaults to `false` for release).
- **Fail Reason**: Debuggable release builds are a security risk and violate Play Store policies. Exposes the app to reverse engineering and data theft.
- **Fix Guide**: Remove `android:debuggable="true"` from `AndroidManifest.xml`. Ensure `build.gradle` release block does not set `debuggable true`:
  ```groovy
  buildTypes {
      release {
          debuggable false
          // ...
      }
  }
  ```

---

## Rule 8: Backup Configuration

- **Rule Name**: Backup Configuration
- **What File to Check**: `AndroidManifest.xml`
- **What to Look For**: `android:allowBackup` attribute on `<application>` element. For API 31+, also check `android:dataExtractionRules` and `android:fullBackupContent`.
- **Pass Condition**: `android:allowBackup` is explicitly set (either `true` with proper backup rules, or `false`). For API 31+, `android:dataExtractionRules` points to a valid XML file that excludes sensitive data.
- **Fail Reason**: Default backup behavior may back up sensitive data (tokens, credentials). Not configuring backup rules is a security finding.
- **Fix Guide**: Add backup configuration:
  ```xml
  <application
      android:allowBackup="true"
      android:fullBackupContent="@xml/backup_rules"
      android:dataExtractionRules="@xml/data_extraction_rules">
  ```
  Create `res/xml/backup_rules.xml` excluding sensitive files.

---

## Rule 9: Exported Components Security

- **Rule Name**: Exported Components
- **What File to Check**: `AndroidManifest.xml`
- **What to Look For**: All `<activity>`, `<service>`, `<receiver>`, `<provider>` with `<intent-filter>` are implicitly exported. For API 31+, components with intent-filters MUST explicitly declare `android:exported="true"` or `android:exported="false"`.
- **Pass Condition**: Every component with an `<intent-filter>` has an explicit `android:exported` attribute. Non-public components use `android:exported="false"` or `android:permission` for protection.
- **Fail Reason**: Missing `android:exported` attribute causes build failure on API 31+. Over-exposed components are a security risk flagged in review.
- **Fix Guide**: Add `android:exported` to every component with intent-filters:
  ```xml
  <activity
      android:name=".MainActivity"
      android:exported="true">
      <intent-filter>...</intent-filter>
  </activity>
  ```
  Set `exported="false"` for internal components. Use `android:permission` for components that need restricted access.

---

## Rule 10: ProGuard/R8 Enabled for Release

- **Rule Name**: Code Shrinking and Obfuscation
- **What File to Check**: `build.gradle` or `build.gradle.kts` (app-level)
- **What to Look For**: In the `release` buildType: `minifyEnabled true` and `shrinkResources true`. Also check for `proguard-rules.pro` or `proguard-android-optimize.txt` usage.
- **Pass Condition**: `minifyEnabled` is `true` for release builds. ProGuard/R8 rules file exists and is referenced.
- **Fail Reason**: Without code shrinking, the APK/AAB is larger, contains all debug symbols, and is easier to reverse-engineer. While not an automatic rejection, Google strongly recommends it and flags it.
- **Fix Guide**: Enable in `build.gradle`:
  ```groovy
  buildTypes {
      release {
          minifyEnabled true
          shrinkResources true
          proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
      }
  }
  ```

---

## Rule 11: Release Signing Configuration

- **Rule Name**: Release Signing Config
- **What File to Check**: `build.gradle` or `build.gradle.kts`, `signingConfigs` block
- **What to Look For**: Release buildType must use a signing configuration that references a release keystore (not debug keystore). Check:
  - `signingConfig signingConfigs.release` in release buildType
  - Keystore file path does not point to `debug.keystore`
  - Keystore password is not hardcoded in plain text (should use environment variables or `local.properties`)
- **Pass Condition**: Release build uses a non-debug signing config. Keystore credentials are not hardcoded in version-controlled files.
- **Fail Reason**: Debug-signed apps are rejected. Hardcoded credentials in `build.gradle` are a security risk.
- **Fix Guide**: Configure release signing:
  ```groovy
  signingConfigs {
      release {
          storeFile file(RELEASE_STORE_FILE)
          storePassword RELEASE_STORE_PASSWORD
          keyAlias RELEASE_KEY_ALIAS
          keyPassword RELEASE_KEY_PASSWORD
      }
  }
  ```
  Store credentials in `local.properties` (gitignored) or environment variables.

---

## Rule 12: Content Rating Considerations

- **Rule Name**: Content Rating Compliance
- **What File to Check**: Source code, assets, strings, web URLs loaded
- **What to Look For**: Content that affects the IARC content rating questionnaire:
  - Violence or graphic content in assets/strings
  - User-generated content features (chat, forums, photo sharing)
  - Location sharing with other users
  - In-app purchases or gambling mechanics
  - Social features or unmoderated content
  - External links or web content loading
  - Alcohol/tobacco/drug references
- **Pass Condition**: Content types are identified for accurate questionnaire completion. No undeclared mature content found.
- **Fail Reason**: Incorrect content rating (rating the app lower than actual content) causes removal from the store.
- **Fix Guide**: Audit all content visible to users. Answer the IARC questionnaire honestly in Play Console. If the app has user-generated content, declare it and implement moderation.
