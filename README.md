# MapZone Alert View SDK — Android

[![](https://jitpack.io/v/mapzone-global/mapzone-alert-view-android.svg)](https://jitpack.io/#mapzone-global/mapzone-alert-view-android)

This SDK is provided only for Vietmap MAPs API enterprise customers. Contact your Vietmap account manager for access or [Vietmap Solutions](https://zalo.me/3189066936017422854) Zalo OA if you are interested in becoming a customer.

**Route-based speed alert.** You hand the SDK **one full route polyline** (the geometry of a navigation route, encoded `1e6`). The native engine splits it into short segments, fetches speed limits + signs for each from the Vietmap service on demand, and — on every GPS frame — hands you back speed-sign bitmaps and voice cues as the driver progresses along that route.

---

## Installation

### 1. Add the JitPack repository

**Gradle 7.0+ (`settings.gradle` / `settings.gradle.kts`):**

```gradle
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

**Legacy (`build.gradle` at project root):**

```gradle
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

### 2. Add the dependency

```gradle
dependencies {
    implementation 'com.github.mapzone-global:mapzone-alert-view-android:<latest-version>'
}
```

### 3. Requirements

| Setting | Value |
|---|---|
| `minSdk` | **24** or higher |
| `compileSdk` | 36 or higher |
| Java / Kotlin JVM target | **11** or higher (`sourceCompatibility` / `targetCompatibility` = `VERSION_11`) |
| Native ABIs shipped | `arm64-v8a`, `armeabi-v7a`, `x86_64` (16 KB page size supported) |
| R8 / ProGuard | Nothing to add — the AAR ships its own consumer rules |

## Permissions

### `AndroidManifest.xml`

```xml
<!-- The SDK fetches segment data over the network. Already declared by the
     AAR and merged into your manifest — listed here for reference. -->
<uses-permission android:name="android.permission.INTERNET" />

<!-- Required by your app to obtain the GPS you feed into onLocation.
     On Android 12+ (API 31) a FINE request must be accompanied by COARSE. -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- Only if you keep alerting while the app is backgrounded (a nav app usually does) -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
<!-- Android 13+ (API 33): to show the foreground-service notification -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

> **Note:** The SDK does **not** request location itself — your app collects GPS
> and feeds it into `onLocation`. On Android 14+ (API 34+), if the engine runs
> inside a foreground service, declare `FOREGROUND_SERVICE_LOCATION` and start
> the service with the matching `foregroundServiceType`.

### Runtime permission request (Android 6.0+)

```kotlin
private val permissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { results ->
    val granted = results.values.all { it }
    if (granted) startNavigation() else showRationale()
}

permissionLauncher.launch(
    arrayOf(
        Manifest.permission.ACCESS_FINE_LOCATION,
        Manifest.permission.ACCESS_COARSE_LOCATION
    )
)
```

---

## Quick Integration — `AlertViewManager`

`AlertViewManager` is the only public entry point. It owns the native engine and
serialises all GPS / network work on an internal single-thread executor — the
public API is safe to call from the main thread, and all callbacks arrive back
on the main thread.

### 1. Create and configure

Call `configure(...)` **before `start`**.

```java
import com.vietmap.alert_view_sdk.AlertViewManager;

AlertViewManager manager = new AlertViewManager();

manager.configure(
        /* baseUrl       */ "https://driving.map.zone",   // Vietmap service base URL
        /* apiKeyId      */ "<your-api-key-id>",           // issued by Vietmap
        /* apiKey        */ "<your-api-key>",              // issued by Vietmap
        /* vehicleId     */ "<your-vehicle-id>",           // issued by Vietmap
        /* vehicleType   */ 1,                             // 1..9 (see Vehicle Types)
        /* seats         */ 0,                             // seats for coaches; 0 = default for the type
        /* weights       */ 0.0,                           // gross weight (tonnes) for trucks; 0 = default
        /* maxSnapMeters */ 0.0);                          // max snap distance (m); <= 0 = server default (~25 m)
```

> **Bundle id / allow-listing:** the SDK sends your app's *installed*
> `applicationId` — **including any `applicationIdSuffix` or flavor suffix**
> (e.g. `com.foo.debug`) — as the auth bundle id. The auth server accepts a
> single exact application id and does **not** whitelist suffixed variants.
> Register the exact `applicationId` of every build you ship with the Vietmap
> team, or build those variants without a suffix.

**Optional configuration** (call after `configure`, before `start`; all settings persist across reroutes):

| Method | Purpose |
|---|---|
| `setExtraHeaders(Map<String,String>)` | **Add** headers to the segment request. Returns `null` on success, or a non-null error message. Cannot override reserved headers (e.g. `Content-Type`, case-insensitive); a header key may not contain `:`. |
| `setExtraBodyFields(Map<String,String>)` | **Add** body fields (string values) to the segment request. Returns `null` on success. Cannot override the SDK's own fields. |
| `setMutedAlertTypes(VoiceAlertType...)` | Mute specific alert voices (camera, toll, …). Speed-limit & speeding voices can never be muted. Call with no arguments to re-enable all. |

`setExtraHeaders` / `setExtraBodyFields` reject empty keys, reserved keys, a
colon in a header key, and any value containing a newline; on rejection they
return a non-null message and **keep the previous extras**. Pass an empty/`null`
map to clear.

### 2. Register callbacks

All callbacks fire on the main thread. Wire only the ones you need.

```java
// (a) Per-tick bitmaps + first voice clip — the core driver of your UI.
manager.setBitmapCallback((currentBmp, speedStatus,
                           nextBmp,    nextDistMeters,
                           cameraBmp,  cameraDistMeters,
                           tollBmp,    tollDistMeters,
                           voiceWav) -> {

    // Current speed-limit sign (null = no link under the GPS point).
    ivSpeedSign.setImageBitmap(currentBmp);

    // Overspeed indicator: 0 = within limit, 1 = approaching (within 5 km/h), 2 = over limit.
    statusBar.setBackgroundColor(
        speedStatus == 2 ? Color.parseColor("#F44336") :   // red
        speedStatus == 1 ? Color.parseColor("#FF9800") :   // orange
                           Color.parseColor("#4CAF50"));    // green

    // Upcoming sign / camera / toll previews. Hide the slot when the bitmap is null.
    bindMiniSign(ivNext,   nextBmp,   nextDistMeters);
    bindMiniSign(ivCamera, cameraBmp, cameraDistMeters);
    bindMiniSign(ivToll,   tollBmp,   tollDistMeters);

    // voiceWav is null when nothing is queued. See the voice note below.
});

// (b) Fetch outcome — surfaces server errors. errorCode == 0 means success.
manager.setResultCallback((success, errorCode, errorMessage) -> {
    if (!success && errorCode != 0) {
        Log.w(TAG, "fetch error code=" + errorCode + ": " + errorMessage);
    }
});

// (c) Route lost — the driver left the route you passed to start().
//     Wire this if your app can re-route; see "Reroute" below.
manager.setRerouteCallback((lat, lng) -> requestNewRoute(lat, lng));
```

> **Note (voice):** The Android SDK has a **built-in, priority-aware
> `MediaPlayer` queue** that plays voice cues automatically.
>
> You only need a `VoiceCallback` if you want to mix audio yourself, route it to
> a specific stream, or apply your own prioritisation. Each cue carries its
> `trigger` and `priority`:
>
> ```java
> manager.setVoiceCallback(new VoiceCallback() {
>     // Richer variant — the SDK always calls this one.
>     @Override public void onVoice(byte[] wavBytes, int trigger, int priority) {
>         // priority: 0 = "current" (lowest, skipped first when congested)
>         //           1 = normal (approaching / camera / sign)
>         //           2 = speeding (most urgent)
>         // trigger:  native VoiceTrigger value (see Voice Alert Types)
>         myPlayer.enqueue(wavBytes, priority);
>     }
>     // Legacy single-arg variant — still required by the interface; the
>     // two-arg default forwards here, so existing code keeps working.
>     @Override public void onVoice(byte[] wavBytes) { myPlayer.enqueue(wavBytes, 1); }
> });
> // Once a callback is set, the built-in player is disabled and your app becomes
> // responsible for ALL voice playback (including the voiceWav from onBitmap).
> // Pass null to restore the default player.
> ```
>
> Existing single-arg lambdas (`setVoiceCallback(wavBytes -> ...)`) still compile
> and run — the two-arg method has a default that forwards to `onVoice(byte[])`.
>
> WAV format: PCM 16-bit little-endian, mono, 22 050 Hz — ready for
> `MediaPlayer`, `AudioTrack`, or `ExoPlayer`.

### 3. Start a route & feed GPS

You start with the **full route polyline** (start→end), then feed every GPS frame.

```kotlin
// Start (or reroute — just call start() again with a new polyline):
manager.start(routePolyline)   // route geometry, encoded 1e6

private val locationListener = LocationListener { loc ->
    val speedKmh = if (loc.hasSpeed()) loc.speed * 3.6 else 0.0   // Location.speed is m/s
    val bearing  = if (loc.hasBearing()) loc.bearing.toDouble() else 0.0

    manager.onLocation(
        loc.latitude,
        loc.longitude,
        bearing,             // degrees, 0 = N, 90 = E
        speedKmh,            // current speed in km/h (SDK converts to m/s internally)
        loc.accuracy.toDouble(),
        loc.time)            // fix timestamp, ms since epoch (UTC)
}
```

> **Note:** `onLocation` takes speed in **km/h** and the fix timestamp in
> **milliseconds since the epoch (UTC)** — pass `Location.getTime()` straight
> through, do not substitute an uptime/elapsed-time clock. Feed each fix as it
> arrives, at the usual GPS cadence (~1 Hz). Pass *snapped-to-route* coordinates
> if your app already runs map-matched navigation: raw `Location` samples can be
> 10–20 m off the road centre even with good `accuracy`.

### 4. Reroute — when the driver leaves the route

The SDK alerts along the route you handed to `start()`; it does **not** compute
routes. If the driver takes a different road and the SDK cannot get back onto
that route, it reports the position once and then has nothing left to announce
until you hand it a new route.

```
   on route ──────────► driver takes a different road
                                   │
                                   ▼
                        onNeedReroute(lat, lng)      ← once per detour
                                   │
              your nav layer builds a new route from (lat, lng)
                                   │
                                   ▼
                        manager.start(newPolyline)   ← alerts resume
```

```java
manager.setRerouteCallback((lat, lng) -> {
    // Ask your directions provider for a fresh route from here, then:
    //   manager.start(newRoute.geometry());
});
```

Registering `RerouteCallback` is optional but strongly recommended for
navigation apps: without it, alerting simply stays quiet after a detour. It
fires at most once per detour and re-arms once the vehicle is back on a route.

### 5. Reset

```kotlin
manager.reset()    // releases route data + voice state
```

Call this when navigation finishes, or before a reroute if you want to free
engine memory immediately.

---

## Voice Alert Types (mute control)

Pass the categories you want to **mute** to `setMutedAlertTypes(...)`; everything
not listed keeps announcing. Muting affects **voice only** — the on-screen
sign/preview bitmap still shows. Speed-limit and speeding warnings are core
safety cues and can never be muted.

```java
import com.vietmap.alert_view_sdk.VoiceAlertType;

// Mute camera + toll voices; keep every other voice and all visual signs.
manager.setMutedAlertTypes(VoiceAlertType.SPEED_CAMERA, VoiceAlertType.TOLL);

// Re-enable all voices:
manager.setMutedAlertTypes();
```

| `VoiceAlertType` | Meaning |
|---|---|
| `SPEED_CAMERA` | Speed camera |
| `TOLL` | Toll booth |
| `TRAFFIC_ENFORCEMENT_CAMERA` | Traffic-enforcement camera |
| `RED_LIGHT_CAMERA` | Red-light camera |
| `AI_CAMERA` | AI camera |
| `NO_LEFT_TURN` / `NO_RIGHT_TURN` / `NO_UTURN` / `NO_STRAIGHT` | Turn restrictions |
| `NO_OVERTAKING` / `NO_OVERTAKING_END` | No-overtaking zone start / end |
| `NO_PARKING` | No parking |
| `BUILDUP_AREA_START` / `BUILDUP_AREA_END` | Built-up (urban) area start / end |
| `REST_STATION` | Rest station |

---

## Vehicle Types

`configure(...)` takes a raw integer `vehicleType` in **1..9**.

| `vehicleType` | Description |
|---|---|
| `1` | Car (xe ô tô) |
| `2` | Motorcycle (xe mô tô) |
| `3` | Truck (xe tải) |
| `4` | Coach (xe khách) |
| `5` | Bus (xe bus) |
| `6` | Taxi (xe taxi) |
| `7` | Bicycle (xe đạp) |
| `8` | Pedestrian (người đi bộ) |
| `9` | Emergency (xe ưu tiên) |

---

## Error Codes (`ResultCallback.onResult`)

The codes below are the ones integrations hit in practice. `errorMessage` is a
short, display-ready message you can surface in your UI as-is.

| Code | Meaning |
|---|---|
| `0` | Success — segment data loaded |
| `1001` | A route point is outside the supported area (Vietnam) |
| `1002` | Missing points / invalid request |
| `2002` | Unauthorized — `apiKeyId` / `apiKey` / bundle id mismatch |
| `3003` | Vehicle type not supported (must be `1..9`) |
| `3004` | Route did not map-match to any road link |
| negative | Local SDK failure (no connectivity, or route data unavailable) — read `errorMessage` |

---

## Threading Model

```
 Main thread ──► start / onLocation / reset
                          │
                          ▼
              Internal single-thread ExecutorService
              (HTTP fetch + 1-D snap + matching + bitmap/voice render)
                          │
                          ▼
              Main thread ◄── BitmapCallback / ResultCallback / VoiceCallback
```

All public methods are safe to call from the Android main thread. `start` and
`onLocation` are serialised on the same executor, so they never race. Callbacks
always arrive back on the main thread — you can update views directly without
`runOnUiThread {}`.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No bitmaps at all | `start()` not called, or empty/invalid polyline | Pass a route polyline **encoded with precision 1e6** (the geometry from a Vietmap Directions response), and call `start` before `onLocation`. |
| `currentBmp` is always `null` | GPS point is too far from the route | Feed real GPS frames near the route; raise `maxSnapMeters` in `configure` (e.g. 30–50 m) if your GPS is noisy. |
| Speed status always `0` | Speed passed in m/s | `onLocation` expects **km/h** — multiply `Location.speed` (m/s) by `3.6`. |
| `errorCode 2002` | Wrong API key / secret, or bundle id not allow-listed | Re-check `apiKeyId` / `apiKey`; make sure your exact `applicationId` is registered with the Vietmap team. |
| `errorCode 3003` | Bad `vehicleType` | Pass a value in **1..9** (see Vehicle Types). |
| `errorCode 3004` | Route did not map-match | Verify the polyline is a real road route and encoded at `1e6`. |
| `errorCode < 0` | Local network / parse failure | Check connectivity, `baseUrl`, and the `INTERNET` permission; read `errorMessage`. |
| `UnsatisfiedLinkError` for `libalert_view_sdk.so` | `abiFilters` / packaging rules in your app exclude the device ABI | Keep the full set `arm64-v8a`, `armeabi-v7a`, `x86_64`. |
| Wrong sign on the wrong road | Raw GPS instead of snapped coordinates | Pass map-matched coordinates (see step 3 note). |
| Voice never plays | A `VoiceCallback` is registered but does nothing | Either implement playback in your callback, or **don't** register one and let the built-in player handle it. |

---

## Demo

Check the demo app on [GitHub](https://github.com/mapzone-global/mapzone-alert-view-app-android).
