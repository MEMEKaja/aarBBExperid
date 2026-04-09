# Bcore AAR — @ace_finder Edition

> Virtual App framework berbasis BlackBox/VirtualApp dengan sistem hook native (C++/NDK), IO redirection, Xposed bypass, dan seccomp filter.
> Library ini di-output sebagai **AAR** siap pakai untuk project Android.

---

## 📋 Daftar Isi

- [Requirement](#-requirement)
- [Clone & Build](#-clone--build)
- [Integrasi ke Project](#-integrasi-ke-project)
- [Inisialisasi](#-inisialisasi)
- [IO Redirection](#-io-redirection--addiorule)
- [Hide Xposed](#-hide-xposed)
- [Disable Hidden API](#-disable-hidden-api)
- [Seccomp Filter](#-seccomp-filter)
- [Load libbgmi.so (Mod Library)](#-load-libbgmiso-mod-library)
- [Hide Root / Magisk](#-hide-root--magisk)
- [Struktur Output AAR](#-struktur-output-aar)
- [Troubleshooting](#-troubleshooting)

---

## ✅ Requirement

| Tool | Versi |
|------|-------|
| Android Studio / AndroidIDE | Bumblebee+ |
| Gradle | **7.5** |
| Android Gradle Plugin (AGP) | **7.4.2** |
| NDK | **24.0.8215888** |
| compileSdk | 34 |
| minSdk | 21 |
| Target ABI | `arm64-v8a` |
| Java | 11 |

> **Termux / AndroidIDE:** pastikan sudah install NDK via SDK Manager dan path NDK sudah benar di `local.properties`.

---

## 📦 Clone & Build

### 1. Clone / Extract Project

Jika dari zip:
```bash
unzip Bcore_StatusCheck.zip -d MyProject
cd MyProject/BcoreModified
```

Jika dari git:
```bash
git clone <repo-url>
cd BcoreModified
```

### 2. Pastikan file konfigurasi ada

Cek bahwa file-file ini ada di root folder:
```
BcoreModified/
├── build.gradle
├── settings.gradle        ← wajib ada
├── gradle.properties      ← wajib ada
├── gradle/wrapper/
│   └── gradle-wrapper.properties   (distributionUrl=gradle-7.5-bin.zip)
└── src/
```

### 3. Build AAR

```bash
# Debug build
bash gradlew assembleDebug

# Release build
bash gradlew assembleRelease
```

Output AAR ada di:
```
build/outputs/aar/BcoreModified-debug.aar
build/outputs/aar/BcoreModified-release.aar
```

> Setelah build selesai, Gradle task otomatis rename `libace_finder.so` → `lib@ace_finder.so` di dalam AAR.

---

## 🔗 Integrasi ke Project

### Step 1 — Copy AAR ke project kamu

```
YourApp/
└── app/
    └── libs/
        └── BcoreModified-debug.aar   ← taruh di sini
```

### Step 2 — Tambahkan di `app/build.gradle`

```groovy
android {
    // ...
    
    // Wajib untuk Virtual App
    packagingOptions {
        jniLibs {
            useLegacyPackaging = true
        }
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.aar', '*.jar'])
    
    // Dependencies yang dibutuhkan Bcore
    implementation 'androidx.appcompat:appcompat:1.2.0'
    implementation 'com.github.tiann:FreeReflection:3.2.2'
    implementation 'com.github.CodingGay.BlackReflection:core:1.1.4'
    annotationProcessor 'com.github.CodingGay.BlackReflection:compiler:1.1.4'
    implementation 'net.lingala.zip4j:zip4j:2.11.5'
}
```

### Step 3 — `settings.gradle` project utama

```groovy
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

### Step 4 — `gradle.properties` project utama

```properties
android.useAndroidX=true
android.enableJetifier=true
```

### Step 5 — AndroidManifest.xml

Tambahkan permission yang dibutuhkan:
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />

<!-- Untuk Virtual App menjalankan game -->
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

---

## 🚀 Inisialisasi

Inisialisasi **wajib** dilakukan di `Application.onCreate()` sebelum fungsi lainnya dipanggil.

```java
public class MyApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        
        // 1. Init BlackBoxCore — ini entry point utama
        BlackBoxCore.get().doAttachBaseContext(this, true);
    }
}
```

Daftarkan di `AndroidManifest.xml`:
```xml
<application
    android:name=".MyApplication"
    ... >
```

### Inisialisasi Native Layer

BlackBoxCore secara otomatis memanggil `NativeCore.init()` di dalam prosesnya. Tapi jika kamu perlu manual:

```java
// Dipanggil otomatis dari BlackBoxCore, tidak perlu manual
// kecuali kamu extend AppLifecycleCallback
NativeCore.init(Build.VERSION.SDK_INT);
```

---

## 📂 IO Redirection — `addIORule`

IO Redirection mengalihkan akses file game ke path lain (misal ke storage virtual BlackBox), sehingga game tidak bisa detect path asli.

### Cara aktifkan IO

```java
// Aktifkan IO hook — wajib sebelum addIORule
NativeCore.enableIO();
```

### Tambah rule redirect

```java
// addIORule(targetPath, relocatePath)
// Semua akses ke targetPath akan diarahkan ke relocatePath

// Contoh: redirect folder data game ke storage virtual
NativeCore.addIORule(
    "/data/data/com.tencent.ig",           // path asli yang dicari game
    BlackBoxCore.getBBlackBoxDir() + "/data/com.tencent.ig"  // path tujuan
);

// Contoh: sembunyikan file Magisk
NativeCore.addIORule("/sbin/.magisk", "/dev/null");
NativeCore.addIORule("/data/adb/magisk", "/dev/null");

// Contoh: redirect proc/self/root
NativeCore.addIORule("/proc/self/root/sbin", "/dev/null");
```

### Cara off IO Redirection

IO redirection tidak ada fungsi `disable` karena di-init sekali per proses. Untuk menonaktifkan, **jangan panggil** `NativeCore.enableIO()` di konfigurasi kamu, atau gunakan flag boolean sendiri:

```java
boolean enableIORedirect = true; // set false untuk off

if (enableIORedirect) {
    NativeCore.enableIO();
    NativeCore.addIORule("/sbin/.magisk", "/dev/null");
    // tambah rule lainnya...
}
```

---

## 🕵️ Hide Xposed

Menyembunyikan keberadaan Xposed/LSPosed dari deteksi game.

```java
// Aktifkan hide Xposed
NativeCore.hideXposed();
```

### Cara on/off

```java
boolean hideXposed = true; // ubah ke false untuk disable

if (hideXposed) {
    NativeCore.hideXposed();
}
```

> **Catatan:** `hideXposed()` bekerja di level ClassLoader hook — dia menyembunyikan class Xposed dari daftar ClassLoader game. Efektif untuk bypass deteksi `de.robv.android.xposed.XposedBridge` dan sejenisnya.

---

## 🔓 Disable Hidden API

Android 9+ membatasi akses ke API internal via reflection. Fungsi ini menonaktifkan batasan tersebut.

```java
boolean success = NativeCore.disableHiddenApi();
if (!success) {
    Log.e(TAG, "disableHiddenApi gagal");
}
```

### Cara on/off

```java
boolean disableHiddenApi = true;

if (disableHiddenApi) {
    NativeCore.disableHiddenApi();
}
```

> Biasanya dipanggil sekali di awal sebelum reflection ke API internal Android.

---

## 🔒 Seccomp Filter

Seccomp (Secure Computing) filter digunakan untuk intercept syscall `openat` — memungkinkan kontrol file access di level kernel tanpa root.

```java
// Aktifkan seccomp filter
NativeCore.init_seccomp();
```

### Cara on/off

```java
boolean useSeccomp = true;

if (useSeccomp) {
    NativeCore.init_seccomp();
}
```

> **Peringatan:** Seccomp bersifat permanen per-proses setelah diaktifkan. Pastikan kamu sudah setup signal handler dengan benar sebelum enable. Jika tidak perlu, cukup jangan panggil fungsi ini.

---

## 💉 Load libbgmi.so (Mod Library)

Bcore mendukung loading library mod tambahan (`libbgmi.so`) dari `filesDir` app. Library ini bisa kamu inject setelah BlackBox mengeksekusi game.

### Cara load

```java
// Copy libbgmi.so ke filesDir terlebih dahulu
File modLib = new File(getContext().getFilesDir(), "libbgmi.so");

// Contoh: copy dari assets
try (InputStream in = getAssets().open("libbgmi.so");
     OutputStream out = new FileOutputStream(modLib)) {
    byte[] buf = new byte[4096];
    int len;
    while ((len = in.read(buf)) > 0) out.write(buf, 0, len);
}

// Library akan otomatis di-load oleh NativeCore saat inisialisasi
// (sudah ada di static block NativeCore.java)
```

### Mekanisme otomatis

`NativeCore` secara otomatis cek dan load `libbgmi.so` dari `filesDir`:

```java
// Ini sudah ada di NativeCore.java — tidak perlu kamu tambahkan
File libFile = new File(BlackBoxCore.getContext().getFilesDir(), "libbgmi.so");
if (libFile.exists()) {
    System.load(libFile.getAbsolutePath());
}
```

### Cara on/off load mod library

```java
// Untuk disable: hapus/jangan copy libbgmi.so ke filesDir
// Atau tambahkan flag di AppLifecycleCallback:

@Override
public void beforeStartApplication(String packageName, String processName, Context context) {
    boolean loadMod = true; // set false untuk skip
    
    if (loadMod) {
        // copy libbgmi.so ke filesDir context game
        copyModLibToProcess(context);
    }
}
```

---

## 🫥 Hide Root / Magisk

Kombinasi IO redirection + seccomp untuk menyembunyikan tanda-tanda root dari deteksi game.

### Setup lengkap Hide Root

```java
public void setupHideRoot() {
    // 1. Aktifkan IO hook
    NativeCore.enableIO();
    
    // 2. Redirect path-path yang dicek game untuk deteksi root/magisk
    
    // Magisk paths
    NativeCore.addIORule("/sbin/.magisk", "/dev/null");
    NativeCore.addIORule("/sbin/.core/mirror", "/dev/null");
    NativeCore.addIORule("/sbin/.core/img", "/dev/null");
    NativeCore.addIORule("/data/adb/magisk", "/dev/null");
    NativeCore.addIORule("/data/adb/magisk.img", "/dev/null");
    NativeCore.addIORule("/cache/magisk.log", "/dev/null");
    
    // Su binary paths
    NativeCore.addIORule("/sbin/su", "/dev/null");
    NativeCore.addIORule("/system/bin/su", "/dev/null");
    NativeCore.addIORule("/system/xbin/su", "/dev/null");
    NativeCore.addIORule("/system/sbin/su", "/dev/null");
    NativeCore.addIORule("/vendor/bin/su", "/dev/null");
    
    // proc/self paths yang dicek anti-cheat
    NativeCore.addIORule("/proc/self/root/sbin", "/dev/null");
    NativeCore.addIORule("/proc/self/root/data/adb", "/dev/null");
    
    // Busybox
    NativeCore.addIORule("/system/xbin/busybox", "/dev/null");
    NativeCore.addIORule("/system/bin/busybox", "/dev/null");
}
```

### Cara on/off Hide Root

```java
// Di AppLifecycleCallback atau Application
boolean hideRoot = true;

if (hideRoot) {
    setupHideRoot();
}
```

### AppLifecycleCallback — tempat terbaik untuk setup

```java
public class MyAppCallback extends AppLifecycleCallback {

    @Override
    public void beforeStartApplication(String packageName, String processName, Context context) {
        // Dipanggil sebelum game process start
        // Ini tempat terbaik untuk setup semua hook
        
        NativeCore.disableHiddenApi();
        NativeCore.hideXposed();
        
        if (BuildConfig.ENABLE_IO) {
            NativeCore.enableIO();
            setupIORules();
        }
        
        if (BuildConfig.ENABLE_SECCOMP) {
            NativeCore.init_seccomp();
        }
    }
    
    private void setupIORules() {
        NativeCore.addIORule("/sbin/.magisk", "/dev/null");
        NativeCore.addIORule("/data/adb/magisk", "/dev/null");
        // ... rule lainnya
    }
}
```

Daftarkan callback:
```java
BlackBoxCore.get().setClientConfiguration(new ClientConfiguration() {
    @Override
    public AppLifecycleCallback getAppLifecycleCallback() {
        return new MyAppCallback();
    }
});
```

---

## 📁 Struktur Output AAR

Setelah build, isi dalam AAR:

```
BcoreModified-debug.aar
├── AndroidManifest.xml
├── classes.jar                    ← semua Java class
├── jni/
│   └── arm64-v8a/
│       └── lib@ace_finder.so     ← native library (sudah di-rename)
├── assets/
│   └── jniLibs/
│       └── arm64-v8a/
│           └── lib@ace_finder.so ← copy untuk load via System.load()
├── res/
├── R.txt
└── proguard.txt
```

---

## 🛠 Troubleshooting

### ❌ `Plugin 'com.android.library' not found`
→ Pastikan `settings.gradle` ada di root folder dan berisi `pluginManagement` dengan repo google.

### ❌ `Minimum supported Gradle version is 7.5`
→ Edit `gradle/wrapper/gradle-wrapper.properties`:
```properties
distributionUrl=https\://services.gradle.org/distributions/gradle-7.5-bin.zip
```

### ❌ `android.useAndroidX property is not enabled`
→ Tambahkan di `gradle.properties`:
```properties
android.useAndroidX=true
android.enableJetifier=true
```

### ❌ `UnsatisfiedLinkError: lib@ace_finder.so`
→ Pastikan `SYSTEM_ALERT_WINDOW` granted. Library di-load via `System.load()` dari `filesDir` — pastikan filesDir writable.

### ❌ `NativeCore.init()` crash / NPE
→ Pastikan `BlackBoxCore.get().doAttachBaseContext()` dipanggil lebih dulu di `Application.onCreate()`.

### ❌ Build C++ gagal: NDK not found
→ Di `local.properties`:
```properties
ndk.dir=/path/to/ndk/24.0.8215888
sdk.dir=/path/to/sdk
```

### ❌ `enableIO()` tidak efektif
→ Pastikan `enableIO()` dipanggil **sebelum** `addIORule()`. Urutan harus:
```java
NativeCore.enableIO();       // 1. dulu ini
NativeCore.addIORule(...);   // 2. baru ini
```

---

## 📌 Catatan Penting

- **ABI:** saat ini hanya support `arm64-v8a`. Untuk tambah `armeabi-v7a`, edit `abiFilters` di `build.gradle` dan rebuild.
- **Seccomp** bersifat **irreversible** per proses — jangan panggil jika tidak diperlukan.
- **IO rules** berlaku untuk semua thread dalam proses yang sama.
- Library `lib@ace_finder.so` menggunakan karakter `@` — tidak bisa di-`loadLibrary()`, harus lewat `System.load()` dengan path lengkap (sudah ditangani otomatis oleh `NativeCore`).

---

> Update by **@ace_finder**
