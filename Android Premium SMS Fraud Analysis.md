# Case Study: Android Premium SMS Fraud Analysis

This lab focuses on the static analysis of a malicious Android application (`ThaiCamera.apk`) to identify indicators of **Premium SMS Fraud**. The goal is to determine if the application sends SMS messages to premium shortcodes without explicit user consent or proper disclosure.

## 🛠 Analysis Environment
<img width="1352" height="878" alt="Снимок экрана 2026-02-18 в 3 44 33 PM" src="https://github.com/user-attachments/assets/fd585ea7-63d7-4cd4-b698-410624081103" />

* **Target Sample:** `ThaiCamera.apk`
* **SHA256:** `55da412157e93153e419c3385ebcd5335bd0d0c3f77a75e2d2413dd128270be2`
* **Tools Used:** JADX-GUI (Java Decompiler)

---

## 🔍 Investigation Workflow

### 1. Identifying the Entry Point
Using **JADX-GUI**, I analyzed the `Loading` class, which serves as a primary trigger for background operations. 

The analysis revealed an `onClick` event listener (line 111) that initiates a sensitive permission request sequence.

### 2. Evidence of Runtime Permission Escalation
I identified a direct call to the Android permission system specifically targeting SMS functionality:

```java
// Line 119 in Loading.java
ActivityCompat.requestPermissions(Loading.this, new String[]{"android.permission.SEND_SMS"}, 1);

```

**Observation:** The application attempts to gain `SEND_SMS` privileges at runtime, which is a significant indicator of potential fraud when the app's stated purpose (Camera/Filter) does not justify SMS access.

### 3. Tracing the Malicious Logic

The code logic shows that once permissions are handled, the application proceeds to execute a `sendMessage` method:

* **Method:** `this.sendMessage(Loading.this.service, Loading.this.content);`
* **Mechanism:** The method dynamically pulls the `service` (target phone number) and `content` (SMS message) from internal variables.
* **Deception:** Instead of clear disclosure, the app displays a vague Toast message: `"Please allow access"`. This fails to inform the user about the financial cost or the destination of the SMS.

---

## 🧠 Forensic Evaluation (Trellix XDR Context)

To classify this sample as **Malware (Premium SMS Fraud)**, I evaluated it against four industry-standard criteria:

| Criterion | Finding | Evidence |
| --- | --- | --- |
| **Sending SMS?** | **YES** | Presence of `SEND_SMS` permission request and `sendMessage` call. |
| **Premium Number?** | **Suspected** | Identified `loginByPost` method likely fetching shortcodes from a remote C2 server. |
| **Obvious Disclosure?** | **NO** | Toast message only says "Please allow access," hiding the financial impact. |
| **User Consent?** | **NO** | The SMS is sent based on a misleading button click without cost confirmation. |

---

## 📈 Key Achievements

* **Static DEX Analysis:** Successfully navigated decompiled Java code to locate high-risk API calls.
* **Threat Modeling:** Applied malware analyst methodology to differentiate between legitimate app behavior and fraudulent activity.
* **API Hook Identification:** Identified the specific Android SDK methods (`requestPermissions`) used for privilege escalation.

---

**Author:** Nikolai Vetrik

**Project:** Android Malware Research Lab

**Focus:** Mobile Security | Reverse Engineering | XDR Detection Logic
