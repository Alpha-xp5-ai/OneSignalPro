# OneSignalPro — UE4 Plugin

> Copyright 2026 Alpha XP. All Rights Reserved.

Production-ready OneSignal push notification integration for Unreal Engine 4 (Android & iOS).

- **Fab Marketplace:** `com.epicgames.launcher://ue/Fab/product/f43d88a0-e5db-48f0-9110-2a1d6f1ddcff`
- **Docs / Support:** https://alpha-xp5-ai.github.io/OneSignalPro/
- **Source:** https://github.com/Alpha-xp5-ai/OneSignalPro

---

## Requirements

| Item | Version |
|------|---------|
| Unreal Engine | 4.27+ |
| Target platforms | Android, iOS |
| OneSignal Android SDK | 4.x (added automatically via Gradle) |
| OneSignal iOS SDK | 3.x (added automatically via CocoaPods) |
| OneSignal account | Free — https://onesignal.com |

---

## Installation

1. Copy the `OneSignalPro` folder into your project's `Plugins/` directory.
2. Reopen the project — UE4 will prompt you to rebuild the plugin. Click **Yes**.
3. Enable the plugin in **Edit → Plugins → Developer Tools → OneSignalPro** (enabled by default after copying).

---

## Quick Setup (2 steps)

### Step 1 — Enter your App ID

Open **Edit → Project Settings → Plugins → OneSignal** and paste your OneSignal App ID.

> Find your App ID in the OneSignal dashboard: **Settings → Keys & IDs → OneSignal App ID**

```ini
; DefaultEngine.ini (written automatically by the Project Settings UI)
[/Script/OneSignalPro.OneSignalSettings]
AppId=YOUR_APP_ID_HERE
bEnableDebugLogs=False
bAutoInitialize=True
```

| Setting | Default | Description |
|---------|---------|-------------|
| `AppId` | _(empty)_ | Your OneSignal Application ID |
| `bAutoInitialize` | `true` | Initialize the SDK automatically when the game starts |
| `bEnableDebugLogs` | `false` | Enable verbose SDK logging (development only) |

### Step 2 — (Optional) Listen for events in Blueprint

If `bAutoInitialize` is **true** and `AppId` is set, no Blueprint code is required — the SDK initializes itself.

To react to incoming notifications, bind to the subsystem delegates:

```
[Begin Play]
    → Get Game Instance
    → Get Subsystem (OneSignal Manager)
    → Bind Event to On Notification Received
    → Bind Event to On Notification Opened
    → Bind Event to On Player ID Received
```

---

## API Reference

All functions are available in Blueprint under the **OneSignal** category, and in C++ via `UOneSignalManager` or `UOneSignalProBPLibrary`.

### Initialization

| Blueprint Node | C++ Method | Description |
|----------------|-----------|-------------|
| Initialize OneSignal | `InitializeOneSignal(AppId, bEnableDebug)` | Start the SDK. Skip if Auto-Initialize is on. |
| Is OneSignal Initialized | `IsInitialized()` | Returns `true` after a successful init. |

### Tagging (Segmentation)

| Blueprint Node | C++ Method | Description |
|----------------|-----------|-------------|
| Send Tag | `SendTag(Key, Value)` | Attach a single key/value tag to this device. |
| Send Tags | `SendTags(TMap<FString,FString>)` | Attach multiple tags in one call. |
| Delete Tag | `DeleteTag(Key)` | Remove a previously set tag. |

### User Identity

| Blueprint Node | C++ Method | Description |
|----------------|-----------|-------------|
| Set External User ID | `SetExternalUserId(Id)` | Link device to your own user ID (e.g. account ID). |
| Remove External User ID | `RemoveExternalUserId()` | Unlink the external user ID. |
| Get Player ID | `GetPlayerId()` | Returns the OneSignal device/player ID. Empty until the SDK registers. |

### Permissions & Subscription

| Blueprint Node | C++ Method | Description |
|----------------|-----------|-------------|
| Prompt For Push Permissions | `PromptForPushPermissions()` | Show the OS permission dialog. **Required on iOS** before notifications can arrive. No-op on Android. |
| Set Subscription | `SetSubscription(bSubscribed)` | Opt the device in (`true`) or out (`false`) of push notifications. |

### Events (Blueprint Delegates)

Bind these on the `UOneSignalManager` subsystem:

| Delegate | Parameters | Fired when |
|----------|-----------|-----------|
| `OnNotificationReceived` | `FOneSignalNotification` | A notification arrives while the app is **foreground**. |
| `OnNotificationOpened` | `FOneSignalNotificationAction` | The user **taps** a notification to open the app. |
| `OnPlayerIdReceived` | `FString PlayerId` | The OneSignal Player ID becomes available after init. |

#### FOneSignalNotification fields

| Field | Type | Description |
|-------|------|-------------|
| `Title` | `FString` | Notification title |
| `Body` | `FString` | Notification message body |
| `NotificationId` | `FString` | Unique OneSignal notification ID |
| `LargeIconUrl` | `FString` | URL of the large icon (Android) |
| `AdditionalData` | `TMap<FString,FString>` | Custom key-value payload sent with the notification |

#### FOneSignalNotificationAction fields

| Field | Type | Description |
|-------|------|-------------|
| `Notification` | `FOneSignalNotification` | The notification that was tapped |
| `ActionId` | `FString` | ID of the action button pressed; empty string = default tap |

---

## C++ Usage

### Get the subsystem

```cpp
#include "OneSignalManager.h"

// From any UObject with a world context (Actor, Widget, GameMode, etc.)
UOneSignalManager* OS = UOneSignalManager::GetInstance(this);
if (OS)
{
    OS->InitializeOneSignal(TEXT("YOUR_APP_ID"), false);
}
```

### Bind to delegates in C++

```cpp
UOneSignalManager* OS = UOneSignalManager::GetInstance(this);
if (OS)
{
    OS->OnNotificationReceived.AddDynamic(
        this, &AMyActor::HandleNotification);
    OS->OnPlayerIdReceived.AddDynamic(
        this, &AMyActor::HandlePlayerId);
}

void AMyActor::HandleNotification(const FOneSignalNotification& N)
{
    UE_LOG(LogTemp, Log, TEXT("Push received: %s — %s"), *N.Title, *N.Body);
}

void AMyActor::HandlePlayerId(const FString& PlayerId)
{
    UE_LOG(LogTemp, Log, TEXT("OneSignal Player ID: %s"), *PlayerId);
}
```

---

## Platform Notes

### Android

- Push permissions are granted at install time — no runtime prompt needed.
- The OneSignal SDK 4.x Gradle dependency is injected automatically via `OneSignalPro_APL.xml`.
- Required manifest permissions (`INTERNET`, `RECEIVE_BOOT_COMPLETED`, `VIBRATE`) are merged in automatically.

### iOS

- You **must** call **Prompt For Push Permissions** before notifications will appear.
  - Best practice: call it after a brief onboarding screen, not immediately at first launch.
- The OneSignal iOS SDK 3.x CocoaPod is added automatically via `OneSignalPro_iOS_APL.xml`.
- Ensure **Push Notifications** and **Background Modes → Remote notifications** capabilities are enabled in Xcode.

### Editor / Desktop (Win64, Mac, Linux)

The plugin compiles and loads cleanly on non-mobile platforms via a null stub — all API calls are no-ops. This lets you iterate in PIE without errors or special build flags.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Notifications not received on Android | Verify `AppId` in Project Settings. Run `adb logcat \| grep OneSignal`. Enable debug logs temporarily. |
| Notifications not received on iOS | Confirm you called **Prompt For Push Permissions** and the user accepted. Verify push capability in Xcode. |
| Player ID is empty | The ID arrives asynchronously — bind to `OnPlayerIdReceived` instead of reading it immediately after init. |
| SDK not initializing | Ensure `bAutoInitialize=true` and `AppId` is non-empty, or call **Initialize OneSignal** manually in Blueprint. |
| `C2065: 'GEngine': undeclared identifier` | Ensure `#include "Engine/Engine.h"` is present in the file that uses `GEngine`. |

---

## License

Copyright 2026 Alpha XP. All Rights Reserved.
