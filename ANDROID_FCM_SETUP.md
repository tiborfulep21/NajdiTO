# Android FCM Setup for OneSignal Push Notifications

> **TL;DR — why no Android device is registered today**
>
> The Lovable cloud sandbox only contains the JS / web side of the app. The
> `android/` native project is generated **on your own machine** by
> `npx cap add android`. The Capacitor + OneSignal native plugin works, but
> Android cannot register an FCM token until **three native build files** are
> wired correctly. None of these can be edited from Lovable — you must do them
> locally and rebuild the APK.
>
> Until that's done, the only "subscription" OneSignal sees for your account is
> the Chrome web push one — that's why every user shows up as **Chrome (Web)**.

---

## ✅ Step 0 — Sync the latest Lovable code

After every change here, in your local clone run:

```bash
git pull
npm install
npm run build
npx cap sync android
```

`cap sync` copies the OneSignal Cordova plugin into `android/`.

## ✅ Step 1 — Add `google-services.json`

1. Go to **Firebase Console → Project Settings → Your apps → Android app**.
   - **Package name MUST exactly match** `capacitor.config.ts` `appId`:
     `app.lovable.355a02350f184485ab8f8b10fe3b1081`
   - If your Firebase Android app uses a different package, click **Add app**
     and create a new Android app with the correct package name.
2. Download `google-services.json`.
3. Place it at:

   ```
   android/app/google-services.json
   ```

   ⚠ Not in `android/`, **inside** `android/app/`.

## ✅ Step 2 — Wire the Google Services Gradle plugin

### `android/build.gradle` (project-level)

In the top-level `buildscript { dependencies { ... } }` block, add:

```gradle
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:8.2.0'
        classpath 'com.google.gms:google-services:4.4.2'
    }
}
```

### `android/app/build.gradle` (app-level)

At the **very bottom** of the file, add:

```gradle
apply plugin: 'com.google.gms.google-services'
```

And inside the `dependencies { }` block, add the FCM library:

```gradle
dependencies {
    implementation fileTree(include: ['*.jar'], dir: 'libs')
    implementation "androidx.appcompat:appcompat:1.6.1"
    implementation "androidx.coordinatorlayout:coordinatorlayout:1.2.0"
    implementation "androidx.core:core-splashscreen:1.0.1"
    implementation project(':capacitor-android')
    implementation 'com.google.firebase:firebase-messaging:24.0.0'
}
```

## ✅ Step 3 — Verify the OneSignal Cordova plugin is wired

`onesignal-cordova-plugin` is already in `package.json`. After `npx cap sync android`,
check that this exists:

```
android/capacitor-cordova-android-plugins/src/main/java/com/onesignal/...
```

If missing, run:

```bash
npm install onesignal-cordova-plugin@latest
npx cap sync android
```

## ✅ Step 4 — Configure OneSignal dashboard

In **OneSignal Dashboard → Settings → Push & In-App → Google Android (FCM)**:

1. Upload the **Firebase Service Account JSON** (FCM v1 — recommended), OR the
   legacy **Server Key** if your project still uses it.
2. Use the **same Firebase project** as the `google-services.json` you placed
   in step 1. Mismatched projects = silent failure.

## ✅ Step 5 — Rebuild & install

```bash
npx cap sync android
npx cap open android       # opens Android Studio
# In Android Studio: Build → Clean Project, then Build → Rebuild Project
# Then run the app on a real device (FCM does NOT work on most emulators
#   without Google Play Services).
```

> Use a real Android device or an emulator image labelled
> **"Google Play"** (not just "Google APIs").

## 🔍 Step 6 — Verify on the device

1. Grant the notification permission when prompted.
2. With the device connected via USB, open Chrome on your laptop and go to
   `chrome://inspect/#devices`. Inspect the WebView for the app.
3. In the console, you should now see:

   ```
   [OneSignal-Native] ▶ STEP 1: importing onesignal-cordova-plugin { platform: 'android' }
   [OneSignal-Native] ▶ STEP 2: plugin found on window
   [OneSignal-Native] ▶ STEP 3: initialized { ... platform: 'android' }
   [OneSignal-Native] ⏳ Waiting for FCM subscription id…
   [OneSignal-Native] ▶ Subscription id received after N attempts { id: 'xxxxx-...' }
   [OneSignal-Native] ▶ PushSubscription changed (FCM register event)
       { subscriptionId: '...', optedIn: true, fcmTokenPresent: true, fcmTokenPreview: 'eXp...' }
   [OneSignal-Native] FINAL OK { userId: '...', subId: '...', platform: 'android' }
   ```

4. Run this in the WebView console for a live status snapshot:

   ```js
   await __onesignalNativeStatus()
   ```

5. In **OneSignal Dashboard → Audience → All Users**, your device must now
   appear with **Device Type: Android** (not Chrome).

## ⚠ Common failure modes

| Symptom in console | Root cause |
| --- | --- |
| `Plugin not found on window` | Forgot `npx cap sync android` after install. |
| `initialize() failed` | `google-services.json` missing, or wrong package name. |
| `Timed out waiting for FCM subscription id…` | Google Services Gradle plugin not applied, OR firebase-messaging dep missing, OR running on emulator without Google Play Services. |
| Device appears in OneSignal as **Chrome (Web)** only | App was previously opened in browser before being installed; the Web SDK created a phantom subscription. **Already fixed in `index.html`** — the Web SDK no longer loads inside Capacitor. Reinstall the app. |
| FCM token generated but pushes never deliver | OneSignal Dashboard FCM credentials don't match `google-services.json` Firebase project. |

## What was fixed in code (already done)

- **`index.html`** — OneSignal Web SDK is **no longer loaded** when running
  inside Capacitor. This prevents the phantom Chrome subscription that was
  hijacking your account on the device.
- **`src/lib/onesignal-native.ts`** — Added verbose, step-by-step logging of
  the initialization, FCM-token wait, and subscription change events so you
  can pinpoint exactly which step fails. Added `Debug.setLogLevel(6)` so
  OneSignal native logs also appear in `adb logcat`. Exposed
  `window.__onesignalNativeStatus()` for live diagnostics.

After completing steps 1-5 above and reinstalling, FCM tokens will be
generated and your devices will register as Android in OneSignal.
