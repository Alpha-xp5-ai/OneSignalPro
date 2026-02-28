# OneSignalPro — Unreal Engine 4 Plugin

> Production-ready OneSignal push notification integration for Android & iOS.
> Full Blueprint support · Zero boilerplate · Drop-in plugin.

**Version:** 1.0.0 &nbsp;|&nbsp; **Engine:** UE 4.27+ &nbsp;|&nbsp; **Platforms:** Android · iOS &nbsp;|&nbsp; **Author:** Alpha XP

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Platform Support](#platform-support)
4. [Architecture](#architecture)
5. [Blueprint API Reference](#blueprint-api-reference)
6. [Events & Delegates](#events--delegates)
7. [Data Structures](#data-structures)
8. [Setup Guide](#setup-guide)
9. [Project Settings](#project-settings)
10. [C++ Usage](#c-usage)
11. [File Structure](#file-structure)
12. [Technical Requirements](#technical-requirements)
13. [Support](#support)

---

## Overview

**OneSignalPro** is a production-grade UE4 plugin that integrates [OneSignal](https://onesignal.com) push notification services into your mobile game. It handles all platform complexity — Android JNI bridges, iOS Objective-C++ wrappers, Gradle dependencies, CocoaPods, and manifest permissions — so you focus on your game logic.

- Clean layered architecture (Subsystem → Interface → Platform impl)
- All features accessible from **Blueprint with no C++ required**
- **Auto-Initialize** mode: set App ID in Project Settings and the SDK starts automatically
- All callbacks safely dispatched to the **game thread**

---

## Features

### Core Notifications
- Initialize OneSignal with App ID
- Auto-initialize on game start (zero Blueprint code needed)
- Receive foreground push notifications
- Handle notification-opened events (user tap from background/killed state)
- Player ID received callback
- Enable / disable push subscription per device
- Prompt for push permissions (iOS OS dialog)

### Device Segmentation — Tags
- Send a single key/value tag
- Send multiple tags in one call (`TMap<FString, FString>`)
- Delete a tag by key
- Link device to your backend user via External User ID
- Remove External User ID

### Blueprint Events (Delegates)
- `OnNotificationReceived` — foreground notification arrived
- `OnNotificationOpened` — user tapped a notification
- `OnPlayerIdReceived` — device Player ID is available
- All events fire on the game thread (safe to use with UObjects)

### Developer Experience
- Project Settings UI via `UDeveloperSettings`
- Verbose debug logging toggle (`bEnableDebugLogs`)
- Dedicated log category: `LogOneSignal`
- No engine source modifications required
- Compatible with packaged **Shipping** builds
- Works in Editor without any platform SDK (null stub)
- Subsystem auto-created/destroyed with the Game Instance

---

## Platform Support

| Platform | Status | SDK |
|----------|--------|-----|
| Android | ✅ Full implementation | OneSignal Android SDK 4.x (Gradle) |
| iOS | ✅ Full implementation | OneSignal iOS SDK 3.x (CocoaPod) |
| Win64 | ⬜ Compiles (null stub) | — |
| Mac | ⬜ Compiles (null stub) | — |
| Linux | ⬜ Compiles (null stub) | — |

> The null stub lets the plugin compile cleanly in the Editor on any desktop platform. All push APIs are no-ops outside Android/iOS.

---

## Architecture

```
Blueprint / C++ user code
        │
        ▼
UOneSignalProBPLibrary      ← thin static Blueprint wrappers (stateless)
        │
        ▼
UOneSignalManager           ← UGameInstanceSubsystem (owns state + delegates)
        │
        ▼
IOneSignalPlatform          ← abstract C++ interface
   ┌────┴──────────────────┬─────────────────┐
   ▼                       ▼                 ▼
FOneSignalAndroid     FOneSignalIOS     FOneSignalDefault
 (JNI bridge)        (Obj-C++ bridge)   (null stub)
```

### Android Data Flow

```
OneSignal Cloud → FCM → Android OS
                           │
               Java foreground handler (GameActivity)
                           │  nativeOnNotificationReceived(title, body, id)
               AsyncTask → GameThread
                           │
               FOneSignalAndroid::OnJavaNotificationReceived()
                           │  broadcasts native delegate
               UOneSignalManager::HandleNotificationReceived()
                           │  broadcasts BP delegate
               OnNotificationReceived  ← Blueprint event fires
```

### iOS Data Flow

```
OneSignal Cloud → APNs → iOS OS
                            │
              OneSignal SDK handler (CocoaPod 3.x)
                            │  dispatch_get_main_queue()
              Obj-C++ OSNotification handler
                            │
              AsyncTask → GameThread
                            │
              UOneSignalManager::HandleNotificationOpened()
                            │
              OnNotificationOpened  ← Blueprint event fires
```

---

## Blueprint API Reference

### Category: `OneSignal`

| Node | Type | Description | Inputs |
|------|------|-------------|--------|
| **Initialize OneSignal** | Callable | Start the SDK. Skip if Auto-Initialize is on. | `AppId` (String), `bEnableDebug` (bool) |
| **Get OneSignal Manager** | Callable | Returns the subsystem instance. Use to bind events. | — |
| **Is Initialized** | Pure | Returns `true` after a successful Initialize. | — |

### Category: `OneSignal\|User`

| Node | Type | Description | Inputs |
|------|------|-------------|--------|
| **Get Player ID** | Pure | OneSignal device Player ID. Empty if not yet available. | — |
| **Set External User ID** | Callable | Link this device to your backend user. | `ExternalId` (String) |
| **Remove External User ID** | Callable | Clear the external user ID. | — |

### Category: `OneSignal\|Tags`

| Node | Type | Description | Inputs |
|------|------|-------------|--------|
| **Send Tag** | Callable | Tag device with a single key/value pair. | `Key` (String), `Value` (String) |
| **Send Tags** | Callable | Send multiple tags in one call. | `Tags` (Map String→String) |
| **Delete Tag** | Callable | Remove a previously sent tag. | `Key` (String) |

### Category: `OneSignal\|Permissions`

| Node | Type | Description | Inputs |
|------|------|-------------|--------|
| **Prompt For Push Permissions** | Callable | Show the OS permission dialog. Required on iOS. No-op on Android. | — |

### Category: `OneSignal\|Subscription`

| Node | Type | Description | Inputs |
|------|------|-------------|--------|
| **Set Subscription** | Callable | Enable or disable push delivery for this device. | `bSubscribed` (bool) |

---

## Events & Delegates

Bind these via **Get OneSignal Manager → Assign Event** in Blueprint.

### `OnNotificationReceived`
Fired when a push notification arrives while the app is in the **foreground**.

```
Signature: (const FOneSignalNotification& Notification)
```

### `OnNotificationOpened`
Fired when the user **taps a push notification** to open the app.

```
Signature: (const FOneSignalNotificationAction& Action)
```

### `OnPlayerIdReceived`
Fired once the OneSignal **Player ID** becomes available after initialization.

```
Signature: (const FString& PlayerId)
```

---

## Data Structures

### `FOneSignalNotification`

| Field | Type | Description |
|-------|------|-------------|
| `Title` | `FString` | Notification title |
| `Body` | `FString` | Notification body / message text |
| `NotificationId` | `FString` | Unique OneSignal notification ID |
| `LargeIconUrl` | `FString` | Large icon URL attached to the notification |
| `AdditionalData` | `TMap<FString,FString>` | Custom key-value pairs sent with the notification |

### `FOneSignalNotificationAction`

| Field | Type | Description |
|-------|------|-------------|
| `Notification` | `FOneSignalNotification` | The notification that was interacted with |
| `ActionId` | `FString` | Action button ID pressed. Empty string = default tap. |

---

## Setup Guide

### Option A — Zero Code (Recommended)

1. Copy `OneSignalPro/` into your project's `Plugins/` folder
2. Regenerate project files and enable the plugin: **Edit → Plugins → Developer Tools → OneSignalPro**
3. Open **Edit → Project Settings → Plugins → OneSignal**
4. Paste your **OneSignal App ID** and enable **Auto-Initialize on Startup**
5. In your GameMode or PlayerController Blueprint:
   - Add **Event BeginPlay**
   - **Get OneSignal Manager**
   - Drag off → **Assign On Notification Received** → implement your logic
   - Drag off → **Assign On Notification Opened** → implement your logic
6. **iOS only:** Call **Prompt For Push Permissions** once (e.g. after tutorial)

### Option B — Manual Init from Blueprint

```
Event BeginPlay
    └─► Initialize OneSignal (AppId = "your-app-id")
            └─► Get OneSignal Manager
                    ├─► Assign On Notification Received
                    └─► Assign On Notification Opened
```

---

## Project Settings

Configure via **Edit → Project Settings → Plugins → OneSignal** or directly in `DefaultEngine.ini`:

```ini
[/Script/OneSignalPro.OneSignalSettings]
AppId=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
bEnableDebugLogs=False
bAutoInitialize=True
```

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `AppId` | String | `""` | Your OneSignal application ID |
| `bEnableDebugLogs` | bool | `false` | Enable verbose SDK logging |
| `bAutoInitialize` | bool | `true` | Initialize SDK automatically on game start |

---

## C++ Usage

### Access the Subsystem

```cpp
#include "OneSignalManager.h"

// From any UObject with a world context
UOneSignalManager* Manager = UGameInstance::GetSubsystem<UOneSignalManager>();

if (Manager && Manager->IsInitialized())
{
    // Tag the device
    Manager->SendTag(TEXT("level"), TEXT("5"));

    // Link to your backend user
    Manager->SetExternalUserId(TEXT("user-123"));

    // Bind notification event
    Manager->OnNotificationReceived.AddDynamic(
        this, &AMyGameMode::HandlePushNotification);
}
```

### Use the Static Blueprint Library from C++

```cpp
#include "OneSignalProBPLibrary.h"

// Works from any UObject — no subsystem reference needed
UOneSignalProBPLibrary::SendTag(this, TEXT("vip"), TEXT("true"));
UOneSignalProBPLibrary::SetSubscription(this, true);
FString PlayerId = UOneSignalProBPLibrary::GetPlayerId(this);
```

### Static Getter Helper

```cpp
// Convenience static getter
UOneSignalManager* Manager = UOneSignalManager::GetInstance(this);
```

---

## File Structure

```
Plugins/OneSignalPro/
├── OneSignalPro.uplugin
├── Resources/                          # Icon assets
└── Source/OneSignalPro/
    ├── OneSignalPro.Build.cs           # Module build rules + platform dependencies
    ├── OneSignalPro_APL.xml            # Android: Gradle dep, Manifest, Java injection
    ├── OneSignalPro_iOS_APL.xml        # iOS: plist entries, CocoaPod integration
    ├── Public/
    │   ├── OneSignalTypes.h            # FOneSignalNotification, FOneSignalNotificationAction
    │   ├── IOneSignalPlatform.h        # Abstract interface + native delegates
    │   ├── OneSignalSettings.h         # UDeveloperSettings (AppId, debug, auto-init)
    │   ├── OneSignalManager.h          # UGameInstanceSubsystem — main API
    │   ├── OneSignalProBPLibrary.h     # Static Blueprint wrapper
    │   └── OneSignalPro.h              # Module class + LogOneSignal log category
    └── Private/
        ├── OneSignalManager.cpp        # Platform selection + subsystem lifecycle
        ├── OneSignalPro.cpp            # IMPLEMENT_MODULE
        ├── Android/
        │   └── OneSignalAndroid.cpp   # JNI bridge (C++ ↔ Java ↔ OneSignal SDK 4.x)
        ├── IOS/
        │   └── OneSignalIOS.mm        # Objective-C++ wrapper (OneSignal SDK 3.x)
        └── Default/
            └── OneSignalDefault.h     # Null stub for editor + desktop platforms
```

---

## Technical Requirements

| Requirement | Value |
|-------------|-------|
| Unreal Engine | 4.27 or newer |
| Plugin Version | 1.0.0 |
| Module Type | Runtime |
| Loading Phase | PreLoadingScreen |
| Android SDK | OneSignal Android SDK 4.x (auto-resolved via Gradle) |
| iOS SDK | OneSignal iOS SDK 3.x (auto-resolved via CocoaPod) |
| Android Min API | API 21 (Android 5.0 Lollipop) |
| iOS Min Version | iOS 11.0 |
| Android Permissions | `WAKE_LOCK`, `VIBRATE`, `RECEIVE_BOOT_COMPLETED`, `ACCESS_NETWORK_STATE` |
| iOS Entitlements | Push Notifications, Background Modes (remote-notification) |
| Engine Source Modification | None — drop-in plugin only |
| OneSignal Account | Required (free tier available at [onesignal.com](https://onesignal.com)) |

---

## Support

- **Creator:** Alpha XP — [alpha-xp5-ai.github.io](https://alpha-xp5-ai.github.io)
- **OneSignal Documentation:** [documentation.onesignal.com](https://documentation.onesignal.com)
- **Issues / Feature Requests:** Submit via the FAB product page

---

*OneSignalPro v1.0.0 · Created by Alpha XP · Unreal Engine 4 Plugin · Android & iOS Push Notifications*
