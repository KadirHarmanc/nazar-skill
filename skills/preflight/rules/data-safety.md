# Android Data Safety Form Rules

Rules for auditing and preparing the Google Play Data Safety Form. The Data Safety section is required for all apps on the Play Store. These rules help pre-check what needs to be declared.

---

## Overview

The Data Safety Form requires developers to declare:
1. What data the app collects
2. Whether data is shared with third parties
3. How data is handled (encryption, deletion)
4. Security practices

The agent must scan the project to identify what data is collected and flag any discrepancies.

---

## Rule 1: Data Types Collected

- **What to Check**: `AndroidManifest.xml` (permissions), source code, third-party SDKs
- **What to Look For**: Identify ALL data types the app collects. Scan for:

### Location Data
- Grep for: `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`, `ACCESS_BACKGROUND_LOCATION`, `FusedLocationProviderClient`, `LocationManager`, `getLastKnownLocation`, `requestLocationUpdates`
- Declare: Approximate location and/or Precise location

### Personal Info
- Grep for: `READ_CONTACTS`, `ContactsContract`, `CNContactStore`, email fields, name fields, phone fields, address fields
- Declare: Name, Email address, Phone number, Address, or other personal info

### Financial Info
- Grep for: `BillingClient`, `Purchase`, credit card fields, payment processing, `Stripe`, `Braintree`, `PayPal`
- Declare: Purchase history, Credit/debit card info, Other financial info

### Health and Fitness
- Grep for: `HealthConnect`, `GoogleFit`, `SensorManager`, `TYPE_HEART_RATE`, `TYPE_STEP_COUNTER`, health-related data models
- Declare: Health info, Fitness info

### Messages
- Grep for: chat/messaging features, `MessageCompat`, SMS-related APIs, email sending
- Declare: Emails, SMS/MMS, Other in-app messages

### Photos and Videos
- Grep for: `READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO`, `MediaStore`, `CameraX`, `Camera2`, `ACTION_IMAGE_CAPTURE`, `ACTION_VIDEO_CAPTURE`, photo upload features
- Declare: Photos, Videos

### Audio Files
- Grep for: `READ_MEDIA_AUDIO`, `MediaRecorder`, `AudioRecord`, `RECORD_AUDIO`, voice recording features
- Declare: Voice/sound recordings, Music files, Other audio files

### Files and Docs
- Grep for: `ACTION_OPEN_DOCUMENT`, `DocumentsProvider`, file upload features, `READ_EXTERNAL_STORAGE`, `MANAGE_EXTERNAL_STORAGE`
- Declare: Files and docs

### Calendar
- Grep for: `READ_CALENDAR`, `WRITE_CALENDAR`, `CalendarContract`
- Declare: Calendar events

### App Activity
- Grep for: analytics SDK initialization (Firebase Analytics, Amplitude, Mixpanel, etc.), page view tracking, event logging, `logEvent`, `track(`, search functionality
- Declare: App interactions, Search history, Other user-generated content

### Web Browsing
- Grep for: `WebView`, `CustomTabsClient`, browser history tracking
- Declare: Web browsing history

### Device Identifiers
- Grep for: `Settings.Secure.ANDROID_ID`, `TelephonyManager`, `getDeviceId`, `IMEI`, `getLine1Number`, `Build.SERIAL`, advertising ID (`AdvertisingIdClient`)
- Declare: Device or other IDs

### Diagnostics
- Grep for: crash reporting SDKs (Sentry, Crashlytics, Bugsnag), performance monitoring, `FirebasePerformance`
- Declare: Crash logs, Performance data, Other diagnostics

- **Pass Condition**: All data types found in code scanning are documented for Data Safety Form completion.
- **Fail Reason**: Any undeclared data collection can lead to policy violation and app removal.

---

## Rule 2: Data Sharing Declarations

- **What to Check**: Third-party SDK integrations, network calls, API endpoints
- **What to Look For**: Data sent to third-party servers constitutes "sharing" under Google's definition.

### Common Data Sharing Scenarios:
- **Analytics SDKs**: Firebase Analytics, Google Analytics, Amplitude, Mixpanel, Segment -> shares app activity data
- **Advertising SDKs**: AdMob, Facebook Ads, Unity Ads, AppLovin -> shares device IDs, app activity
- **Crash Reporting**: Sentry, Crashlytics, Bugsnag -> shares diagnostics data
- **Attribution SDKs**: Adjust, AppsFlyer, Branch -> shares device IDs, install data
- **Social SDKs**: Facebook Login, Google Sign-In -> shares profile data
- **Maps SDKs**: Google Maps, Mapbox -> shares location data
- **Push SDKs**: OneSignal, Firebase Cloud Messaging -> shares device tokens

### Validation Steps:
1. Scan `build.gradle` dependencies for known SDK packages
2. Grep source code for SDK initialization calls
3. For each SDK found, note what data it collects/shares
4. Flag any SDK that shares data without user consent mechanism

- **Pass Condition**: All third-party data sharing is identified and documented.
- **Fail Reason**: Undeclared data sharing is the most common Data Safety violation.

---

## Rule 3: Data Handling - Encryption in Transit

- **What to Check**: Network configuration, API calls, `network_security_config.xml`
- **What to Look For**:
  - `android:networkSecurityConfig` in AndroidManifest.xml
  - `cleartextTrafficPermitted` in network security config
  - `http://` URLs in source code (non-HTTPS)
  - `android:usesCleartextTraffic="true"` in manifest
  - OkHttp/Retrofit base URLs

- **Pass Condition**: All network traffic uses HTTPS. No cleartext HTTP traffic permitted (except localhost for development). `network_security_config.xml` does not allow cleartext for production domains.
- **Fail Reason**: Unencrypted data transmission must be declared in Data Safety Form as "not encrypted in transit". This significantly impacts user trust and store listing appearance.
- **Fix Guide**:
  1. Use HTTPS for all API endpoints
  2. Set `android:usesCleartextTraffic="false"` in manifest
  3. Configure `network_security_config.xml` to only allow cleartext for debug builds:
  ```xml
  <network-security-config>
      <base-config cleartextTrafficPermitted="false">
          <trust-anchors>
              <certificates src="system" />
          </trust-anchors>
      </base-config>
      <debug-overrides>
          <trust-anchors>
              <certificates src="user" />
          </trust-anchors>
      </debug-overrides>
  </network-security-config>
  ```

---

## Rule 4: Encryption at Rest

- **What to Check**: Source code for local data storage patterns
- **What to Look For**:
  - `SharedPreferences` storing sensitive data (tokens, passwords, PII) -> should use `EncryptedSharedPreferences`
  - `SQLiteDatabase` with sensitive data -> should use SQLCipher or similar
  - `Room` database with sensitive data -> check for encryption
  - File storage of sensitive data -> check for encryption
  - `getExternalFilesDir` / `getExternalStorageDirectory` for sensitive data (BAD - world readable on older Android)

### Sensitive Data Patterns to Grep:
  - `token`, `access_token`, `refresh_token`, `auth_token`, `api_key`, `secret`, `password`, `credential`
  - `ssn`, `social_security`, `credit_card`, `card_number`
  - Personal identifiers being stored locally

- **Pass Condition**: Sensitive data stored locally uses encryption (EncryptedSharedPreferences, SQLCipher, or Android Keystore). No sensitive data in plain SharedPreferences or plain files.
- **Fail Reason**: Unencrypted sensitive data at rest must be declared in Data Safety Form. Also a security audit failure.
- **Fix Guide**:
  1. Replace `SharedPreferences` with `EncryptedSharedPreferences` for sensitive data:
  ```kotlin
  val masterKey = MasterKey.Builder(context)
      .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
      .build()
  val sharedPreferences = EncryptedSharedPreferences.create(
      context, "secret_prefs", masterKey,
      EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
      EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
  )
  ```
  2. Use Android Keystore for cryptographic keys
  3. Never store sensitive data on external storage

---

## Rule 5: Account Deletion Capability

- **What to Check**: Source code, settings screens, API endpoints
- **What to Look For**: If the app supports account creation (sign up, registration), it MUST also provide account deletion capability. Check for:
  - Registration/sign-up flows in code
  - Login functionality (Firebase Auth, custom auth, OAuth)
  - Account settings or profile screens
  - Delete account API endpoint or UI button

- **Pass Condition**:
  - If the app has no user accounts (no login/signup), this rule passes automatically.
  - If the app has user accounts: a delete account option must be accessible within the app (not just via email/website). Must delete the account AND associated data.
- **Fail Reason**: Google Play requires apps with accounts to offer in-app account deletion (enforced since December 2023). Missing this feature causes rejection.
- **Fix Guide**:
  1. Add a "Delete Account" option in app settings/profile
  2. Implement API endpoint to delete user data server-side
  3. Clear local data upon deletion:
  ```kotlin
  // Delete account flow
  fun deleteAccount() {
      // 1. Call server API to delete account and data
      apiService.deleteAccount(userId)
      // 2. Clear local data
      sharedPreferences.edit().clear().apply()
      database.clearAllTables()
      // 3. Sign out and navigate to login
      FirebaseAuth.getInstance().signOut()
      navigateToLogin()
  }
  ```
  4. Show confirmation dialog before deletion
  5. If using Firebase Auth: `FirebaseAuth.getInstance().currentUser?.delete()`
  6. Ensure deletion is completed within a reasonable time (Google's guideline: within 60 days max, ideally immediate)
