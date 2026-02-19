# 🛡️ Vulnerability Research: Command Execution in Adups OTA (`FotaProvider.apk`)

This research focuses on identifying a critical privilege escalation vulnerability within the `FotaProvider.apk` application. This system-level app, responsible for Over-The-Air (OTA) updates, was found to contain a backdoor/vulnerability that allows arbitrary command execution as a privileged user.

## 🔬 Lab Overview
* **Target Sample:** `FotaProvider.apk` (Adups OTA Provider)
* **SHA256:** `6fddd183bc832659cbea0e55d08ae72016fae25a4aa3eca8156f0a9a0db7f491`
* **Tool Used:** JADX-GUI (Java Decompiler)

---

## 🔍 Investigation & Finding the Vulnerability

### 1. Attack Surface Analysis (Manifest Audit)
The investigation began by auditing the `AndroidManifest.xml` to identify exported components that lack proper permission restrictions.

I identified a critical misconfiguration in the following component:
* **Receiver:** `com.adups.fota.sysoper.WriteCommandReceiver`
* **Exported Status:** `android:exported="true"`
* **Intent Action:** `android.intent.action.AdupsFota.operReceiver`

**Vulnerability:** Because the receiver is exported and does not require any specific signature-level permissions, any third-party application installed on the device can send a broadcast intent to it.

### 2. Bytecode Analysis (Command Injection)
I performed a deep-dive into the `onReceive` method of the `WriteCommandReceiver` class to understand how it processes incoming data.



The decompiled logic reveals a dangerous execution flow:
1.  **Input Extraction:** The receiver looks for a String extra in the intent named `"cmd"`.
2.  **Privileged Execution:** It passes this `"cmd"` string to a method (identified as `a` in the decompilation) which executes the command as the **System UID**.

**Observation:** There is no validation, filtering, or authentication of the command being passed.
<img width="1352" height="878" alt="Снимок экрана 2026-02-18 в 4 22 07 PM" src="https://github.com/user-attachments/assets/86fa60b0-f7fd-48fc-a171-7f63d6f52443" />
---

## 🚩 Impact Assessment

* **Vulnerability Type:** Arbitrary Command Execution / Backdoor.
* **Privilege Level:** `System UID` (one of the most privileged accounts on Android, just below root).
* **Risk:** A malicious app with zero permissions could exploit this "bridge" to execute privileged commands, exfiltrate private data, or install other malicious packages.

---
<img width="1352" height="878" alt="Снимок экрана 2026-02-18 в 4 39 49 PM" src="https://github.com/user-attachments/assets/94a48a8d-d057-4b2f-9021-f940ecc055de" />

## 📈 Security Engineering Takeaways
* **Supply Chain Risk:** Demonstrated how pre-installed firmware components can bypass the standard Android security model.
* **Component Hardening:** Identified the necessity of setting `android:exported="false"` or enforcing `signature` level permissions for sensitive system components.
* **XDR Detection:** This finding provides a blueprint for creating behavioral detection rules that monitor for unauthorized broadcasts to known vulnerable OTA receivers.

---
**Author:** Nikolai Vetrik  
**Project:** Mobile Vulnerability Research  
**Focus:** Android Internals | Reverse Engineering | Firmware Security
