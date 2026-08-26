# Release Notes - NewBlackbox

## Version: v4.0.1 - Comprehensive WebView & Network Fix (2026-08-26)

---

### Bug Fixes

#### WebView & Web-Based Apps `ERR_CACHE_MISS` Comprehensive Fix
**Problem:** Web-based apps and websites opened inside BlackBox failed to load with `net::ERR_CACHE_MISS`, while working properly when installed directly on Android.

**Root Causes & Solutions:**
1. **Android 11+ Permission Manager Hooking (`IPermissionManagerProxy`):**
   - On Android 11+ (API 30+), permissions are checked via `permissionmgr` (`IPermissionManager`), which was forwarding checks for virtual packages directly to the host OS and returning `PERMISSION_DENIED` (`-1`).
   - *Fix:* Added `checkPermission`, `checkUidPermission`, and `isPermissionRevokedByPolicy` hooks to automatically grant network and runtime permissions (`INTERNET`, `ACCESS_NETWORK_STATE`, etc.) to sandboxed apps.
2. **DNS Resolver Interception (`IDnsResolverProxy`):**
   - `IDnsResolverProxy` was hijacking Android's `dnsresolver` system service and returning `null` / mock lists, breaking standard DNS resolution for WebView Chromium sockets.
   - *Fix:* Removed `IDnsResolverProxy` hijacking to let sandboxed apps use standard Android DNS resolution.
3. **Connectivity Manager Clean Delegation (`IConnectivityManagerProxy`):**
   - `IConnectivityManagerProxy` was returning synthetic `Network(1)` objects and fake `LinkProperties`, causing `android_setsocknetwork` socket binding in Chromium to fail with invalid network IDs.
   - *Fix:* Cleaned up `IConnectivityManagerProxy` to cleanly delegate all network queries and callbacks to the real system `ConnectivityManager`, while correctly mapping virtual UIDs to the host process UID for UID queries.
4. **WebView Provider System Packages (`AppSystemEnv`):**
   - Modern Android uses `com.android.chrome`, `com.google.android.trichromelibrary*`, and vendor-specific browsers (Samsung, Xiaomi, Vivo, Heytap) as WebView providers. `AppSystemEnv.isOpenPackage` did not include these packages, causing `PackageManager.getPackageInfo` to return `null` when WebView loaded its native libraries.
   - *Fix:* Added all standard WebView providers, Trichrome libraries, and vendor WebView packages to `AppSystemEnv`.
5. **WebView Sandboxed Renderer Service (`BindIsolatedService`):**
   - `BindIsolatedService` in `IActivityManagerProxy` was wiping `args[6] = null`, corrupting the instance name required by Android ActivityManager to bind isolated WebView renderer processes.
   - *Fix:* Preserved the isolated service `instanceName` intact.
6. **WebView Data Directory Initialization Order (`BActivityThread`):**
   - `WebView.setDataDirectorySuffix` was previously called before `IOCore.get().enableRedirect()`, creating data directories in inconsistent paths.
   - *Fix:* Moved `setDataDirectorySuffix` to execute immediately after `IOCore.get().enableRedirect()`.

**Files Changed:**
- `Bcore/src/main/java/top/niunaijun/blackbox/fake/service/IPermissionManagerProxy.java`
- `Bcore/src/main/java/top/niunaijun/blackbox/fake/service/IConnectivityManagerProxy.java`
- `Bcore/src/main/java/top/niunaijun/blackbox/fake/hook/HookManager.java`
- `Bcore/src/main/java/top/niunaijun/blackbox/core/env/AppSystemEnv.java`
- `Bcore/src/main/java/top/niunaijun/blackbox/fake/service/IActivityManagerProxy.java`
- `Bcore/src/main/java/top/niunaijun/blackbox/app/BActivityThread.java`
- `app/src/main/AndroidManifest.xml`
- `Bcore/src/main/java/top/niunaijun/blackbox/fake/service/IPackageManagerProxy.java`

---

## Version: Build (2026-01-31)

---

### New Features

#### VPN Network Mode Toggle
Added a new setting to choose between VPN and normal network mode for sandboxed apps.

- **Location:** Settings → Others → Use VPN Network
- **Default:** OFF (normal network mode)
- When enabled, traffic is routed through BlackBox's VPN service
- Requires app restart to take effect

**Files Changed:**
- `app/src/main/java/top/niunaijun/blackboxa/view/main/BlackBoxLoader.kt`
- `app/src/main/java/top/niunaijun/blackboxa/view/setting/SettingFragment.kt`
- `app/src/main/res/xml/setting.xml`
- `app/src/main/res/values/strings.xml`
- `Bcore/src/main/java/top/niunaijun/blackbox/app/configuration/ClientConfiguration.java`
- `Bcore/src/main/java/top/niunaijun/blackbox/BlackBoxCore.java`

#### Device Information Logging
Added comprehensive device info header in logcat for easier debugging:
- Android version, SDK level, security patch
- Device manufacturer, brand, model, hardware
- Supported CPU/ABIs (32-bit and 64-bit)
- Memory info (heap usage)
- App version and package info
- Build fingerprint and timestamps

---

### Bug Fixes

#### VPN Permission Fix
**Problem:** VPN service failed to establish interface (`builder.establish()` returned null).

**Root Cause:** Android requires `VpnService.prepare()` to be called from an Activity before VPN can be established.

**Solution:** Added VPN permission request to `MainActivity.kt` on app launch.

**Files Changed:**
- `app/src/main/java/top/niunaijun/blackboxa/view/main/MainActivity.kt`

---

#### Android 10 Black Screen Fix
**Problem:** Apps would show a black screen and timeout on Android 10 (API 29).

**Root Cause:** 
- `BRAttributionSource.getRealClass()` returns `null` on Android < 31
- `SystemProviderStub.invoke()` crashed calling `.getName()` on null class
- `ClassInvocationStub.injectHook()` crashed when `getWho()` returned null

**Solution:**
- Added null checks in `SystemProviderStub.java` for API version checks
- Added null check in `ClassInvocationStub.java` to skip hooks when services don't exist

**Files Changed:**
- `Bcore/src/main/java/top/niunaijun/blackbox/fake/service/context/providers/SystemProviderStub.java`
- `Bcore/src/main/java/top/niunaijun/blackbox/fake/hook/ClassInvocationStub.java`

---

### Removed Features

#### Xposed Framework Support
- Removed `BXposedManagerService` and related AIDL interfaces
- Removed "Install Xposed Module" UI and Settings entries
- Cleaned up Xposed-related flags and package checks

---

### Stability Improvements

#### Anti-Detection Native Hook Stability
- Removed `LOGD` calls from critical native hooks to prevent infinite recursion
- Fixed syntax errors in hook implementations
- Hooks now silently return `ENOENT` for blocked paths

---

### Known Issues

#### Oppo/ColorOS Thermal Stats Error
On Oppo/ColorOS devices, you may see errors like:
```
OppoThermalStats: PackageManager$NameNotFoundException: top.niunaijun.blackboxa:p0
```
**This is harmless** - it's an Oppo system bug where their thermal management incorrectly uses process names (with `:p0` suffix) instead of package names. The app works normally.

---

### Compatibility

| Android Version | Status |
|-----------------|--------|
| Android 10 (Q)  | ✅ Fixed |
| Android 11 (R)  | ✅ Supported |
| Android 12 (S)  | ✅ Supported |
| Android 13 (T)  | ✅ Supported |
| Android 14 (U)  | ✅ Supported |
| Android 15+     | ✅ Supported |
