# SnapEnhance Fix for Snapchat 14.13.0.45

This repository documents the analysis, diagnosis, and fix for the instant crash occurring on rooted Android devices (KernelSU/LSPosed).

### 📋 Version Info
* **Snapchat:** `14.13.0.45` (arm64-v8a)
* **SnapEnhance:** `2.2.0` (patched version `1.0.1`)

---

## 🚨 The Problems & Fixes

### 1. Instant Startup Crash (Native Segfault)
* **Symptom:** Snapchat crashes immediately with exit code `139` (**`SIGSEGV`**).
* **Cause:** `PlatformClientAttestationMapper` maps the native attestation class to `jc.tyqs`. In Snapchat `14.13.0.45`, the structures changed, and hooking it dynamically caused memory corruption.
* **Fix:** We patched `ClassMapper$Companion.smali` to remove `PlatformClientAttestationMapper` from the mappers array, disabling the native hook while keeping all other features active.

### 2. "Failed to init feature half swipe notifier" Toast Warning
* **Symptom:** A toast warning pops up on startup saying *"Failed to init feature half swipe notifier"*.
* **Cause:** In Snapchat `14.13.0.45`, the old talk/presence package (`com.snapchat.talkcorev3`) has been completely removed/renamed to `com.snap.presence`. The feature's `init()` tried to load `com.snapchat.talkcorev3.PresenceService$CppProxy` which throws a `ClassNotFoundException`, crashing the initialization lifecycle.
* **Fix:** We wrapped the initialization logic of `HalfSwipeNotifier` in a try-catch block inside its Smali file to safely ignore this exception on newer Snapchat versions.

---

## 🔍 How We Isolated the Culprits

We used custom Python diagnostic scripts executing on the device to binary-search and isolate the crash-inducing hooks:
1. Checked root mapping keys in `mappings.json` and isolated the startup crash to `PlatformClientAttestationMapper`.
2. Monitored logcat output for Snapchat process PID to trace the `ClassNotFoundException` thrown by the `HalfSwipeNotifier` feature initialization.

---

## 🛠️ Step-by-Step Patching Guide

### Step 1: Decompile SnapEnhance APK
```bash
apktool d -o apktool-out snapenhance.apk
```

### Step 2: Remove PlatformClientAttestationMapper (Fix Crash)
1. Open `apktool-out/smali/me/rhunk/snapenhance/mapper/ClassMapper$Companion.smali`.
2. Locate `getDEFAULT_MAPPERS()[Lme/rhunk/snapenhance/mapper/AbstractClassMapper;`.
3. Change the array size from `19` (`0x13`) to `18` (`0x12`):
   ```smali
   const/16 v15, 0x12
   new-array v15, v15, [Lme/rhunk/snapenhance/mapper/AbstractClassMapper;
   ```
4. Delete the lines instantiating `PlatformClientAttestationMapper`:
   ```smali
   new-instance v18, Lme/rhunk/snapenhance/mapper/impl/PlatformClientAttestationMapper;
   invoke-direct/range {v18 .. v18}, Lme/rhunk/snapenhance/mapper/impl/PlatformClientAttestationMapper;-><init>()V
   ```
5. Remove the array assignment line:
   ```smali
   const/16 v0, 0x12
   aput-object v18, v15, v0
   ```

### Step 3: Try-Catch HalfSwipeNotifier (Fix Toast Warning)
1. Open `apktool-out/smali/me/rhunk/snapenhance/core/features/impl/spying/HalfSwipeNotifier.smali`.
2. Locate `.method public init()V`.
3. Wrap the main body under `:cond_0` in a try-catch block:
   ```smali
       :cond_0
       :try_start_0
       new-instance v0, Lkotlin/jvm/internal/x;
       # ... [Existing setup and mapping calls] ...
       invoke-virtual {v1, v2, v3}, Lme/rhunk/snapenhance/common/bridge/wrapper/MappingsWrapper;->useMapper(LT2/c;LM2/c;)V
       :try_end_0
       .catch Ljava/lang/Throwable; {:try_start_0 .. :try_end_0} :catch_0

       return-void

       :catch_0
       return-void
   ```

### Step 4: Rebuild & Sign
```bash
apktool b -o snapenhance-patched-unsigned.apk apktool-out

keytool -genkey -v -keystore debug.keystore -storepass android -alias androiddebugkey -keypass android -keyalg RSA -keysize 2048 -validity 10000 -dname "CN=Android Debug,O=Android,C=US"

apksigner sign --ks debug.keystore --ks-key-alias androiddebugkey --ks-pass pass:android --key-pass pass:android --out snapenhance-patched-signed.apk snapenhance-patched-unsigned.apk
```

---

## ⚖️ Pros and Cons of This Approach

### **Pros**
* **100% Stability & No Toast Warnings:** Instantly launches without crash or annoying popup warning.
* **All Other Features Active:** Media download, stealth mode, unlimited view times, etc., work perfectly.
* **Argos Bypasses Active:** Core attestation bypasses using standard Java package names are still running.

### **Cons**
* **Peeking/Half Swipe Notifier Disabled:** Half-swipe notifications will not work on Snapchat `14.13.0.45` since the underlying talkcore classes are absent.
* **Manual Re-patching:** Required for new SnapEnhance updates until official support is updated.
