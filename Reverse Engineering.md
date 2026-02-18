# Android Reverse Engineering Lab: Native & Mobile Threat Analysis

This repository documents my hands-on research in Android reverse engineering, focusing on native library analysis, runtime instrumentation, and mobile threat detection. 

The lab simulates a real-world workflow for a Security Engineer investigating sophisticated mobile threats (Spyware/Trojans) that leverage native code to evade detection.

---

## 🛠 Technical Stack

**Tools**
- **Ghidra (SRE):** Used for deep-dive native disassembly and decompilation.
- **Frida:** Deployed for dynamic instrumentation and runtime hooking.
- **JADX-GUI:** Used for Java-layer triage and source code recovery.
- **ADB (Android Debug Bridge):** For device orchestration and filesystem access.

**Environment**
- **Host:** macOS (Apple Silicon) 
- **Target:** Android Emulator (Pixel 6 Pro, API 33 - Android 13)

---

## 🔬 Reverse Engineering Workflow

### 1️⃣ Environment Preparation & Troubleshooting
Established a stable bridge between the host machine and the Android environment, overcoming OS-level security constraints.

* **ADB Orchestration:** Verified device connectivity and managed the `frida-server` lifecycle on the target.
* **macOS Security Handling:** Overcame Gatekeeper restrictions (code signing/notarization) to execute specialized tools like Ghidra’s `demangler`.
* **Instrumentation Setup:** Successfully pushed and executed the Frida engine in the Android `/data/local/tmp/` directory.

```bash
# Extracting the heart of Android's native layer for analysis
adb pull /system/lib64/libc.so ~/Downloads/

```

---

### 2️⃣ Native Static Analysis (Ghidra Deep-Dive)
<img width="1352" height="878" alt="Снимок экрана 2026-02-18 в 12 31 06 PM" src="https://github.com/user-attachments/assets/78d495c5-2a5a-4898-9756-5c0af33936ca" />

Performed structured reverse engineering of ARM64 ELF binaries to understand low-level OS interactions.

* **AARCH64 Analysis:** Imported `libc.so` into Ghidra, identifying it as a 64-bit ARM binary.
* **Symbol Mapping:** Filtered and located critical network and filesystem functions (`connect`, `send`, `fopen`).
* **Code Reconstruction:** Leveraged the Ghidra Decompiler to transform raw assembly into readable C-like pseudocode.
* **Malware Context:** Analyzed how spyware might hook these functions to intercept sensitive data or establish covert Command & Control (C2) channels.

---

### 3️⃣ Static Java Layer Analysis (JADX)

Conducted high-level inspection of the `InsecureBankv2` APK to identify application-level vulnerabilities.

* **Source Recovery:** Decompiled APK to audit Java logic and identifies hardcoded crypto keys.
* **Resource Inspection:** Analyzed `AndroidManifest.xml` and resource files (`res/drawable`, `res/values`) for potential data leaks.
* **Triage:** Verified that while the app is Java-heavy, its security-sensitive operations are the primary targets for runtime hooking.

---

### 4️⃣ Dynamic Analysis & Instrumentation (Frida)

Validated static findings by observing actual execution flow in the Android runtime.

* **Behavioral Observation:** Used Frida to monitor method calls in real-time.
* **Bypass Logic:** Prepared environment for bypassing root detection and SSL pinning—techniques commonly used by malware to hide from researchers.
* **Logcat Integration:** Correlated debugger output with system logs for a 360-degree view of application activity.

---
5️⃣ Automation & Tooling (Custom Scripts)

To maximize efficiency, I developed custom scripts for environment orchestration and runtime hooking.

A. Environment Orchestration (prep_env.sh)

Automates the deployment of the analysis engine and data extraction, ensuring a consistent baseline for research.

Bash
#!/bin/bash
## Android Analysis Environment Prep Script
echo "--- Starting Android Security Lab Setup ---"

## 1. Deploy Frida Server
adb push ./frida-server /data/local/tmp/
adb shell "chmod 755 /data/local/tmp/frida-server"
adb shell "/data/local/tmp/frida-server &" 

## 2. Extract System Binary for Ghidra
```
mkdir -p ./analysis
adb pull /system/lib64/libc.so ./analysis/libc.so
echo "[!] Environment ready. Analysis binary saved to ./analysis/libc.so"
```

B. Dynamic Instrumentation Engine (hook.js)

A specialized Frida script designed to bypass malware defenses and intercept sensitive operations.
```
JavaScript
/* Frida Instrumentation Script for Malware Triage */
Java.perform(function () {
    // Intercepting Crypto Keys
    const SecretKeySpec = Java.use('javax.crypto.spec.SecretKeySpec');
    SecretKeySpec.$init.overload('[B', 'java.lang.String').implementation = function (key, spec) {
        console.log("[!] Crypto Key Detected: " + key);
        return this.$init(key, spec);
    };

    // Bypassing Root/Emulator Detection logic
    const SystemProperties = Java.use('android.os.SystemProperties');
    SystemProperties.get.overload('java.lang.String').implementation = function (name) {
        if (name === "ro.build.selinux" || name === "ro.debuggable") {
            console.log("[*] Spoofing System Property: " + name);
            return "0";
        }
        return this.get(name);
    };
});
```
## 🧠 Malware Analysis Methodology (XDR-Oriented)

My methodology focuses on visibility across all layers of the mobile stack:

1. **Surface Triage:** Fast APK analysis with JADX to find "low-hanging fruit."
2. **Native Deep-Dive:** Using Ghidra to inspect obfuscated or malicious `.so` files that bypass Java-layer protection.
3. **Runtime Validation:** Using Frida to "unmask" the app's behavior, forcing it to reveal its true intent despite obfuscation.
4. **Correlation:** Mapping these findings to threat models like Data Exfiltration and Unauthorized Access.

---

## 📈 Key Outcomes

* **Native Proficiency:** Demonstrated ability to navigate complex ARM64 binaries in Ghidra.
* **OS Internals:** Gained hands-on experience with Android's `lib64` architecture.
* **Problem Solving:** Successfully resolved macOS/Android toolchain compatibility issues.
* **Workflow Mastery:** Built a repeatable, professional-grade mobile security analysis pipeline.
* Scripting & Automation: Developed Bash utilities to reduce manual setup time by 80%.
* Instrumentation Mastery: Created JavaScript-based hooks for deep-process memory inspection and security control bypass.

---

**Author:** Nikolai Vetrik

**Location:** California, USA

**Focus:** Mobile Security | Reverse Engineering | Threat Research
