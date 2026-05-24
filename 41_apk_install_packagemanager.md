# APK Install & PackageManager

## 1. Tổng quan

PackageManager (PMS) quản lý toàn bộ vòng đời của app: install, update, uninstall, permission grant, query.

```
APK file
  │
  ▼
Package Installer (UI)
  │ installPackage() via PackageInstaller API
  ▼
PackageManagerService (system_server)
  │
  ├── Verify APK signature (APK Signature Scheme v2/v3/v4)
  ├── Parse AndroidManifest.xml
  ├── dexopt (compile DEX → OAT)
  ├── Copy to /data/app/<package>/
  ├── Update /data/system/packages.xml
  └── Notify components (broadcast)
```

---

## 2. APK Signature Schemes

```
v1 (JAR signing): Manifest entries + .SF + .RSA
  → Vulnerable to Janus attack

v2 (APK Signing Block): Signature over entire ZIP
  → Android 7+, signs all bytes
  
v3 (Key rotation): Proof-of-rotation chain
  → Android 9+, allows key change with trust chain

v4 (Streaming): fs-verity based
  → Android 11+, incremental install support
  → Requires v2/v3 also present
```

```bash
# Kiểm tra APK signature
apksigner verify --verbose app.apk
# Verifies using V1, V2, V3 schemes

# Show signer info
apksigner verify --print-certs app.apk
# Signer #1 certificate:
#   Subject: CN=Android Debug, OU=Android
#   SHA-256 digest: ...

# Adb install
adb install app.apk
adb install -r app.apk  # replace existing
adb install --instant app.apk  # instant apps
```

---

## 3. Install Flow Chi Tiết

```
Session-based install (Android 5+):
  
  PackageInstaller.createSession(params)
    → /data/app/vmdl<sessionId>.tmp/ created
  
  session.openWrite("base.apk")
    → Stream APK bytes vào temp dir
  
  session.commit()
    → PMS validates APK
    → Move temp → /data/app/<package>-<hash>/
    → dexopt triggered
    → packages.xml updated
    → ACTION_PACKAGE_ADDED broadcast
```

---

## 4. DEX Optimization (dexopt)

```
Khi install APK:
  /data/app/com.myapp-xxx/base.apk (DEX classes.dex inside)
    │
    ▼
  dex2oat (AOT compile)
    │
    ▼
  /data/dalvik-cache/arm64/
    data@app@com.myapp-xxx@base.apk@classes.dex.art  ← .art (profile)
    data@app@com.myapp-xxx@base.apk@classes.dex.oat  ← .oat (compiled)
    data@app@com.myapp-xxx@base.apk@classes.dex.vdex ← .vdex (verified DEX)
```

```bash
# Xem dex optimization state
adb shell cmd package compile -m speed-profile com.myapp
# Modes: verify, quicken, speed-profile, speed, everything

# Force recompile
adb shell cmd package compile --reset com.myapp

# Check compilation state
adb shell cmd package dump com.myapp | grep -A5 "Dex Modes"

# Dexopt all apps
adb shell cmd package bg-dexopt-job
```

---

## 5. Package Metadata Storage

```
/data/system/packages.xml:
  Lưu tất cả installed apps:
    <package name="com.myapp"
             codePath="/data/app/com.myapp-xxx"
             version="123"
             userId="10456">
      <perms>
        <item name="android.permission.INTERNET" granted="true"/>
      </perms>
    </package>

/data/system/packages.list:
  com.myapp 10456 0 /data/user/0/com.myapp platform:privapp:targetSdkVersion=33

/data/app/com.myapp-xxx/:
  base.apk        ← Original APK
  oat/arm64/      ← Compiled code
  lib/arm64/      ← Native libs extracted
```

---

## 6. Permissions

```
Permission Types:
  Normal:     Auto-granted, no runtime prompt
  Dangerous:  Runtime prompt (READ_CONTACTS, CAMERA, etc.)
  Signature:  Only apps signed with same key
  Privileged: System apps in /system/priv-app/

Permission Groups:
  CAMERA group:   CAMERA
  CONTACTS group: READ_CONTACTS, WRITE_CONTACTS
  LOCATION group: ACCESS_FINE_LOCATION, ACCESS_COARSE_LOCATION
```

```java
// Runtime permission request (Android 6+)
if (checkSelfPermission(Manifest.permission.CAMERA)
        != PackageManager.PERMISSION_GRANTED) {
    requestPermissions(
        new String[]{Manifest.permission.CAMERA},
        REQUEST_CODE_CAMERA
    );
}

// Result callback
@Override
public void onRequestPermissionsResult(int requestCode,
        String[] permissions, int[] grantResults) {
    if (grantResults[0] == PackageManager.PERMISSION_GRANTED) {
        openCamera();
    }
}
```

```bash
# Grant permission via adb
adb shell pm grant com.myapp android.permission.CAMERA

# Revoke
adb shell pm revoke com.myapp android.permission.CAMERA

# List permissions
adb shell pm list permissions -d  # dangerous only
adb shell dumpsys package com.myapp | grep "permission"
```

---

## 7. App Data Directories

```
/data/user/0/com.myapp/
  ├── databases/   ← SQLite databases
  ├── shared_prefs/ ← SharedPreferences XML
  ├── files/       ← app-private files
  ├── cache/       ← Clearable cache
  └── lib → /data/app/com.myapp-xxx/lib/arm64/

/sdcard/Android/data/com.myapp/
  ├── files/       ← External files (user-visible)
  └── cache/       ← External cache
  
Multi-user:
  /data/user/0/  ← User 0 (primary)
  /data/user/10/ ← Work profile (user 10)
```

---

## 8. Intent Resolution

```java
// PMS resolves intents to components
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setData(Uri.parse("https://google.com"));

// PMS queries:
// 1. Explicit? → use component directly
// 2. Implicit? → match action/category/data against
//    all registered IntentFilters in packages.xml

ResolveInfo info = pm.resolveActivity(intent, 0);
// Returns best matching activity

// Query all handlers
List<ResolveInfo> list = pm.queryIntentActivities(intent, 0);
// Shows "Open with" chooser
```

---

## 9. System vs User Apps

```
/system/app/           ← Pre-installed, không xóa được
/system/priv-app/      ← Privileged apps (extra permissions)
/vendor/app/           ← Vendor-specific
/data/app/             ← User-installed (removable)
/data/app-private/     ← DRM-forward-locked (cũ)

Pre-installed app update:
  Play Store update → /data/app/com.google.android.gms-xxx/
  PMS ưu tiên /data/app/ version cao hơn
  Uninstall → revert về /system/app/ version
```

---

## 10. Debug PackageManager

```bash
# Dump full package info
adb shell dumpsys package com.myapp

# List all packages
adb shell pm list packages
adb shell pm list packages -3    # third-party only
adb shell pm list packages -s    # system only

# Clear app data
adb shell pm clear com.myapp

# Disable/enable app
adb shell pm disable com.myapp
adb shell pm enable com.myapp

# Force stop
adb shell am force-stop com.myapp

# Get APK path
adb shell pm path com.myapp
# package:/data/app/com.myapp-xxx/base.apk
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java` | PMS (~20k lines) |
| `frameworks/base/services/core/java/com/android/server/pm/InstallParams.java` | Install logic |
| `frameworks/base/core/java/android/content/pm/PackageManager.java` | Public API |
| `frameworks/base/services/core/java/com/android/server/pm/Settings.java` | packages.xml management |
| `tools/apksig/` | APK signature library |
| `libcore/dalvik/src/main/java/dalvik/system/DexFile.java` | DEX loading |
