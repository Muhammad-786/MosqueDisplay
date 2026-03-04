# Faizan-E-Madinah Display  Complete Project Documentation

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Website  Features and Architecture](#2-website--features-and-architecture)
3. [Firebase Setup](#3-firebase-setup)
4. [Admin Panel Guide](#4-admin-panel-guide)
5. [Fire TV Browser Fix](#5-fire-tv-browser-fix)
6. [Android APK Project  Full Setup](#6-android-apk-project--full-setup)
7. [GitHub Actions  Automated APK Build](#7-github-actions--automated-apk-build)
8. [All Crash Fixes and Hardening  Chronological Log](#8-all-crash-fixes-and-hardening--chronological-log)
9. [Final Architecture Summary](#9-final-architecture-summary)
10. [Reusable Prompt for Similar Projects](#10-reusable-prompt-for-similar-projects)

---

## 1. Project Overview

**Faizan-E-Madinah Display** is a digital signage system designed to run 24/7 on a TV screen (Amazon Fire Stick) inside a mosque. It shows:

- Live clock and date
- Jummah prayer schedule (Speech, Khutbah, Jamaat times)
- Automatic Friday countdown / "Jummah Mubarak" overlay
- An image/text slideshow managed from a cloud admin panel
- Real-time updates: any admin change instantly reflects on the display with no page refresh

**Technology stack:**
- Single-file front-end: `index.html` (HTML + CSS + JavaScript)
- Backend: Google Firebase (Firestore database + Email/Password Auth)
- Android app wrapper: Kotlin + WebView (minSdk 22, targetSdk 34)
- CI/CD: GitHub Actions  auto-builds and releases the APK on every push
- Hosting: GitHub Pages / any static host for the web version

**Repositories:**
- Web display: `https://github.com/Muhammad-786/MosqueDisplay.git` (branch: `master`)
- Android APK: `https://github.com/Muhammad-786/-mosquedisplay-app.git` (branch: `master`)

---

## 2. Website  Features and Architecture

### Key features

| Feature | Detail |
|---|---|
| Live clock | Updates every second, visible in `#clock-time` |
| Jummah schedule | Speech / Khutbah / Jamaat times stored in Firestore `mosque_data/settings` |
| Countdown | Auto-detects Friday; shows time-to-Jamaat, then "Jummah Mubarak", then "Next Jummah" |
| Slideshow | Image and text slides, 8 s per slide, progress bar, lazy background-image loading |
| Ken Burns | CSS zoom on slides  disabled inside the app to save GPU |
| Themes | Gold (default), Emerald, Royal, Rose, Custom hex |
| Layouts | Classic, Media Focus, Fullscreen, Vertical (portrait TVs) |
| Admin panel | Hidden trigger (bottom-left corner), Firebase Auth login, full settings UI |
| Real-time sync | Firestore `onSnapshot` listeners update display instantly on admin save |
| 10-min keep-alive | Polls Firestore every 10 min to catch changes after socket drops |

### JavaScript globals

```js
APP_CONFIG            // live settings: jummah, theme, customColor, layout, slides[]
DEFAULT_CONFIG        // fallback values if Firestore doc missing
CONSTANTS             // SLIDE_DURATION=8000, CLOCK_UPDATE_INTERVAL, etc.
dynamicSlides[]       // DOM elements currently in the slideshow container
currentAnimationFrame // setInterval handle for the slideshow ticker
currentSlideIndex     // which slide is active right now
slideshowGeneration   // incremented on every reinit; stale callbacks check this to self-abort
slideshowReinitTimer  // debounce handle  collapses double snapshot fires into one reinit
settingsApplyTimer    // debounce handle  collapses double settings snapshot fires
pendingFreeTimers[]   // setTimeout IDs for lazy background-image free step; cancelled on reinit
```

### How the slideshow engine works

1. Firebase `slides` snapshot arrives
2. **Change-detection guard**  JSON.stringify compare; skip reinit if slides are identical
3. If changed  **800 ms debounce** before calling `initSlideshowFromConfig()`
4. `initSlideshowFromConfig()`:
   - Cancels all `pendingFreeTimers` (stale free-memory callbacks from previous cycle)
   - Increments `slideshowGeneration`, captures `myGeneration = ++slideshowGeneration`
   - Removes old DOM slide elements, clears `dynamicSlides = []`
   - Builds new slide elements; preloads images via `new Image()`
   - Every `onload`/`onerror` checks `if (myGeneration !== slideshowGeneration) return`
   - When all loaded  calls `startSlideshowAnimation(dynamicSlides)`
5. `startSlideshowAnimation(allSlides)`:
   - Runs a `setInterval` at 250 ms (NOT requestAnimationFrame  saves CPU on Fire Stick)
   - On each slide advance: restores background-image on next slide; schedules 1.2 s cleanup for departing slide
   - Timer IDs stored in `pendingFreeTimers`
   - Entire callback in `try-catch`; error auto-recovers by restarting after 2 s
   - Bounds guard: if `currentSlideIndex` out of range, resets to 0

### How the settings snapshot works

1. Firebase `settings` snapshot arrives
2. **Change-detection**  compares jummah, theme, customColor, layout individually
3. **300 ms debounce** before applying to the DOM
4. Only calls `applyTheme()` if theme/color changed; only `applyLayout()` if layout changed
5. Only calls `loadSettingsToUI()` if the admin panel is open

---

## 3. Firebase Setup

1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. Create a project  add a **Web App**  copy the `firebaseConfig` object
3. Paste it into `index.html` replacing the existing `const firebaseConfig = { ... }` block
4. Enable **Firestore Database** (production mode)
5. Create these paths (or let the app create them on first run):
   - `mosque_data` / `settings` document
   - `slides` collection
6. Enable **Authentication  Email/Password**
7. Add your admin user in Authentication  Users

### Firestore Security Rules

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /mosque_data/{doc} {
      allow read: if true;
      allow write: if request.auth != null;
    }
    match /slides/{doc} {
      allow read: if true;
      allow write: if request.auth != null;
    }
  }
}
```

---

## 4. Admin Panel Guide

### Access
1. Open the display in a browser
2. Hover over the **bottom-left corner**  a glowing trigger button appears
3. Click it  enter your Firebase email/password

### Timings Tab
- Set Speech, Khutbah, Jamaat hours/minutes (023 / 059)
- **Display Start Time**  when on Friday the countdown activates

### Slideshow Tab
- **Upload Image**  client-side compressed before saving to Firestore
- **Add by URL**  direct link to any HTTPS image
- **Slide Manager**  reorder (up/down), hide/show toggle, delete
- Text slides support icon, title, body, colors, font size

### Appearance Tab
- **Themes:** Gold, Emerald, Royal, Rose, Custom
- **Custom color:** hex picker  dynamically adjusts all CSS color variables
- **Layouts:** Classic, Media Focus, Fullscreen, Vertical

### Save
Click **"Save & Update Cloud"**  writes to Firestore. All active displays update instantly.

---

## 5. Fire TV Browser Fix

**Problem:** Opening the website in the Fire Stick browser (even with "Show Desktop Site") still rendered the mobile layout because no proper desktop viewport or UA was set.

**Fix added to `<head>` of `index.html`:**
```html
<script>
  if (/AFT/i.test(navigator.userAgent)) {
    document.documentElement.setAttribute('data-device', 'firetv');
    var m = document.querySelector('meta[name="viewport"]');
    if (!m) { m = document.createElement('meta'); m.name='viewport'; document.head.appendChild(m); }
    m.content = 'width=1920, initial-scale=1.0';
  }
</script>
```

**CSS override block added at the bottom of the stylesheet:**
```css
html[data-device="firetv"] body { min-width: 1920px; }
html[data-device="firetv"] .container { max-width: 1920px; }
```

---

## 6. Android APK Project  Full Setup

### Project location
`C:\Users\Shahi\OneDrive\Desktop\mosquedisplay-app\`

### Tech specs

| Setting | Value |
|---|---|
| Language | Kotlin |
| minSdk | 22  covers Fire Stick Gen 1/2 (Android 5.1) |
| targetSdk | 34 |
| compileSdk | 34 |
| AGP | 8.2.2 |
| Gradle | 8.2 |
| Kotlin | 1.9.22 |
| JDK | 17 |

### Project structure

```
mosquedisplay-app/
 .github/workflows/build.yml         <- Auto-build + release APK on push
 app/
    build.gradle
    src/main/
        AndroidManifest.xml
        assets/index.html           <- Copy of the display website
        java/com/faizanemadinah/mosquedisplay/
            MainActivity.kt
            BootReceiver.kt
            NetworkReceiver.kt
 build.gradle
 settings.gradle
```

### AndroidManifest.xml  key settings

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-feature android:name="android.software.leanback" android:required="false" />
<uses-feature android:name="android.hardware.touchscreen" android:required="false" />

<application
    android:largeHeap="true"
    android:hardwareAccelerated="true"
    android:usesCleartextTraffic="true">

  <activity
      android:screenOrientation="landscape"
      android:launchMode="singleInstance"
      android:alwaysRetainTaskState="true"
      android:configChanges="orientation|screenSize|keyboardHidden">

    <!-- Standard app launcher -->
    <intent-filter>
      <action android:name="android.intent.action.MAIN" />
      <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>

    <!-- Fire TV / Android TV home screen -->
    <intent-filter>
      <action android:name="android.intent.action.MAIN" />
      <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
    </intent-filter>
  </activity>

  <receiver android:name=".BootReceiver" android:exported="true"
      android:permission="android.permission.RECEIVE_BOOT_COMPLETED">
    <intent-filter android:priority="999">
      <action android:name="android.intent.action.BOOT_COMPLETED" />
      <action android:name="android.intent.action.QUICKBOOT_POWERON" />
    </intent-filter>
  </receiver>
</application>
```

### MainActivity.kt  feature table

| Feature | How it works |
|---|---|
| Loads the display | `webView.loadUrl("file:///android_asset/index.html")` |
| Desktop UA | Custom Chrome/Windows user agent string  makes the site render its desktop layout |
| Wide viewport | `useWideViewPort=true`, `loadWithOverviewMode=true`, `setInitialScale(100)` |
| Zoom | `setSupportZoom(true)`, `builtInZoomControls=true`, `displayZoomControls=false`  pinch-to-zoom and double-tap to reset |
| Screen always on | `FLAG_KEEP_SCREEN_ON` on the window |
| Fullscreen | `hideSystemUI()`  hides status bar and nav bar on both old and new Android APIs |
| Cursor dot | JS injection: white 18px circle follows mouse pointer; shrinks/turns gold on click |
| Viewport injection | JS sets `meta[name="viewport"]` to `width=1920` on page load |
| Ken Burns disabled | JS tags `body[data-app="1"]`; CSS rule `body[data-app] .slide.active .slide-content { animation: none !important }` |
| Memory flush | `clearCache(false)` every 10 min via Handler |
| `onTrimMemory` | Clears cache on `TRIM_MEMORY_MODERATE` or higher |
| Heartbeat | Every 2 min: reads `#clock-time` via `evaluateJavascript`; if unchanged for 2 checks in a row (4 min)  `webView.reload()` |
| Network watchdog | `NetworkReceiver` detects offlineonline transition  `webView.reload()` |
| Error recovery | `onReceivedError` only retries when the failing URL is `index.html` itself  prevents reload storm from sub-resource failures |
| Boot auto-launch | `BootReceiver` handles `BOOT_COMPLETED` + `QUICKBOOT_POWERON` |
| Clean shutdown | `onDestroy` removes all handler callbacks, unregisters receivers, destroys WebView |

### BootReceiver.kt

```kotlin
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED ||
            intent.action == "android.intent.action.QUICKBOOT_POWERON") {
            val i = Intent(context, MainActivity::class.java).apply {
                addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            }
            context.startActivity(i)
        }
    }
}
```

### NetworkReceiver.kt

```kotlin
class NetworkReceiver(private val onNetworkRestored: () -> Unit) : BroadcastReceiver() {
    private var wasOffline = false
    override fun onReceive(context: Context, intent: Intent) {
        val cm = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val isOnline = cm.activeNetworkInfo?.isConnected == true
        if (!isOnline) { wasOffline = true }
        else if (wasOffline) { wasOffline = false; onNetworkRestored() }
    }
}
```

---

## 7. GitHub Actions  Automated APK Build

File: `.github/workflows/build.yml`

**Trigger:** Every push to `master`

**Steps:**
1. Check out code
2. Set up JDK 17
3. Set up Android SDK via `android-actions/setup-android@v3`
4. Cache Gradle via `gradle/actions/setup-gradle@v3`
5. Generate Gradle wrapper if missing
6. Run `./gradlew assembleDebug`
7. Upload APK as workflow artifact
8. Create GitHub Release with APK attached (tagged with commit SHA)

**Critical  must have this or the release step gives 403:**
```yaml
permissions:
  contents: write
```

**To get a new APK:**
1. Push any change to `master`
2. Wait 35 minutes for Actions to complete
3. Go to the repo  **Releases**  download the latest `.apk`
4. Sideload onto the Fire Stick via ADB or a file manager app

---

## 8. All Crash Fixes and Hardening  Chronological Log

### A  Mobile layout on Fire TV browser
**Problem:** Fire Stick browser showed mobile layout even with "Show Desktop Site".
**Fix:** UA-detection script in `<head>` sets `html[data-device="firetv"]` and forces `width=1920` viewport. CSS block applies desktop layout overrides.

### B  APK "Problem parsing the package"
**Problem:** Fire Stick Gen 1/2 run Android 5.1 (API 22). `minSdk` was set to 26.
**Fix:** Lowered `minSdk` to 22 in `app/build.gradle`.

### C  403 error when GitHub Actions tried to create a release
**Problem:** Workflow job had no write permission on the repo contents.
**Fix:** Added `permissions: contents: write` to the job in `build.yml`.

### D  Duplicate class compile error in MainActivity.kt
**Problem:** A file-edit operation prepended new code without removing the old class body, creating two identical Kotlin class declarations.
**Fix:** Rewrote the file completely using PowerShell `Set-Content`.

### E  App crashes after a few slides (Out of Memory)
**Problem:** All slide images were held in memory simultaneously. Fire Stick has ~1 GB RAM shared with the OS.
**Fixes:**
- `android:largeHeap="true"` in AndroidManifest
- Lazy background-image: stored in `data-bg-url` attribute; only applied to the currently visible slide
- 1.2 s deferred clear of background-image on the slide that just left
- `clearCache(false)` every 10 min via Handler
- `onTrimMemory` / `onLowMemory` event handlers

### F  Sluggishness and jank
**Problem:** `requestAnimationFrame` was being used for the progress bar  60 calls/sec, 3600 DOM writes/min on a slow CPU. Ken Burns CSS animation was running on the GPU continuously.
**Fixes:**
- Replaced `requestAnimationFrame` with `setInterval(fn, 250)` for the progress bar/slide tick
- `body[data-app] .slide.active .slide-content { animation: none !important }`  disables Ken Burns inside the app
- Removed all `System.gc()` calls (caused visible GC pause jank)
- `onReceivedError` scoped to `index.html` only  prevents a reload loop from favicon/font/image failures
- Cache flush interval raised from 3 min to 10 min

### G  App does not relaunch after Fire Stick reboot
**Problem:** Turning the Fire Stick off and on again returned to the Fire TV home screen, not the app.
**Fix:** `BootReceiver.kt` handles `BOOT_COMPLETED` + `QUICKBOOT_POWERON` (Fire TV variant) with `android:priority="999"`.

### H  Display stays blank after internet drops and returns
**Problem:** If the internet dropped while the WebView was active, the page stayed on an error or blank state even after connectivity returned.
**Fix:** `NetworkReceiver.kt` tracks offlineonline transitions and calls `webView.reload()`.

### I  Display silently freezes after long uptime
**Problem:** After hours/days of continuous running, the WebView could enter a frozen state with no visible error or crash.
**Fix:** Heartbeat Handler (every 2 min) reads the text content of `#clock-time` via `evaluateJavascript`. If the value is unchanged for 2 consecutive checks (4+ min), it triggers `webView.reload()`.

### J  Race condition crash when saving slides from admin panel
**Problem:** Firebase `onSnapshot` fires twice per write (local echo + server confirmation). Both calls triggered `initSlideshowFromConfig()`. The second call cleared `dynamicSlides = []` while the first call's `img.onload` callbacks were still running. Both eventually reached `startSlideshowAnimation()` and created two competing `setInterval` loops mutating `currentSlideIndex` simultaneously.
**Fixes:**
- **800 ms debounce** on the slides snapshot: collapses the two rapid fires into one `initSlideshowFromConfig()` call
- **Generation counter**: `slideshowGeneration` is incremented at the start of every reinit. Every async callback captures `myGeneration` and checks `if (myGeneration !== slideshowGeneration) return` before proceeding

### K  Crash when changing colour or layout from admin panel
**Problem:** Saving settings (theme/layout/colour) writes to the Firestore `settings` document. The Firebase SDK re-delivers the full `slides` snapshot to all active listeners on every reconnect  including the reconnect triggered by the settings write itself. A settings-only save therefore caused a full slideshow reinit with all images reloading, creating an OOM spike while the existing slideshow was still running.

Secondary: the settings snapshot also fires twice (local + server), so `applyTheme()` and `applyLayout()` ran simultaneously twice.
**Fixes:**
- **Slides change-detection guard**: `JSON.stringify(APP_CONFIG.slides) !== JSON.stringify(newSlides)`  if identical, skip reinit entirely. This is the primary fix for this crash.
- **Settings change-detection**: compare each field individually; only call `applyTheme` / `applyLayout` when that value actually changed
- **300 ms debounce** on settings application
- **`pendingFreeTimers[]` cleanup**: `initSlideshowFromConfig` cancels all pending 1.2 s free-memory callbacks before starting a new cycle, preventing them from blanking out images on the newly-loaded slides
- **Bounds guard** inside `setInterval`: if `currentSlideIndex` is out of range, reset to 0 instead of throwing
- **`try-catch` + auto-recovery**: any uncaught error inside the `setInterval` body logs, clears the interval, and schedules a clean restart after 2 s

---

## 9. Final Architecture Summary

```
index.html (single file, ~2700 lines)

 CSS
    Themes via data-theme attribute on body
    Layouts via data-layout attribute on body
    Fire TV overrides via html[data-device="firetv"]
    Ken Burns disabled via body[data-app] selector

 HTML
    Clock + date
    Jummah times table
    Countdown/Jummah Mubarak overlay
    Slideshow container
    Admin modal

 JavaScript
     Firebase init (Firestore + Auth)
     APP_CONFIG, DEFAULT_CONFIG, CONSTANTS
     initApp()
        setupRealtimeListener()
           settings onSnapshot  change detection  300ms debounce  applyTheme/applyLayout
           slides onSnapshot  change detection  800ms debounce  initSlideshowFromConfig
        setupAdminListeners()
        updateClock() + setInterval
        10-min keep-alive polls (settings + slides)
     initSlideshowFromConfig()
        Cancel pendingFreeTimers
        Bump slideshowGeneration, capture myGeneration
        Remove old DOM elements
        Async image preload with generation guards
     startSlideshowAnimation()
        setInterval at 250ms
        Lazy background-image (data-bg-url)
        pendingFreeTimers tracking
        Bounds guard
        try-catch + auto-recovery
     Admin panel (login, tabs, slide management, saveSettings)
     updateClock + checkCountdown

Android App (Kotlin)
 MainActivity.kt
    WebView: desktop UA, wide viewport, zoom, FLAG_KEEP_SCREEN_ON, fullscreen
    JS injection: viewport=1920, data-app tag, cursor dot, blob cleanup
    Handler: cache flush (10min) + heartbeat (2min)
    NetworkReceiver: reload on internet restore
    Lifecycle: clean pause/resume/destroy
 BootReceiver.kt  auto-launch on device boot
 NetworkReceiver.kt  detect offline  online, trigger reload
```

---

## 10. Reusable Prompt for Similar Projects

Use the prompt below when you want to repeat this entire process for a different web app.
Fill in the bracketed placeholders before submitting.

---

```
I have a single-file web app (index.html) that I want to:
1. Fix for Amazon Fire TV / Fire Stick browser (it shows mobile layout)
2. Package as an Android APK using Kotlin + WebView
3. Set up automated APK builds via GitHub Actions
4. Harden for 24/7 unattended display use on a TV

== CONTEXT ==
- My web app is: [DESCRIBE IT  e.g. "a digital signage display showing prayer times and a
  photo slideshow, powered by Firebase Firestore real-time listeners"]
- Real-time data source: [e.g. "Firebase Firestore onSnapshot on 'settings' and 'slides' collections"]
- Target device: Amazon Fire Stick [GENERATION] running Android [VERSION] / API [LEVEL]
- App name: [e.g. "MosqueDisplay"]
- Package name: [e.g. "com.yourname.appname"]
- Web repo: [YOUR WEB REPO URL]  branch: master
- APK repo:  [YOUR APK REPO URL] branch: master
- Local APK project path: [e.g. C:\Users\Me\Desktop\myapp\]

== PART 1  FIRE TV BROWSER FIX ==

Add a UA-detection script to the <head> of index.html (before any other scripts) that:
- Detects Fire TV (navigator.userAgent contains "AFT")
- Sets html[data-device="firetv"] attribute
- Overrides the viewport meta tag to: width=1920, initial-scale=1.0

Add a CSS block at the bottom of the stylesheet for html[data-device="firetv"] that
forces the desktop layout width (min-width: 1920px on body, max-width: 1920px on .container).

== PART 2  ANDROID PROJECT SCAFFOLD ==

Create a complete Android Studio Kotlin project with:
- minSdk: 22  (covers Fire Stick Gen 1/2 which run Android 5.1  API 22)
- targetSdk: 34, compileSdk: 34
- AGP: 8.2.2, Kotlin: 1.9.22, Gradle: 8.2, JDK: 17

The project must contain:

MainActivity.kt:
- Full-screen WebView loading index.html from assets/
- Desktop Chrome/Windows user agent string so the site renders desktop layout
- useWideViewPort=true, loadWithOverviewMode=true, setInitialScale(100)
- setSupportZoom(true), builtInZoomControls=true, displayZoomControls=false
  (enables pinch-to-zoom and double-tap to reset zoom)
- FLAG_KEEP_SCREEN_ON on the window
- hideSystemUI() that hides status bar and nav bar on both old (View flags) and
  new (WindowInsetsController) Android APIs
- JS injection on onPageFinished that:
    * Sets meta[name="viewport"] content to "width=1920, initial-scale=1.0"
    * Tags body with data-app="1" (so CSS can disable GPU animations inside the app)
    * Creates a visible white 18px cursor dot that follows mouse events (for Fire TV
      remote pointer visibility); shrinks and turns gold on mousedown; hides on keydown
    * Every 60 s: cleans up detached img elements and revokes orphaned blob URLs
- Handler task (every 10 min): calls webView.clearCache(false)
- Handler task (every 2 min  heartbeat): reads a clock/timer element via
  evaluateJavascript; if value unchanged for 2 consecutive checks (4+ min), calls
  webView.reload(); resets frozenCount on onPageFinished
- onReceivedError: only schedules a retry when failingUrl ends with the main HTML
  file name  do NOT retry on sub-resource failures (images, fonts, etc.)
- onTrimMemory and onLowMemory: call webView.clearCache(false)
- onDestroy: remove all handler callbacks, runCatching { unregisterReceiver },
  webView.stopLoading(), webView.clearCache(false), webView.destroy()

BootReceiver.kt:
- Handles Intent.ACTION_BOOT_COMPLETED and "android.intent.action.QUICKBOOT_POWERON"
- Launches MainActivity with FLAG_ACTIVITY_NEW_TASK

NetworkReceiver.kt:
- Tracks wasOffline boolean
- On connectivity change: if transitioning from offline to online, call the
  onNetworkRestored lambda
- Register in MainActivity with CONNECTIVITY_ACTION intent filter; unregister in onDestroy

AndroidManifest.xml:
- Permissions: INTERNET, ACCESS_NETWORK_STATE, RECEIVE_BOOT_COMPLETED, FOREGROUND_SERVICE
- <uses-feature android:name="android.software.leanback" android:required="false"/>
- <uses-feature android:name="android.hardware.touchscreen" android:required="false"/>
- Application flags: largeHeap="true", hardwareAccelerated="true", usesCleartextTraffic="true"
- Activity: screenOrientation="landscape", launchMode="singleInstance",
  alwaysRetainTaskState="true", configChanges="orientation|screenSize|keyboardHidden"
- Two intent-filter blocks on the activity: LAUNCHER and LEANBACK_LAUNCHER
  (so it appears on both normal Android and Fire TV home screen)
- BootReceiver with android:priority="999" and android:exported="true"

Add this CSS rule to index.html to disable GPU-heavy animations inside the app:
  body[data-app] .YOUR-SLIDE-CLASS.active .YOUR-CONTENT-CLASS {
      animation: none !important;
      transform: none !important;
  }
(Replace class names with whatever your app uses for animated elements)

Copy index.html into app/src/main/assets/index.html

== PART 3  GITHUB ACTIONS CI/CD ==

Create .github/workflows/build.yml that:
- Triggers on push to master
- Uses ubuntu-latest
- Sets up JDK 17
- Sets up Android SDK via android-actions/setup-android@v3
- Caches Gradle via gradle/actions/setup-gradle@v3
- Generates Gradle wrapper if missing (chmod +x gradlew)
- Runs ./gradlew assembleDebug
- Uploads the APK as a workflow artifact
- Creates a GitHub Release with the APK attached, tagged with the commit SHA
- Has this at the job level (REQUIRED  without it you get a 403 error on release creation):
    permissions:
      contents: write

== PART 4  REAL-TIME LISTENER HARDENING ==

Apply ALL of the following patterns to every real-time data listener in index.html.
These are non-optional  skipping any one of them will cause crashes on a real device.

4a. CHANGE DETECTION (most important fix)
Before reinitialising the UI from new data, compare new data to current data:
  const dataActuallyChanged = JSON.stringify(currentData) !== JSON.stringify(newData);
  if (!dataActuallyChanged) return;
Why: Firestore re-delivers the full snapshot on every reconnect, including reconnects
triggered by unrelated writes. Without this, saving settings causes the slideshow to
reinit and reload all images for no reason.

4b. DEBOUNCE ON DATA LISTENER
Wrap the reinit call in a clearTimeout + setTimeout(fn, 800ms):
  if (reinitTimer) clearTimeout(reinitTimer);
  reinitTimer = setTimeout(function() { reinitTimer = null; doReinit(); }, 800);
Why: Firestore fires each snapshot twice rapidly (local write echo + server confirmation).
Without the debounce, two reinit calls race each other.

4c. GENERATION COUNTER
Add a global: let dataGeneration = 0;
At the start of every reinit function: const myGeneration = ++dataGeneration;
In every async callback inside that function (onload, onerror, fetch.then, etc.):
  if (myGeneration !== dataGeneration) return;
Why: If a second reinit starts while the first is still loading images asynchronously,
this makes all the first call's pending callbacks silently abandon themselves.

4d. PENDING TIMER CLEANUP
If your reinit function schedules any setTimeout callbacks (e.g. to free memory,
clear DOM, etc.), track the IDs in an array: let pendingTimers = [];
At the START of every reinit:
  pendingTimers.forEach(function(id) { clearTimeout(id); });
  pendingTimers = [];
Store each new timer: const id = setTimeout(...); pendingTimers.push(id);
Why: Without this, a 1.2 s deferred "clear old content" callback fires AFTER the new
content has already been set up, blanking out the freshly-loaded display.

4e. SETTINGS LISTENER  SEPARATE DEBOUNCE + CHANGE DETECTION
If you have a separate listener for visual settings (theme, colour, layout, etc.):
  - Compare each individual field before applying it
  - Debounce at 300ms (settings fires are faster than data fires)
  - Only call applyTheme() if theme/colour actually changed
  - Only call applyLayout() if layout actually changed
  - Only update admin UI if the admin panel is currently open

4f. MAIN ANIMATION LOOP HARDENING
Wrap the entire body of every setInterval or requestAnimationFrame callback in try-catch:
  currentInterval = setInterval(function() {
      try {
          // ... your animation code ...
          // Bounds guard  before accessing array by index:
          if (currentIndex < 0 || currentIndex >= myArray.length) {
              currentIndex = 0; return;
          }
      } catch(err) {
          console.error('Animation error, restarting in 2s:', err);
          clearInterval(currentInterval);
          currentInterval = null;
          setTimeout(function() {
              if (!currentInterval) restartAnimation();
          }, 2000);
      }
  }, 250); // Use 250ms, NOT requestAnimationFrame  rAF fires 60x/sec on a slow CPU

4g. DO NOT USE requestAnimationFrame for progress bars or slide timers.
Use setInterval at 250ms instead. rAF fires 60 times per second = 7200 DOM writes per
minute on a Fire Stick CPU that is already throttled.

== PART 5  PUSH BOTH REPOS ==

After all changes are applied:
1. In the web repo:
   git add index.html
   git commit -m "fix: Fire TV viewport, 24/7 hardening, crash fixes"
   git push origin master

2. Copy updated index.html to the Android project assets:
   Copy-Item "index.html" "path/to/mosquedisplay-app/app/src/main/assets/index.html" -Force

3. In the APK repo:
   git add .
   git commit -m "fix: updated assets + Android hardening"
   git push origin master

GitHub Actions will automatically build and release the new APK within 35 minutes.

== COMMON GOTCHAS ==
- "Problem parsing the package" on install = minSdk too high for the device.
  Fire Stick Gen 1 and 2 run API 22. Always use minSdk 22.
- 403 on GitHub Release = missing "permissions: contents: write" in the workflow job.
- Duplicate class compile error = a file was prepended to instead of replaced.
  Rewrite the entire file using Set-Content (PowerShell) or direct file creation.
- App crashes when admin saves = almost always the race condition / double snapshot fire.
  Apply fixes 4a through 4f above completely.
- Ken Burns GPU animation makes Fire Stick overheat and stutter within minutes.
  Always disable it with a body[data-app] CSS selector.
- Do NOT call System.gc()  it causes visible GC pause jank.
- The Fire TV remote's D-pad moves a virtual mouse pointer. Inject the cursor dot
  so users can see where they are pointing.
```

---

*Last updated: March 2026. Project: Faizan-E-Madinah Display for Faizan-E-Madinah & Education Center.*
