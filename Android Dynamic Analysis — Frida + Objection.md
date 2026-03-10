# Android Dynamic Analysis — Frida + Objection Practice

Hands-on dynamic instrumentation lab using Frida and Objection against intentionally vulnerable Android applications. Focused on runtime hooking, root detection bypass, and SSL pinning analysis.

---

## Environment Setup

| Component | Details |
|---|---|
| Attack machine | Kali Linux (VM) |
| Target | Android Emulator (AVD) — Pixel 6 Pro, API 33, arm64-v8a |
| Frida version | 17.8.0 |
| Objection version | 1.12.3 |
| Target app | [AndroGoat](https://github.com/satishpatnayak/AndroGoat) — intentionally vulnerable Android app |

### Network Bridge
The emulator runs on macOS host while Kali runs in a VM — direct localhost connection is not possible. Solution: SSH reverse tunnel to forward the ADB port.

```bash
# On macOS host — enable TCP mode on emulator
adb tcpip 5555

# SSH reverse tunnel: forward port 5555 from Mac to Kali
ssh -R 5555:localhost:5555 user@<kali-ip>

# On Kali — connect via tunnel
adb connect 127.0.0.1:5555
adb devices
# 127.0.0.1:5555  device
```

---

## frida-server Deployment

```bash
# Check emulator architecture
adb shell getprop ro.product.cpu.abi
# arm64-v8a

# Download matching frida-server
wget https://github.com/frida/frida/releases/download/17.8.0/frida-server-17.8.0-android-arm64.xz
xz -d frida-server-17.8.0-android-arm64.xz

# Deploy to emulator
adb push frida-server-17.8.0-android-arm64 /data/local/tmp/frida-server
adb shell "chmod 755 /data/local/tmp/frida-server"

# Restart adb as root and launch server
adb root
adb shell "/data/local/tmp/frida-server &"

# Verify — list all running processes
frida-ps -U
```

---

## Target: AndroGoat

AndroGoat is an open-source intentionally vulnerable Android application covering OWASP Mobile Top 10. It includes dedicated exercises for:

- Root detection bypass (RootCloack, Repackaging, **Frida**)
- Insecure storage (SharedPreferences, SQLite, SD card)
- SSL pinning bypass
- Hardcoded secrets
- Emulator detection

### Discovered Attack Surface via Objection

```
android hooking list activities
```
<img width="1352" height="878" alt="Снимок экрана 2026-03-10 в 10 03 17 AM" src="https://github.com/user-attachments/assets/cc190b6f-1618-4c27-8da4-c8c9374d32b1" />
```
owasp.sat.agoat.RootDetectionActivity
owasp.sat.agoat.InsecureStorageActivity
owasp.sat.agoat.InsecureStorageSQLiteActivity
owasp.sat.agoat.InsecureStorageSharedPrefs
owasp.sat.agoat.HardCodeActivity
owasp.sat.agoat.EmulatorDetectionActivity
owasp.sat.agoat.BioMetricAuthActivity
owasp.sat.agoat.SQLinjectionActivity
# + 22 more
```

---

## Root Detection Bypass via Frida

### Objective
Hook `RootDetectionActivity` methods at runtime and force them to return `false` — making the app believe the device is not rooted even when running with root privileges.

### Hook Script

```javascript
Java.perform(function() {
    var RootDetection = Java.use("owasp.sat.agoat.RootDetectionActivity");

    RootDetection.checkRoot1.implementation = function() {
        console.log("[*] checkRoot1() hooked — returning false");
        return false;
    };
    RootDetection.checkRoot2.implementation = function() {
        console.log("[*] checkRoot2() hooked — returning false");
        return false;
    };
    RootDetection.checkRoot3.implementation = function() {
        console.log("[*] checkRoot3() hooked — returning false");
        return false;
    };

    console.log("[*] All root checks hooked");
});
```

### Execution

```bash
frida -U -n "AndroGoat - Insecure App (Kotlin)" -l ~/hook_root.js
```

### Output
```
Attaching...
[*] RootDetectionActivity found, hooking methods...
[*] checkRoot1() hooked — returning false
[*] checkRoot2() hooked — returning false
[*] checkRoot3() hooked — returning false
```

Result: App displays **"Device is not rooted"** despite running on a rooted emulator with frida-server active.
<img width="1352" height="878" alt="Снимок экрана 2026-03-09 в 11 20 25 PM" src="https://github.com/user-attachments/assets/0ea6212b-2fa5-40ae-840d-2d0fce7b8022" />

---

## SSL Pinning Analysis

### Objection auto-detection
```
android sslpinning disable
```
```
(agent) Custom TrustManager ready, overriding SSLContext.init()
(agent) Found okhttp3.CertificatePinner, overriding CertificatePinner.check()
(agent) Found okhttp3.CertificatePinner, overriding CertificatePinner.check$okhttp()
```

**Finding:** AndroGoat uses OkHttp3 `CertificatePinner` for SSL pinning. Objection successfully identified the pinning implementation. The `check$okhttp()` Kotlin-specific overload required a custom Frida script to fully bypass due to signature mismatch in objection 1.12.3.

**Takeaway:** Automated tools handle common cases. Custom Frida scripts are required for Kotlin-specific method signatures.

---

## Key Findings

| Finding | Severity | Detail |
|---|---|---|
| Root detection bypassable via Frida hooks | High | All 3 checkRoot methods hookable at runtime |
| OkHttp3 SSL pinning identified | High | CertificatePinner detected, partially bypassed |
| 30 exported Activities discovered | Medium | Multiple sensitive Activities accessible |
| Insecure storage implementations | High | SharedPreferences, SQLite, temp files |

---

## Tools Used

| Tool | Purpose |
|---|---|
| `frida` 17.8.0 | Dynamic instrumentation framework |
| `objection` 1.12.3 | Frida-based mobile security testing toolkit |
| `adb` | Android Debug Bridge — device control |
| `jadx` | Static APK decompilation |
| Android Studio AVD | Emulator — Pixel 6 Pro API 33 |

---

## References

- [AndroGoat — OWASP vulnerable app](https://github.com/satishpatnayak/AndroGoat)
- [Frida documentation](https://frida.re/docs/home/)
- [Objection](https://github.com/sensepost/objection)
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)
- Yang et al., *"Understanding Miniapp Malware: Identification, Dissection, and Characterization"*, NDSS 2025
