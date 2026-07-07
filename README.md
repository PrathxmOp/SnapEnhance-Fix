# SnapEnhance Fix for Snapchat 14.13.0.45

This repository documents the analysis, diagnosis, and fix for the instant crash occurring on rooted Android devices (KernelSU/LSPosed).

### 📋 Version Info
* **Snapchat:** `14.13.0.45` (arm64-v8a)
* **SnapEnhance:** `2.2.0` (v2.2.0 stable / latest release)

---

## 🚨 The Problem

On newer Snapchat versions (like `14.13.0.45`), launching the app with SnapEnhance enabled causes Snapchat to crash instantly with a native segmentation fault (**`SIGSEGV` / Exit Code 139**). 

### Root Cause
SnapEnhance uses a class mapping system (`mappings.json`) to resolve obfuscated Snapchat class names and signatures. One of these mappers, **`PlatformClientAttestationMapper`**, targets native client attestation validation classes (specifically looking for the class mapping `pluginNativeClass` to `jc.tyqs` and `apiInvocationHandler` to `vL1`).

Because Snapchat `14.13.0.45` restructured its native security modules, the mapper resolved incorrect/shifted structures. When SnapEnhance attempted to hook these invalid locations, it triggered memory corruption.

---

## 🔍 How We Isolated the Culprit

We wrote a diagnostic python script that systematically excluded different mapping keys from `mappings.json`, launched Snapchat, and checked if the process stayed alive:
1. When we excluded the root key `Callbacks`, Snapchat ran fine, but all Xposed hooks were disabled.
2. We then performed a definitive exclusion sweep and discovered that excluding **`PlatformClientAttestationMapper`** alone made Snapchat **100% stable** with all other hooks and callbacks loaded!

---

## 🛠️ The Permanent Fix

Instead of manually editing and locking `mappings.json` on the device, we patched the SnapEnhance APK itself to prevent it from registering the failing mapper.

### Step-by-Step Patching Guide

If you want to replicate this fix:

1. **Decompile SnapEnhance APK:**
   ```bash
   apktool d -o apktool-out snapenhance.apk
   ```

2. **Locate the Class Mapper:**
   Open `apktool-out/smali/me/rhunk/snapenhance/mapper/ClassMapper$Companion.smali`.

3. **Modify `getDEFAULT_MAPPERS`:**
   - Change the array size from `19` (`0x13`) to `18` (`0x12`):
     ```smali
     const/16 v15, 0x12
     new-array v15, v15, [Lme/rhunk/snapenhance/mapper/AbstractClassMapper;
     ```
   - Delete the instantiation lines for `PlatformClientAttestationMapper`:
     ```smali
     # REMOVE THIS BLOCK
     new-instance v18, Lme/rhunk/snapenhance/mapper/impl/PlatformClientAttestationMapper;
     invoke-direct/range {v18 .. v18}, Lme/rhunk/snapenhance/mapper/impl/PlatformClientAttestationMapper;-><init>()V
     ```
   - Remove it from the array assignment (index `18` / `0x12`):
     ```smali
     # REMOVE THIS BLOCK
     const/16 v0, 0x12
     aput-object v18, v15, v0
     ```

4. **Rebuild the APK:**
   ```bash
   apktool b -o snapenhance-patched-unsigned.apk apktool-out
   ```

5. **Sign the APK:**
   Create a debug keystore if you don't have one and sign the APK:
   ```bash
   keytool -genkey -v -keystore debug.keystore -storepass android -alias androiddebugkey -keypass android -keyalg RSA -keysize 2048 -validity 10000 -dname "CN=Android Debug,O=Android,C=US"
   
   apksigner sign --ks debug.keystore --ks-key-alias androiddebugkey --ks-pass pass:android --key-pass pass:android --out snapenhance-patched-signed.apk snapenhance-patched-unsigned.apk
   ```

6. **Install and Enable:**
   Uninstall the old SnapEnhance, install `snapenhance-patched-signed.apk`, enable the module in LSPosed manager, check the Snapchat scope, and launch Snapchat!

---

## ⚖️ Pros and Cons of This Approach

### **Pros**
* **100% Stability:** Bypasses the crash entirely; Snapchat launches and functions flawlessly.
* **No File System Workarounds:** No need to chown/chmod `mappings.json` to read-only or lock files with root access.
* **Fully Functional Features:** All customization features of SnapEnhance (media downloading, chat tweaks, UI features) remain fully operational since their mappings are successfully registered.
* **Backup Bypasses Intact:** The main Argos client attestation hooks (loaded via hardcoded unobfuscated class strings like `com.snapchat.client.client_attestation.ArgosClient`) are still active in SnapEnhance, meaning you still retain basic integrity protection.

### **Cons**
* **Bypassed Native Verification Hooks:** The secondary obfuscated native attestation hook is skipped (though this is already broken on newer Snapchat builds anyway).
* **Manual Updates:** If you update SnapEnhance, you will need to re-apply this smali patch to the newer APK until the official developers update the mapper rules.
