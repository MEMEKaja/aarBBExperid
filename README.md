<div align="center">

<img src="https://readme-typing-svg.demolab.com?font=Fira+Code&weight=700&size=32&pause=1000&color=00D4FF&center=true&vCenter=true&width=700&lines=BcoreModified+AAR;Virtual+Space+Engine+for+Android;Powered+by+BlackBox+Core" alt="Typing SVG" />

<br/>

[![Platform](https://img.shields.io/badge/Platform-Android%205.0%2B-brightgreen?style=for-the-badge&logo=android&logoColor=white)](https://developer.android.com)
[![ABI](https://img.shields.io/badge/ABI-arm64--v8a-blue?style=for-the-badge&logo=arm&logoColor=white)](https://developer.android.com/ndk/guides/abis)
[![minSdk](https://img.shields.io/badge/minSdk-21-orange?style=for-the-badge)](https://developer.android.com)
[![targetSdk](https://img.shields.io/badge/targetSdk-36-red?style=for-the-badge)](https://developer.android.com)
[![License](https://img.shields.io/badge/License-Private-black?style=for-the-badge)](.)

<br/>

```
  ██████╗  ██████╗ ██████╗ ██████╗ ███████╗
  ██╔══██╗██╔════╝██╔═══██╗██╔══██╗██╔════╝
  ██████╔╝██║     ██║   ██║██████╔╝█████╗  
  ██╔══██╗██║     ██║   ██║██╔══██╗██╔══╝  
  ██████╔╝╚██████╗╚██████╔╝██║  ██║███████╗
  ╚═════╝  ╚═════╝ ╚═════╝ ╚═╝  ╚═╝╚══════╝
         M O D I F I E D   A A R
```

**A modified BlackBox virtual space engine packaged as an Android AAR library.**
*Install, clone, run, hide — all in one drop-in dependency.*

</div>

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Integration](#integration)
  - [Add the AAR](#add-the-aar)
  - [Configure Dependencies](#configure-dependencies)
  - [AndroidManifest Setup](#androidmanifest-setup)
- [Initialization](#initialization)
- [Feature Reference](#feature-reference)
  - [Install & Uninstall App](#install--uninstall-app)
  - [Clone App (Multi-User)](#clone-app-multi-user)
  - [Launch App](#launch-app)
  - [Hide Root](#hide-root)
  - [Hide Xposed](#hide-xposed)
  - [On / Off Hook Toggle](#on--off-hook-toggle)
  - [Device ID Injection](#device-id-injection)
  - [Bypass Engine](#bypass-engine)
  - [Xposed Module Support](#xposed-module-support)
  - [GMS / Google Play Support](#gms--google-play-support)
  - [App Lifecycle Callbacks](#app-lifecycle-callbacks)
  - [Package Management Utilities](#package-management-utilities)
- [ClientConfiguration Reference](#clientconfiguration-reference)
- [Architecture Notes](#architecture-notes)
- [ProGuard Rules](#proguard-rules)

---

## Overview

`BcoreModified` is a pre-built Android AAR wrapping a heavily modified **BlackBox** virtual engine. It provides a fully isolated process space where target apps run sandboxed — without being installed on the real system. The library intercepts Android framework calls via Binder proxies, JNI hooks, and native IO redirection, enabling features like:

- Running multiple instances of the same app (cloning)
- Hiding root and Xposed detection
- Injecting stable device identifiers into sandboxed apps
- Executing a native ELF payload with a delayed bypass sequence after app launch
- Full Xposed module support inside the virtual space

---

## Requirements

| Component | Requirement |
|---|---|
| Android API | 21+ (Android 5.0 Lollipop) |
| Target API | 36 |
| ABI | `arm64-v8a` only |
| NDK | 24.0.8215888 or compatible |
| Java | 11+ |
| Gradle | 7.4.2+ |
| Build Tools | 34.0.4 |

---

## Integration

### Add the AAR

Copy `BcoreModified-release.aar` (or the debug variant) into your project's `libs/` folder:

```
YourApp/
  app/
    libs/
      BcoreModified-release.aar   <-- place here
    build.gradle
```

### Configure Dependencies

In your **app-level** `build.gradle`:

```groovy
android {
    defaultConfig {
        ndk {
            abiFilters "arm64-v8a"
        }
        multiDexEnabled true
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_11
        targetCompatibility JavaVersion.VERSION_11
    }
}

dependencies {
    // BcoreModified AAR
    implementation fileTree(dir: "libs", include: ["*.aar", "*.jar"])

    // Required transitive dependencies
    implementation 'androidx.appcompat:appcompat:1.2.0'
    implementation 'com.github.tiann:FreeReflection:3.2.2'
    implementation 'com.github.CodingGay.BlackReflection:core:1.1.4'
    annotationProcessor 'com.github.CodingGay.BlackReflection:compiler:1.1.4'
    implementation 'net.lingala.zip4j:zip4j:2.11.5'
}
```

In your **project-level** `build.gradle` or `settings.gradle`, ensure JitPack is available:

```groovy
repositories {
    google()
    mavenCentral()
    maven { url 'https://jitpack.io' }
}
```

### AndroidManifest Setup

The AAR ships with its own `AndroidManifest.xml` containing all required proxy activities, services, and providers. The Gradle merger handles this automatically. However, you **must** declare these permissions in your own manifest:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
```

---

## Initialization

Initialize BcoreModified inside your `Application` class. You must override **both** `attachBaseContext` and `onCreate`.

```java
import top.niunaijun.blackbox.BlackBoxCore;
import top.niunaijun.blackbox.app.configuration.ClientConfiguration;

public class MyApplication extends Application {

    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);

        // Build your configuration
        BlackBoxCore.get().doAttachBaseContext(base, new ClientConfiguration() {

            @Override
            public String getHostPackageName() {
                // Return YOUR app's package name
                return "com.yourapp.packagename";
            }

            @Override
            public boolean isHideRoot() {
                // true = hide root from apps running inside the virtual space
                return true;
            }

            @Override
            public boolean isHideXposed() {
                // true = hide Xposed framework from sandboxed apps
                return true;
            }

            @Override
            public boolean isEnableDaemonService() {
                // true = start a persistent daemon process for stability
                return true;
            }

            @Override
            public boolean isEnableLauncherActivity() {
                // true = use built-in LauncherActivity as transition screen
                return true;
            }

            @Override
            public boolean requestInstallPackage(File file, int userId) {
                // Called when a sandboxed app requests to install another APK
                // Return true if you handled it, false to deny
                return false;
            }
        });
    }

    @Override
    public void onCreate() {
        super.onCreate();
        BlackBoxCore.get().doCreate();
    }
}
```

> **Important:** `doAttachBaseContext()` must be called before `super.attachBaseContext()` would normally run. The `ClientConfiguration` you pass here is the single source of truth for all feature flags.

---

## Feature Reference

### Install & Uninstall App

Install a target app into the virtual space. The app does not get installed on the real Android system.

```java
// Install from package name of an already-installed real app (copy into virtual space)
InstallResult result = BlackBoxCore.get().installPackageAsUser("com.target.app", 0);

// Install from an APK File object
InstallResult result = BlackBoxCore.get().installPackageAsUser(new File("/sdcard/target.apk"), 0);

// Install from a Uri (e.g. from a file picker)
InstallResult result = BlackBoxCore.get().installPackageAsUser(uri, 0);

// Check result
if (result.success) {
    Log.d("BCore", "Installed: " + result.packageName);
} else {
    Log.e("BCore", "Failed: " + result.error);
}

// Uninstall from a specific user slot
BlackBoxCore.get().uninstallPackageAsUser("com.target.app", 0);

// Uninstall from all user slots
BlackBoxCore.get().uninstallPackage("com.target.app");

// Check if already installed in a slot
boolean installed = BlackBoxCore.get().isInstalled("com.target.app", 0);
```

---

### Clone App (Multi-User)

BcoreModified supports running multiple isolated instances of the same app via a virtual user system. Each `userId` represents a separate clone slot.

```java
// Create a new virtual user (clone slot)
BUserInfo user = BlackBoxCore.get().createUser(1); // userId = 1

// Get all existing virtual users
List<BUserInfo> users = BlackBoxCore.get().getUsers();
for (BUserInfo u : users) {
    Log.d("BCore", "User id=" + u.id + " name=" + u.name);
}

// Install the same app into multiple slots
BlackBoxCore.get().installPackageAsUser("com.target.app", 0); // first clone
BlackBoxCore.get().installPackageAsUser("com.target.app", 1); // second clone
BlackBoxCore.get().installPackageAsUser("com.target.app", 2); // third clone

// Launch a specific clone
BlackBoxCore.get().launchApk("com.target.app", 0); // open first clone
BlackBoxCore.get().launchApk("com.target.app", 1); // open second clone

// Delete a virtual user and all its data
BlackBoxCore.get().deleteUser(1);
```

> Each user slot is fully isolated. Data, accounts, and sessions do not bleed between slots.

---

### Launch App

```java
// Launch by package name in a specific user slot
boolean launched = BlackBoxCore.get().launchApk("com.target.app", 0);

// Launch with a custom Intent
Intent intent = new Intent(Intent.ACTION_MAIN);
intent.setPackage("com.target.app");
BlackBoxCore.get().startActivity(intent, 0);

// Stop a running app without uninstalling
BlackBoxCore.get().stopPackage("com.target.app", 0);

// Clear app data (cache + storage) without uninstalling
BlackBoxCore.get().clearPackage("com.target.app", 0);
```

---

### Hide Root

Root hiding is configured at initialization time via `ClientConfiguration.isHideRoot()`. When enabled, the virtual engine intercepts filesystem probes, `/proc` reads, and su binary lookups made by apps running inside the space.

```java
// Enable at init time — see Initialization section
@Override
public boolean isHideRoot() {
    return true; // ON: sandboxed apps cannot detect root
}
```

```java
// You can also query the current state at runtime
boolean rootHidden = BlackBoxCore.get().isHideRoot();
```

To **disable** root hiding for a specific build:

```java
@Override
public boolean isHideRoot() {
    return false; // OFF: root is visible inside virtual space
}
```

> Root hiding is enforced at the native IO redirection layer and the JNI hook layer simultaneously.

---

### Hide Xposed

Same pattern as root hiding. When enabled, Xposed-specific artifacts (`XposedBridge.jar`, `de.robv.android.xposed`, exposed API classes) are hidden from apps inside the virtual space.

```java
@Override
public boolean isHideXposed() {
    return true; // ON
}
```

```java
// Runtime query
boolean xposedHidden = BlackBoxCore.get().isHideXposed();
```

---

### On / Off Hook Toggle

The hook system initializes automatically in `doAttachBaseContext()`. You do not manually call hook init. However, you can observe the process type to understand what hooks are active in the current process:

```java
// Is this the main host process?
boolean isMain = BlackBoxCore.get().isMainProcess();

// Is this the virtual space server process?
boolean isServer = BlackBoxCore.get().isServerProcess();

// Is this a sandboxed app process (where hooks are active)?
boolean isSandboxed = BlackBoxCore.get().isBlackProcess();
```

Hooks are active **only** when `isBlackProcess()` is `true`. They are never applied in your host app's main process, keeping your app clean.

---

### Device ID Injection

The AAR includes `DeviceIdInjector` which pre-populates stable SharedPreferences device identifiers before a sandboxed app launches. This is critical for apps that rely on the ByteDance AppLog SDK (e.g. MLBB) — without it, those apps fail to initialize with `"init fail empty field: appkey"`.

This runs automatically inside `launchApk()`. No manual call required.

```java
// Runs automatically — called internally before every launchApk()
// DeviceIdInjector.inject(packageName, userId);
```

The generated ID is a stable MD5 hash of `hostPackage + targetPackage + userId`. It is consistent across sessions and unique per clone slot.

---

### Bypass Engine

After an app is launched via `launchApk()`, the bypass engine fires a delayed sequence that executes a native ELF binary embedded in the AAR at `res/raw/temp`. The binary is copied to the app's private data directory on first run and executed with staged delays.

```java
// Called automatically inside launchApk() — no manual call needed
// Fires at +15s, +45s, and +83s after launch

// You can also trigger it manually if needed
BlackBoxCore.get().bypass();

// Or execute a specific binary from the virtual space cache
BlackBoxCore.get().excpp("/blackbox/cache/temp 992");
```

The binary path resolves to:
```
/data/data/<your_host_package>/blackbox/cache/temp
```

---

### Xposed Module Support

You can load Xposed modules into the virtual space and enable them per-module. Modules run inside the sandbox and only affect apps launched within it.

```java
// Install an Xposed module from APK file
InstallResult result = BlackBoxCore.get().installXPModule(new File("/sdcard/MyModule.apk"));

// Install from Uri
InstallResult result = BlackBoxCore.get().installXPModule(uri);

// Install from an already-installed real package
InstallResult result = BlackBoxCore.get().installXPModule("io.my.xposed.module");

// Enable/disable the Xposed framework globally
BlackBoxCore.get().setXPEnable(true);
boolean xpEnabled = BlackBoxCore.get().isXPEnable();

// Enable/disable a specific module
BlackBoxCore.get().setModuleEnable("io.my.xposed.module", true);
boolean moduleOn = BlackBoxCore.get().isModuleEnable("io.my.xposed.module");

// List all installed modules
List<InstalledModule> modules = BlackBoxCore.get().getInstalledXPModules();

// Uninstall a module
BlackBoxCore.get().uninstallXPModule("io.my.xposed.module");

// Check if a file is a valid Xposed module
boolean isModule = BlackBoxCore.get().isXposedModule(new File("/sdcard/something.apk"));

// Check if a module is installed
boolean installed = BlackBoxCore.get().isInstalledXposedModule("io.my.xposed.module");
```

---

### GMS / Google Play Support

```java
// Check if the device supports GMS virtualisation
boolean gmsSupported = BlackBoxCore.get().isSupportGms();

// Install Google Play Services into a user slot
if (gmsSupported) {
    InstallResult result = BlackBoxCore.get().installGms(0);
}

// Check if GMS is installed in a slot
boolean gmsInstalled = BlackBoxCore.get().isInstallGms(0);

// Uninstall GMS from a slot
BlackBoxCore.get().uninstallGms(0);
```

---

### App Lifecycle Callbacks

Hook into the lifecycle of apps running inside the virtual space.

```java
BlackBoxCore.get().addAppLifecycleCallback(new AppLifecycleCallback() {

    @Override
    public void beforeMainLaunchApk(String packageName, int userId) {
        // Called before the target app is launched
        Log.d("BCore", "About to launch: " + packageName + " user=" + userId);
    }

    @Override
    public void beforeMainApplicationAttach(Application app, Context context) {
        // Called before the virtual app's Application.attachBaseContext()
    }

    @Override
    public void afterMainApplicationAttach(Application app, Context context) {
        // Called after the virtual app's Application.attachBaseContext()
    }

    @Override
    public void beforeMainActivityOnCreate(android.app.Activity activity) {
        // Called before the virtual app's first Activity.onCreate()
    }

    @Override
    public void afterMainActivityOnCreate(android.app.Activity activity) {
        // Called after the virtual app's first Activity.onCreate()
    }
});

// Remove a specific callback
BlackBoxCore.get().removeAppLifecycleCallback(myCallback);

// Retrieve all registered callbacks
List<AppLifecycleCallback> all = BlackBoxCore.get().getAppLifecycleCallbacks();
```

---

### Package Management Utilities

```java
// List all apps installed in a virtual user slot
List<ApplicationInfo> apps = BlackBoxCore.get().getInstalledApplications(0, 0);

// List all packages with full PackageInfo
List<PackageInfo> pkgs = BlackBoxCore.get().getInstalledPackages(0, 0);

// Low-level service access (advanced)
BPackageManager pm = BlackBoxCore.getBPackageManager();
BActivityManager am = BlackBoxCore.getBActivityManager();
BStorageManager sm = BlackBoxCore.getBStorageManager();
BJobManager jm     = BlackBoxCore.getBJobManager();
```

---

## ClientConfiguration Reference

| Method | Default | Description |
|---|---|---|
| `getHostPackageName()` | *abstract* | Must return your host app's package name |
| `isHideRoot()` | `false` | Hide root detection inside virtual space |
| `isHideXposed()` | `false` | Hide Xposed artifacts inside virtual space |
| `isEnableDaemonService()` | `true` | Enable persistent daemon service for stability |
| `isEnableLauncherActivity()` | `true` | Use built-in loading screen when launching apps |
| `requestInstallPackage(File, int)` | `false` | Handle install requests from sandboxed apps |

---

## Architecture Notes

```
Your App (host process)
    │
    ├── BlackBoxCore.doAttachBaseContext()   — sets up config, hooks HookManager
    ├── BlackBoxCore.doCreate()              — starts ServiceManager, copies native binary
    │
    ├── [Server Process] :blackbox_server
    │       └── BActivityManagerService, BPackageManagerService, etc.
    │
    └── [Sandboxed App Process]
            ├── Binder proxies active (IActivityManagerProxy, IPackageManagerProxy, ...)
            ├── Native IO redirection (addIORule via JNI → libnative.so)
            ├── DexFileHook, RuntimeHook, VMClassLoaderHook
            └── DeviceIdInjector → stable SharedPreferences before app init
```

The native layer (`lib@ace_finder.so`) is extracted from AAR assets and loaded via `System.load()` at runtime. The `@` character in the filename makes it undetectable by standard `loadLibrary` scanners.

---

## ProGuard Rules

The AAR ships with `consumer-rules.pro`. If you have additional minification rules, ensure you **do not** strip the following:

```proguard
-keep class top.niunaijun.blackbox.** { *; }
-keep class black.** { *; }
-keep class android.app.ActivityThread { *; }
-keepclassmembers class * {
    @androidx.annotation.Keep *;
}
```

---

<div align="center">

---

*BcoreModified is a private modified build. Do not redistribute.*

[![Made with pain](https://img.shields.io/badge/Made%20with-pain%20%26%20caffeine-red?style=flat-square)](.)
[![Virtual Space](https://img.shields.io/badge/Engine-BlackBox%20Modified-00D4FF?style=flat-square)](.)
[![arch](https://img.shields.io/badge/arch-arm64--v8a-222?style=flat-square&logo=arm)](.)

</div>
