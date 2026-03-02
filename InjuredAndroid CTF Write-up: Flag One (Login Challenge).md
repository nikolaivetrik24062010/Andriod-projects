# InjuredAndroid CTF Write-up: Flag One (Login Challenge)

This repository contains my step-by-step solution for the first flag of the **InjuredAndroid** vulnerable application. The challenge was solved using dynamic instrumentation to bypass standard login checks and intercept sensitive data in real-time.

## 📝 Challenge Overview
* **Target:** Flag One - Login
* **Difficulty:** Beginner
* **Objective:** Identify the hardcoded flag/password by intercepting string comparisons in the application's memory.

## 🛠️ Environment & Tools
* **Host OS:** Kali Linux (Apple Silicon/arm64)
* **Device:** Android Emulator (Pixel 6 Pro - API 33)
* **Framework:** Frida 17.7.3
* **Utilities:** `adb` (Android Debug Bridge), `nano`, `python3-pip`

---

## 🚀 Execution Steps

### 1. Installation & Environment Setup
To begin, I installed the Frida toolkit using the Python package manager. This is the standard way to ensure all dependencies are correctly handled:

```bash
# Installing Frida via pip
pip3 install frida-tools frida

```

Before running the exploit, I performed a "clean sweep" to ensure no ghost processes were interfering with the instrumentation:

```bash
# Killing any existing Frida processes on the host
pkill -9 frida
pkill -9 frida-server

# Verifying connection to the emulator
adb devices

# Starting the Frida server on the target device
adb shell "/data/local/tmp/frida-server &"

```

### 2. Identifying the Problem: "System Noise"

Initially, a broad hook on `java.lang.String.equals` resulted in a massive flood of system logs. Android constantly compares strings for internal processes like keyboard state (`input_method`) and UI rendering (`layout_inflater`).

To find the flag, I had to develop a **Precision Hook** that targets only the login logic.

### 3. Developing the Exploit (`solve_flag.js`)

I wrote a custom JavaScript injection that monitors the `submitFlag` method of the `FlagOneLoginActivity` class. This script includes a blacklist filter to ignore system "junk" and highlight only strings that match the flag pattern.

```javascript
/*
 * Dynamic Instrumentation script for InjuredAndroid
 * Goal: Capture Flag One by intercepting string comparisons
 * Author: Nikolai
 */

Java.perform(function () {
    console.log("[*] Precision Mode Active");
    console.log("[*] Filtering out system noise... Waiting for Submit click.");

    // Targeting the specific Activity class for Flag One
    var FlagOneActivity = Java.use("b3nac.injuredandroid.FlagOneLoginActivity");

    // Intercepting the submit button implementation
    FlagOneActivity.submitFlag.implementation = function (view) {
        console.log("[!] Submit button pressed! Intercepting data...");

        var StringClass = Java.use("java.lang.String");

        StringClass.equals.implementation = function (arg) {
            var result = this.equals(arg);
            var target = (arg !== null) ? arg.toString() : "";

            // Blacklist: Filtering out common Android system strings
            var isJunk = target.includes("input_method") || 
                         target.includes("timeout") || 
                         target.includes("window") || 
                         target === "UTF-8";

            // Pattern Matching: Identifying the flag structure
            if ((target.includes("F1ag") || target.includes("_")) && !isJunk) {
                console.log("\n*******************************************");
                console.log("[+] SUCCESS! FLAG CAPTURED: " + target);
                console.log("*******************************************\n");
            }

            return result;
        };

        // Execute original logic to maintain application stability
        this.submitFlag(view);
    };
});

```

---

## 🎯 Results

After launching the script with the following command:

```bash
frida -U -l solve_flag.js -f b3nac.injuredandroid

```

I entered a dummy value in the login field and hit **SUBMIT**. The filter successfully isolated the hardcoded flag from the system background noise.

**Captured Flag:** `F1ag_0n3`
<img width="1352" height="878" alt="Снимок экрана 2026-03-02 в 11 36 10 AM" src="https://github.com/user-attachments/assets/5c922d2b-a7f9-48ec-aa6c-ab5a13e0851e" />

---

## 💡 Lessons Learned

* **Installation:** Using `pip` for Frida ensures a reliable environment setup on Kali Linux.
* **Dynamic Instrumentation:** Using Frida is often faster than static analysis because it allows us to see data *after* it has been decrypted or processed in memory.
* **Output Management:** In Android pentesting, writing robust filters for `String.equals` is vital to avoid being overwhelmed by OS-level telemetry.
