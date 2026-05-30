# Unifium Video Resolver SDK

Developer guide for building **resolver plugin apps** that work
with [Unifium Player](https://play.google.com/store/apps/details?id=com.andatsoft.apps.unifium).

A resolver plugin receives a **page URL** (for example a YouTube or Facebook watch link) and returns
a **direct playable stream URL** (HLS, DASH, MP4, etc.) plus optional playback metadata.

---

## Table of contents

1. [Overview](#overview)
2. [Requirements](#requirements)
3. [Add the SDK to your app](#add-the-sdk-to-your-app)
4. [Quick start](#quick-start)
5. [Implementing the resolver service](#implementing-the-resolver-service)
6. [Match scoring](#match-scoring)
7. [Bundle contracts (V1)](#bundle-contracts-v1)
8. [AIDL interfaces](#aidl-interfaces)
9. [Threading and performance](#threading-and-performance)
10. [Error handling](#error-handling)
11. [ProGuard / R8](#proguard--r8)
12. [Testing](#testing)

---

## Overview

```text
┌─────────────────────┐         bind + AIDL          ┌──────────────────────┐
│  Host app           │  ──────────────────────────► │  Your plugin app     │
│  (Unifium Player)   │  ◄────────────────────────── │  ResolverService     │
│                     │   Bundle result (stream URL) │                      │
└─────────────────────┘                              └──────────────────────┘
```

| Role           | Responsibility                                                                          |
|----------------|-----------------------------------------------------------------------------------------|
| **Host app**   | Discovers plugins, calls `match()`, invokes `resolveAsync()`, plays the returned stream |
| **Plugin app** | Exposes an exported `Service`, implements URL matching and extraction                   |

Communication uses:

- **Android Bound Service** + **AIDL**
- **Bundle** request/response (no `Parcelable` models, no `Serializable`)

The SDK module (`resolver-sdk`) ships:

- AIDL interfaces
- Contract constants (`ResolverContract`)
- Bundle helpers (`ResolverRequest`, `ResolverResult`)
- Optional base class (`BaseVideoResolverService`)

---

## Requirements

| Item         | Value                                     |
|--------------|-------------------------------------------|
| Language     | Kotlin                                    |
| `minSdk`     | 23+                                       |
| `compileSdk` | 36 (recommended; align with the host app) |
| JVM          | 17                                        |
| Coroutines   | Required for `BaseVideoResolverService`   |

---

## Add the SDK to your app

Add the Unifium Maven repository to your `settings.gradle` or root `build.gradle`:

```gradle
dependencyResolutionManagement {
    repositories {
        mavenCentral()
        maven { url 'https://raw.githubusercontent.com/andatsoft/unifium-resolver-sdk/main/' }
    }
}
```

Then add the dependency to your app module `build.gradle`:

```gradle
dependencies {
    implementation 'com.andatsoft.ext:resolver-sdk:1.0.0'
}
```

---

## Quick start

### 1. Create the service

```kotlin
package com.example.myresolver

import android.os.Bundle
import com.andatsoft.ext.resolver.BaseVideoResolverService
import com.andatsoft.ext.resolver.ResolverContract
import com.andatsoft.ext.resolver.ResolverResult

class MyVideoResolverService : BaseVideoResolverService() {

    override fun provideResolverName(): String = "My Site Resolver"

    override fun computeMatchScore(url: String): Int {
        return when {
            url.contains("example.com/watch") -> ResolverContract.MATCH_EXACT
            url.contains("example.com") -> ResolverContract.MATCH_PROBABLE
            else -> ResolverContract.MATCH_UNSUPPORTED
        }
    }

    override suspend fun resolvePageUrl(pageUrl: String, request: Bundle): Bundle {
        // Run network / parsing on Dispatchers.IO (already provided by the base class).
        val streamUrl = extractStreamUrl(pageUrl) // your logic
            ?: return ResolverResult.failure("Could not extract stream URL")

        val headers = Bundle().apply {
            putString("Referer", pageUrl)
            putString("User-Agent", "MyResolver/1.0")
        }

        return ResolverResult.success(
            videoUrl = streamUrl,
            streamType = ResolverContract.STREAM_TYPE_HLS,
            headers = headers,
        )
    }

    private fun extractStreamUrl(pageUrl: String): String? {
        // TODO: implement extraction
        return null
    }
}
```

### 2. Register in AndroidManifest.xml

```xml

<service android:name=".MyVideoResolverService" android:exported="true">

    <intent-filter>
        <action android:name="com.andatsoft.ext.intent.RESOLVE_VIDEO" />
    </intent-filter>
</service>
```

> **Important:** `android:exported="true"` and the intent action
`com.andatsoft.ext.intent.RESOLVE_VIDEO` are **required** for host discovery.
---

## Implementing the resolver service

The base class:

- Implements `IVideoResolverService` binder stubs
- Runs `resolvePageUrl()` on `Dispatchers.IO`
- Never blocks the Binder thread
- Dispatches `onSuccess` / `onError` on the callback safely

Override three methods:

| Method                             | Purpose                                       |
|------------------------------------|-----------------------------------------------|
| `provideResolverName()`            | Display name in the host resolver picker      |
| `computeMatchScore(url)`           | Return 0–100 support score (see below)        |
| `resolvePageUrl(pageUrl, request)` | Return a result `Bundle` (success or failure) |

---

## Match scoring

Return an `int` from `match(url)`:

| Constant                             | Value | Meaning                                      |
|--------------------------------------|------:|----------------------------------------------|
| `ResolverContract.MATCH_UNSUPPORTED` |     0 | This plugin cannot handle the URL            |
| `ResolverContract.MATCH_GENERIC`     |     1 | Weak / fallback support                      |
| `ResolverContract.MATCH_PROBABLE`    |    50 | Likely supported (same domain, typical path) |
| `ResolverContract.MATCH_EXACT`       |   100 | Confident match (known URL pattern)          |

**Guidelines:**

- Return `0` when you are sure the URL is not yours.
- Prefer `100` only for URLs you handle reliably.
- The host sorts plugins by score and may **auto-select** when one plugin clearly wins.

**Example:**

```kotlin
override fun computeMatchScore(url: String): Int {
    val host = Uri.parse(url).host?.lowercase() ?: return ResolverContract.MATCH_UNSUPPORTED
    return when {
        host == "www.youtube.com" && url.contains("watch?v=") -> ResolverContract.MATCH_EXACT
        host.endsWith("youtube.com") -> ResolverContract.MATCH_PROBABLE
        else -> ResolverContract.MATCH_UNSUPPORTED
    }
}
```

---

## Bundle contracts (V1)

Contracts use string keys in `android.os.Bundle` for forward compatibility. Optional fields may be
added in future API versions.

### Request (host → plugin)

Built with `ResolverRequest.build()`:

| Key           | Type   | Required | Description                               |
|---------------|--------|----------|-------------------------------------------|
| `url`         | String | Yes      | Page or input URL to resolve              |
| `api_version` | int    | No       | Contract version (default `1`)            |
| `timeout`     | long   | No       | Suggested timeout in ms (default `30000`) |

```kotlin
val request = ResolverRequest.build(
    url = "https://example.com/watch/123",
    apiVersion = ResolverContract.API_VERSION,
    timeoutMs = 30_000L,
)
```

Read values in your service:

```kotlin
val pageUrl = ResolverRequest.getUrl(request)
val timeoutMs = ResolverRequest.getTimeoutMs(request)
```

Respect `timeout` in long-running work; cancel jobs when possible if the host disconnects.

### Result (plugin → host)

#### Success

Use `ResolverResult.success()`:

| Key            | Type    | Required | Description                                                                                             |
|----------------|---------|----------|---------------------------------------------------------------------------------------------------------|
| `success`      | boolean | Yes      | Must be `true`                                                                                          |
| `video_url`    | String  | Yes      | Direct playable URL (HLS, DASH, MP4, etc.). Must use `http://` or `https://` (`localhost` is rejected). |
| `stream_type`  | String  | No       | Hint: `hls`, `dash`, `mp4`, `other`                                                                     |
| `headers`      | Bundle  | No       | HTTP headers for playback (nested string map)                                                           |
| `subtitle_url` | String  | No       | External subtitle URL. Must use `http://` or `https://` (`localhost` is rejected).                      |

```kotlin
val headers = Bundle().apply {
    putString("Referer", "https://example.com/")
    putString("Origin", "https://example.com")
}

return ResolverResult.success(
    videoUrl = "https://cdn.example.com/video.m3u8",
    streamType = ResolverContract.STREAM_TYPE_HLS,
    headers = headers,
    subtitleUrl = "https://cdn.example.com/subs.vtt",
)
```

#### Failure

Use `ResolverResult.failure()` or `onError()` via the base class:

| Key       | Type    | Required | Description                  |
|-----------|---------|----------|------------------------------|
| `success` | boolean | Yes      | Must be `false`              |
| `error`   | String  | Yes      | Human-readable error message |

```kotlin
return ResolverResult.failure("Video is private or unavailable")
```

---

## AIDL interfaces

Package: `com.andatsoft.ext.resolver`

### `IVideoResolverService`

```java
interface IVideoResolverService {
    int getApiVersion();

    String getResolverName();

    int match(String url);

    void resolveAsync(in Bundle request, IResolveCallback callback);
}
```

| Method                            | Description                                 |
|-----------------------------------|---------------------------------------------|
| `getApiVersion()`                 | Return `ResolverContract.API_VERSION` (`1`) |
| `getResolverName()`               | User-visible plugin name                    |
| `match(url)`                      | Support score 0–100                         |
| `resolveAsync(request, callback)` | **Async only** — invoke callback when done  |

### `IResolveCallback`

```java
interface IResolveCallback {
    void onSuccess(in Bundle result);

    void onError(String error);
}
```

Call **exactly one** of `onSuccess` or `onError` per request.

---

## Threading and performance

| Rule                           | Detail                                                                        |
|--------------------------------|-------------------------------------------------------------------------------|
| Do not block the Binder thread | No network, disk, or heavy CPU in `match()` or binder methods                 |
| Use background threads         | `BaseVideoResolverService` uses coroutines + `Dispatchers.IO`                 |
| Async only                     | `resolveAsync` must return quickly; deliver result via callback               |
| Timeouts                       | Host may unbind after `timeout` ms; handle `RemoteException` if the host dies |

---

## Error handling

| Scenario                      | Recommended response                                 |
|-------------------------------|------------------------------------------------------|
| Network error                 | `ResolverResult.failure("Network error: ...")`       |
| Parse / extraction failed     | `ResolverResult.failure("...")` with a clear message |
| Invalid request (missing URL) | `ResolverResult.failure("Missing or invalid url")`   |

Do not throw uncaught exceptions from the Binder thread. The base class maps exceptions to
`onError()`.

---

## ProGuard / R8

If you enable minification in your plugin app, keep AIDL stubs:

```proguard
-keep class com.andatsoft.ext.resolver.** { *; }
-keep interface com.andatsoft.ext.resolver.** { *; }
```

The SDK ships `consumer-rules.pro` for library consumers.

---

## Testing

### With Unifium Player

1. Build and install your resolver plugin APK.
2. Install Unifium Player and enable your plugin.
3. In Unifium: **More → Network stream**, paste a URL your plugin supports.
4. If multiple plugins match, the host shows a **resolver picker** (name, icon, confidence).
5. After resolve, playback should start with the returned `video_url`.
