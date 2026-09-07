---
name: android-security-wizard
description: "Use when doing Android security: ADB/Shizuku access, malware hunting, reverse engineering, kernel exploitation, and instrumentation."
version: 1.0.0
author: savagedamage
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [Android, Security, ADB, Shizuku, Reverse Engineering, Malware, Forensics, Frida, Exploitation, Instrumentation]
    category: security
---

# Android Security Wizard — Skill

**Purpose:** Become a deep Android security operator: ADB/Shizuku privileged access, stealthy malware hunting, exploit isolation and identification, reverse engineering, kernel exploitation, and advanced instrumentation. This skill encodes the tooling, workflows, and methodology to operate at that level on Android devices you own or have explicit authorization to test.

**Scope:** Android app security, on-device forensics, malware triage, IPC/exported-component attack surface, kernel-level exploit identification, ADB/Shizuku/Dhizuku privilege models, Frida/Objection dynamic instrumentation, and integrated static+dynamic analysis pipelines.

**Out of scope (for now):** Hardware forensics (JTAG/ISP/chip-off), baseband/firmware extraction, iOS, non-Android Linux kernel exploitation without Android context.

---

## 1. The Four Pillars

An Android security operator works across four layers. Know which layer you're in before you run a command.

### Pillar 1 — Privileged Access (how you get deep)

**ADB (Android Debug Bridge):** USB or Wi-Fi debugging gives you a shell as UID 2000 (`shell`). Enough for most `pm`, `dumpsys`, `logcat`, `appops`, and package inspection. Not enough for another app's `/data/data/` or kernel access. Get it from: [SDK Platform Tools](https://developer.android.com/studio/releases/platform-tools).

**Shizuku (UID 2000 shell, via app_process):** A Java server started with `app_process` that exposes a Binder service (`moe.shizuku.privileged.api`). Client apps wrap system-service Binder interfaces with `ShizukuBinderWrapper` and get shell-grade API access without being system apps. [RikkaApps/Shizuku](https://github.com/RikkaApps/Shizuku) (29.7k stars) is the core; [RikkaApps/Shizuku-API](https://github.com/RikkaApps/Shizuku-API) is the developer library; [thejaustin/ShizukuPlus](https://github.com/thejaustin/ShizukuPlus) is an enhanced fork. [awesome-shizuku](https://github.com/timschneeb/awesome-shizuku) (10k stars) catalogs 980+ apps using it.

**Shizuku UserService:** Run your own Java/JNI code in a separate process as UID 2000 (shell) or UID 0 (root). This is how you build long-lived tooling or complex operations that don't fit a single Binder call. Check `Shizuku.getUid()` to know which tier you're in (2000 vs 0).

**Dhizuku:** Alternative that uses Device Owner privileges (different axis — install/uninstall/managed profiles). Mutually exclusive with other DO apps. Useful when you have DO provisioning, not ADB.

**Root (UID 0):** Full filesystem, kernel, SELinux control. Magisk + Sui module for root-mode Shizuku.

**Privilege tier decision rule:** Start with ADB. If you need another app's private data or kernel access, escalate to root. If you need to build tooling that runs as a service, use Shizuku UserService. If you have Device Owner, evaluate Dhizuku.

### Pillar 2 — Static Analysis (what the app says it does)

**MobSF** ([MobSF/Mobile-Security-Framework-MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF), 21.7k stars): All-in-one static + dynamic. Auto-extracts manifest, permissions, API calls, hardcoded secrets, certificate pinning; runs YARA; REST API + CLI. v4.5.2 (Aug 2026). The single most complete open-source Android analysis platform.

**Androguard** ([androguard/androguard](https://github.com/androguard/androguard)): Python toolkit for APK/DEX/odex. Parses manifests, extracts permissions/services/receivers/providers/intent filters, decompiles DEX to Python AST, has a built-in YARA module. The programmatic backbone inside MobSF.

**JADX** ([skylot/jadx](https://github.com/skylot/jadx), 50.3k stars, active yesterday): Dex→Java decompiler, CLI + GUI. Handles APK, dex, aar, aab, zip. Decodes manifest. Smali view. Plugins marketplace. Use when you need readable Java from DEX.

**apktool** ([iBotPeaches/APKTool](https://github.com/iBotPeaches/APKTool)): Decode resources to near-original form, rebuild after Smali edits. Essential for manifest inspection and repackaging.

**YARA** ([VirusTotal/yara](https://github.com/VirusTotal/yara)): Pattern-matching swiss knife. Android-relevant rules from: [InQuest/awesome-yara](https://github.com/InQuest/awesome-yara) (index), [bartblaze/Yara-rules](https://github.com/bartblaze/Yara-rules) (broad), [ReversingLabs](https://github.com/ReversingLabs) (ATT&CK-mapped), [MalwareBazaar](https://bazaar.abuse.ch/) (see what's matching in the wild).

**Static triage rule:** Run MobSF first (automatic everything). Use Androguard for programmatic inspection. Use JADX when you need to read Java. Use YARA when you have a specific family/hash/IOC to check against.

### Pillar 3 — Dynamic Instrumentation (what the app actually does)

**Frida** ([frida/frida](https://github.com/frida/frida), 21.8k stars, active 10h ago): Dynamic instrumentation. `frida-server` on-device (push to `/data/local/tmp/`, chmod 755, run in background). `frida-ps -U` to list processes. `frida-trace -U -i open -N com.android.chrome` for tracing. `frida --codeshare` for community scripts. Gadget mode for non-root (repackage APK with `libfrida-gadget.so`). [Frida docs](https://frida.re/docs/android/).

**Objection** ([sensepost/objection](https://github.com/sensepost/objection), 9.4k stars, commit Jul 2026): `pip3 install objection`. Runtime mobile exploration powered by Frida. SSL pinning bypass, root detection bypass, keychain dump, memory dump/patch, heap exploration. `objection explore com.target.app`.

**Frida CodeShare scripts:** `frida --codeshare ssecurityy/universal-robust-advanced-root--ssl-pinning-bypass -f YOUR_BINARY` — universal root + SSL pinning bypass covering: TLS trust manager unpinning (TrustManagerImpl, WebViewClient, OkHttp, Apache, iOS NSURL), root hiding framework hooks (Magisk, RootCloak, Xposed, Substrate), debugger detection tamper (`isDebuggerConnected`/`waitingForDebugger`), obfuscated method hooks (auto-detect boolean/int/String return, <=2 args), JNI/B|iner/ClassLoader/anti-debug/emulator/network/WebView/memory test hooks.

**httptoolkit/frida-interception-and-unpinning** ([httptoolkit/frida-interception-and-unpinning](https://github.com/httptoolkit/frida-interception-and-unpinning), 2.3k stars, commit 20h ago): Frida scripts to MITM all HTTPS traffic at runtime. Android scripts: `android-disable-certificate-pinning.js`, `android-disable-flutter-certificate-pinning.js` (experimental). iOS scripts: `ios-connect-hook.js` (capture all traffic), `ios-disable-detection.js` (JailMonkey bypass). Uses `native-connect-hook.js` and `native-tls-hook.js` for native TLS interception. EU NGI Zero Entrust funded.

**ReverseLabs DCL detection:** [Dynamic Code Loading on Android](https://reverselabs.dev/blog/dynamic-code-loading-android) — hook `dalvik.system.DexClassLoader.$init` to log every path loaded at runtime. Pattern: hidden DEX in assets, two-stage dropper (BRATA, Ghimob). Static indicators: `DexClassLoader`, `PathClassLoader`, `loadClass`, `.dex/.apk/.jar` in assets/OBB.

**Dynamic rule:** Run Frida/Objection after static analysis tells you what to hook. Use CodeShare scripts for common bypasses. Build custom hooks for app-specific behavior. Use DCL hooks when static analysis shows DexClassLoader usage.

### Pillar 4 — Exploit Identification & Isolation (what broke the device)

**drozer** ([ReversecLabs/drozer](https://github.com/ReversecLabs/drozer), 4.6k stars): IPC security assessment. Install `drozer-agent.apk` on device, connect console via port 31415 (direct) or infrastructure mode. `run app.package.attacksurface <pkg>` to enumerate exported components. `run app.package.info -a <pkg>` for metadata. Custom Python modules. Does NOT exploit kernel bugs — it maps the app-layer IPC attack surface. [Documentation](https://labs.reversec.com/tools/drozer/).

**Ghost Framework** ([EntySec/Ghost](https://github.com/EntySec/Ghost), 3.4k stars): ADB post-exploitation RAT. `pip3 install git+https://github.com/EntySec/Ghost`. Needs ADB access first. Command execution, file ops, APK extraction, shell upgrade. Represents the post-exploit phase — useful for detecting post-compromise activity.

**Isolate-and-identify workflow (5 phases):**

Phase 0 — Detect & triage: `getenforce`, `adb shell getprop ro.debuggable`, `ps -A | grep -iE 'root|system|adbd|shell'`, `netstat -tulpn`, `logcat -d | tail -1000`.

Phase 1 — Isolate: Disable Wi-Fi/data/BT. If ADB over TCP: `adb shell setprop service.adb.tcp.port -1`. Do NOT power off. Keep ADB connection open. Take screenshots with `adb shell screencap`.

Phase 2 — Collect artifacts: `logcat -d -v time -b all > logcat_full.txt`, `dumpsys > dumpsys_full.txt`, `pm list packages -f > pm_packages_f.txt`, `pm path <pkg>`, pull APK + hash (sha256sum/md5sum), `getprop ro.build.fingerprint`, `getprop ro.build.version.security_patch`, `uname -a`, pull `/data/tombstones/`, `ps -A -o pid,uid,comm,args`, `netstat -tulpn`, `getenforce` + `dmesg | grep -i avc`.

Phase 3 — CVE/kernel match: Fingerprint → Android Security Bulletin (source.android.com/docs/security/bulletin/<year>-<month>-01) → patch level → kernel version → xairy/linux-kernel-exploitation index → component hints from artifacts (binder crash → binder CVEs, mali/kgsl/adreno → GPU CVEs, alsa/snd → ALSA CVEs).

Phase 4 — PoC correlation: sfewer-r7/pocindex (82k PoCs), GitHub by CVE ID, Exploit-DB. Evaluate: same kernel version? same access level? matches observed behavior? upstream or Android-specific? Reproduce in lab only (0xbinder/android-kernel-exploitation-lab QEMU setup, never on the compromised device).

Phase 5 — Attribution: CVE → exploit chain structure → Google TAG/Project Zero writeups → APK/sample comparison → network indicators → acknowledge limitations (clean exploits leave minimal traces, SELinux state matters, attribution to specific actor usually needs more than device artifacts).

**Kernel exploit lab (reproducible):** [android-kernel-exploit-lab-setup.md](../../../android-kernel-exploit-lab-setup.md) on disk — full QEMU + vulnerable kernel + GDB walkthrough for CVE-2019-2215. 0xbinder lab (43 stars, active Apr 2025) + cloudfuzz.github.io/workshop. Same core flow: shallow repo sync of `q-goldfish-android-goldfish-4.14-dev` → apply patch → fix DTC lexer → build x86_64 KASAN kernel → create AVD (android-29, x86_64) → launch emulator with `-kernel bzImage` + optionally `-qemu -s -S` → GDB attach on :1234 → `root-by-pid <pid>`.

---

## 2. Integrated Pipeline: APK In → Finding Out

### Step 1 — Intake & hash

```
sha256sum app.apk > hashes.txt
md5sum app.apk >> hashes.txt
```

### Step 2 — Quick static (MobSF, 5 minutes)

Start MobSF, upload APK, let it run. Look at: manifest permissions (dangerous combos), exported components, API calls (SMS, phone, location, camera, microphone), hardcoded secrets/keys, certificate pinning detected, YARA hits.

### Step 3 — Manifest + component inspection (apktool + manual)

```
apktool d app.apk -o app_decoded
grep -r "android:exported=\"true\"" app_decoded/AndroidManifest.xml
grep -r "DexClassLoader\|PathClassLoader\|loadClass" app_decoded/smali/
```

For each exported component, check permission guarding: `grep -A5 "exported"` in manifest; `aapt dump badging app.apk` for permissions/activities.

### Step 4 — Decompile to Java (JADX, when you need to read logic)

```
jadx-gui app.apk
# or headless:
jadx -d out_java app.apk
```

Focus on: services with `android:process=":remote"`, receivers for `SMS_RECEIVED`/`BOOT_COMPLETED`/`PACKAGE_INSTALLED`, content providers with unusual URIs, activities with `android:exported="true"` and no icon.

### Step 5 — Dynamic instrumentation (Frida/Objection, target what static found)

Start frida-server on device. Attach to app. Hook what Step 3-4 flagged:

- **SMS fraud:** `SmsManager.sendTextMessage`, `sendMultipartTextMessage`
- **Camera/mic abuse:** `CameraManager`, `MediaRecorder`, `AudioRecord`, invisible `SurfaceView` (1x1)
- **Network exfil:** `HttpURLConnection`, `OkHttp`, `Socket`, `ContentResolver` queries on contacts/SMS/call_log
- **DCL:** `dalvik.system.DexClassLoader.$init` — log every path
- **Accessibility abuse:** `AccessibilityService` usage, `sendPointerSync`, `MotionEvent`, `Instrumentation`
- **Root/anti-Frida evasion:** JailMonkey bypass, `isDebuggerConnected` tamper, root hider hooks (Magisk/RootCloak/Xposed/Substrate)

### Step 6 — Behavioral correlation (on-device, live device)

```
adb shell pm list packages -3 | grep -iE "spy|track|monitor|record|stealth|parent|secure|control|guard"
adb shell settings get secure enabled_accessibility_services
adb shell dumpsys package <pkg> | grep -A20 "granted permissions"
adb shell dumpsys batterystats --charged
adb shell dumpsys notification
adb shell dumpsys device_policy
```

Cross-reference against IOC lists: [AssoEchap/stalkerware-indicators](https://github.com/AssoEchap/stalkerware-indicators) (80+ families), [mvt-project/mvt-indicators](https://github.com/mvt-project/mvt-indicators).

### Step 7 — Report

MobSF JSON report + Frida hook logs + on-device triage notes + YARA hits + any IOC matches. Correlate: what static said vs what dynamic showed vs what the device is doing.

---

## 3. Stealthy Malware Detection — On-Device Triage (6 phases)

Phase 0 — Quick screen: `pm list packages -3`, grep for spy/track/monitor/record/stealth/parent/secure/control/guard. Check accessibility services (`settings get secure enabled_accessibility_services`). Check overlay permission abuse. Check battery anomalies (`dumpsys batterystats --charged`).

Phase 1 — Enumerate ALL components (including disabled): `dumpsys package <pkg>`. Look for services with `enabled=false` but `exported=true`, receivers for hidden intents (SMS_RECEIVED, BOOT_COMPLETED, PACKAGE_INSTALLED, TIME_TICK, NETWORK_STATUS), providers with unfamiliar URIs, activities with `exported=true` and no icon. Check for `android:process=":remote"` on disabled components (hidden background services). Check multiple processes from one package (`ps -A | grep <pkg>`).

Phase 2 — Detect DCL: Decompile with apktool, grep smali for `DexClassLoader`, `PathClassLoader`, `loadClass`, `URLClassLoader`. Check assets/OBB for `.dex/.apk/.jar`. Hook DexClassLoader at runtime with Frida. Check `/data/local/tmp/`, `/sdcard/Download/`, `/sdcard/Android/data/<pkg>/` for dropped payloads.

Phase 3 — Clickers/adware/SMS fraud: Look for AccessibilityService usage (tap simulation), `sendPointerSync`/`MotionEvent`/`Instrumentation` in smali. Check notification abuse (`dumpsys notification`). Check SMS/telephony: hidden SMS_RECEIVED receiver, CALL_PHONE/SEND_SMS permissions, `SmsManager` usage. Frida hook `SmsManager.sendTextMessage`.

Phase 4 — Spyware/exfiltration: Check dangerous permission combos (READ_CONTACTS + READ_SMS + ACCESS_FINE_LOCATION + RECORD_AUDIO + CAMERA together). Check camera/mic: `CameraManager`, `MediaRecorder`, `AudioRecord`, 1x1 `SurfaceView`. Check ContentResolver queries on contacts/SMS/call_log. Check network: `HttpURLConnection`, `OkHttp`, `Socket` sending to external domains. Frida hook network/camera/SMS APIs.

Phase 5 — Persistence/stealth: Boot persistence (BOOT_COMPLETED receiver + RECEIVE_BOOT_COMPLETED). Package installation persistence (PACKAGE_INSTALLED receivers). Icon hiding (`android:label="@android:string/empty"` or no icon, `ComponentInfo` flags). Device Admin abuse (`dumpsys device_policy`). Hidden APKs (packages with `exported=false` and no launcher activity). Overlay attacks (`dumpsys activity | grep "mToken"`, TYPE_LAYER_TYPE_HARDWARE floating overlays).

Phase 6 — Network/forensic correlation: `netstat -tulpn`, `/proc/net/tcp`, `/proc/net/tcp6`, `dumpsys connectivity`. Capture PCAP if possible. Cross-reference domains/IPs against IOC feeds.

---

## 4. Exploit Identification — What Broke the Device

### Signs that warrant escalation

- Unexpected root/su access
- adbd in insecure mode (`ro.debuggable=1`, ADB on network)
- Device overheating/battery drain/network spikes with no user cause
- Unknown apps with SYSTEM or root UID
- SELinux Permissive/Disabled on a device that should be Enforcing
- Kernel panic/tombstone referencing suspicious addresses
- Network connections to unexpected IPs from system services
- Repeated crashes in security-critical components (mediaserver, keystore, binder, surfaceflinger)

### The 5-phase workflow (see Pillar 4 above)

Key nuance: many Android kernel exploits leave minimal forensic traces if they succeed cleanly (no crash, no tombstone). Post-exploitation tools like Ghost may obscure the original entry vector. If SELinux was already Permissive, the device may have been compromised before the kernel exploit stage. Attribution to a specific actor usually requires more than device artifacts alone.

### CVE reference card (partial, Android-relevant)

- CVE-2019-2215 — Binder UAF (epoll + BINDER_THREAD_EXIT), Android 8.x/9.0/early 10
- CVE-2020-0041 — Binder OOB access (sandbox escape)
- CVE-2022-20409 — io_uring LPE
- CVE-2022-0847 — Dirty Pipe (pipe buffer flag overwrite)
- CVE-2023-0266 — ALSA compat layer race (in-the-wild, Samsung)
- CVE-2023-20938 — Binder UAF (GKI 5.4/5.10, partial fix Feb 2023, full fix Jul 2023)
- CVE-2023-26083 — Mali tlstream info leak (in-the-wild, Samsung)
- CVE-2023-21255 — Binder UAF (full fix for CVE-2023-20938, Jul 2023)
- CVE-2025-21479 — Adreno GPU driver
- CVE-2026-31431 — Copy Fail (kernel copy primitive)
- CVE-2025-68260 — rust_binder (first Rust-code CVE in Linux kernel)

### Key repos for exploit identification

- [xairy/linux-kernel-exploitation](https://github.com/xairy/linux-kernel-exploitation) (6.6k stars, updated Aug 2026) — kernel exploit link index, cross-references Android CVEs
- [0xbinder/android-kernel-exploitation-lab](https://github.com/0xbinder/android-kernel-exploitation-lab) (43 stars, active Apr 2025) — CVE-2019-2215 hands-on lab (QEMU + exploit + GDB scripts)
- [cloudfuzz.github.io/android-kernel-exploitation](https://cloudfuzz.github.io/android-kernel-exploitation/) — structured workshop on CVE-2019-2215
- [sfewer-r7/pocindex](https://github.com/sfewer-r7/pocindex) — 82k PoC aggregation, JSON API
- Google Project Zero blog — in-the-wild Android exploit analyses
- [Android OffSec blog](https://androidoffsec.withgoogle.com/) — Google's Binder/kernel exploit writeups (e.g., CVE-2023-20938)

### drozer + Ghost in the chain

drozer maps the app-layer IPC attack surface BEFORE any kernel exploit. Ghost represents the post-exploit phase AFTER gaining shell. A typical full chain: malicious app/browser RCE → app-layer IPC escape or kernel CVE → root → Ghost/ADB post-exploitation. Forensically, you go backwards: post-exploit artifacts (Ghost process) → kernel artifacts (tombstones, logcat) → app-layer artifacts (exported components, suspicious packages) → entry point + CVEs used.

---

## 5. Shizuku/Dhizuku — Deep Operational Notes

### Starting Shizuku

**Via USB ADB:** `adb shell sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh`

**Via wireless debugging:** Android 11+ built-in Wireless Debugging → pair → Shizuku app → "Start via Wireless debugging"

**Via root:** `su -c sh /data/adb/shizuku/start.sh` (or Sui Magisk module)

**Verify:** `adb shell dumpsys activity service moe.shizuku.privileged.api | head` or `adb shell service list | grep shizuku`

### Shizuku in terminal apps (rish)

Enable "Use Shizuku in terminal apps" in Shizuku settings → downloads rish → remote privileged shell backed by Shizuku. On Android 14+: `adb -s $target shell /data/app/~~<hash>/moe.shizuku.privileged.api-<hash>/lib/arm64/libshizuku.so` (new path, per Shizuku discussions).

### Shizuku vs Dhizuku vs root

- Shizuku (ADB): UID 2000 shell. Most pm/dumpsys/am/appops/logcat. Cannot read another app's private data.
- Shizuku (root): UID 0. Full filesystem, kernel, SELinux.
- Dhizuku: Device Owner axis. Install/uninstall/managed profiles. Different from ADB axis. Mutually exclusive with other DO apps.
- Root (direct): UID 0, full control.

### Security model implications

Treat Shizuku-compatible tooling as ADB-equivalent in threat models. The combination of Accessibility + Wireless debugging + shell-backed Binder helpers is a high-risk chain, not independent low-severity events. Monitor for Binder services exposing `moe.shizuku.privileged.api`. Enforce work-profile/MDM restrictions that remove debugging features from managed users. Disable USB/Wireless debugging on production devices.

### Notable Shizuku-powered tools (security/forensic angle)

From awesome-shizuku and topic search: ShizuCoreFetch (app manager), Buge-App-Manager (app/permission management), AndroSH (multi-distro Linux via Shizuku+proot — Arch/Fedora/Alpine/Debian/Ubuntu/Kali/Void/Manjaro/OpenSUSE/Chimera), Shizuku-Keeper (keep Shizuku alive via Automate), ShizukuShortcuts (launcher shortcuts + custom shell actions), MonProject (system optimizer, root + non-root Shizuku), various file managers, log viewers, package managers, debloaters.

---

## 6. Frida — Advanced Techniques (beyond basic hooking)

### Two deployment modes

1. **frida-server on rooted device:** Push binary to `/data/local/tmp/`, chmod 755, run in background. Simpler, works on rooted devices. Detectable (frida-server process, port listening, known paths).
2. **frida-gadget repackaging (non-root):** Decompile APK with apktool, place correct `libfrida-gadget.so` in APK, re-align with zipalign, resign with jarsigner, install. App loads gadget on launch, connects to Frida client remotely. More complex, requires resigning, but works on non-rooted devices.

### Anti-Frida/anti-root detection — what apps check and how to bypass

**Root detection:** Magisk presence, su binary, root package names, `getProp` checks, build tags. Bypass: root hide hooks (Magisk, RootCloak, Xposed, Substrate), `getPackageInfo` hook to fake package names, return false for all root checks.

**Frida detection:** Suspicious libraries loaded, unusual system calls, process injection, port listening, known frida-server paths, `libc.so` hook patterns. Talsec freeRASP is the #1 RASP solution by popularity — detects frida-server regardless of port/communication method, also blocks Objection patches. [Talsec Frida article](https://docs.talsec.app/appsec-articles/articles/hook-hack-defend-fridas-impact-on-mobile-security-and-how-to-fight-back).

**Emulator detection:** Build properties (model, brand, product names), sensor presence, specific hardware IDs. Bypass: fake device properties (codeshare.frida.re/@fdciabdul/frida-multiple-bypass).

**SSL pinning:** TrustManagerImpl, WebViewClient, OkHttp, Apache HTTP client, iOS NSURL, Flutter (uses自己的certificate validation, ignores system CA). Bypass: universal SSL pinning bypass codeshare scripts, httptoolkit android-disable-certificate-pinning.js, httptoolkit android-disable-flutter-certificate-pinning.js (experimental).

### Native code hooking

Hook native exports with `Interceptor.attach(Module.findExportByName("libnative-lib.so", "target_function"), { onEnter, onLeave })`. Read/write arguments, modify return values. For JNI, hook the Java method that calls the native function, or hook the native export directly.

### Persistent instrumentation

- frida-gadget in APK (survives app restart, but requires repackaging)
- Shizuku UserService (run your own code as UID 2000/0 in a separate process)
- Magisk module + Sui (root-mode Shizuku, survives reboot)
- Automate flows (Shizuku Keeper — keep Shizuku alive without root/cables)

### Staying ahead of app defenses

- Apps adapt: new root detection methods, new Frida detection, new pinning approaches.
- Defense-in-depth for instrumentation: combine multiple bypass techniques, don't rely on a single script.
- Test on the specific app/version: generic scripts may not cover app-specific checks.
- When generic bypass fails: static analysis to find the exact check, then write a targeted hook.

---

## 7. Deobfuscation & Unpacking — When the APK Hides Itself

### Obfuscation tiers

1. **ProGuard/R8 (rename-only):** Class/method/field name shortening, unused code removal. Not real security — JADX often reconstructs readable enough code. Easy to reverse.
2. **String encryption, control flow flattening, anti-debug:** Added on top of ProGuard/R8. Requires Smali-level analysis or automated deobfuscation.
3. **Commercial obfuscators (DexGuard, etc.):** Class encryption, string encryption, control flow obfuscation, native code protection, anti-tamper, anti-debug, anti-Frida. Hard. May require commercial deobfuscators or manual Smali work.
4. **Native code obfuscation (NDK):** C/C++ in .so files. Control flow flattening, string encryption, anti-debug in native code. Requires GDB/IDA/Ghidra + native analysis skills.

### Detection approach

1. **Identify the obfuscator:** Look at APK structure (multiple .dex files? encrypted classes?obfuscated Smali patterns). MobSF may detect known obfuscators. Check for known packer signatures.
2. **Static deobfuscation (if possible):** Some obfuscators have community deobfuscators. Search GitHub for "<obfuscator name> deobfuscator". Automated deobfuscation frameworks exist for common patterns.
3. **Manual Smali-level work:** When automated tools fail, read the Smali. Look for: string decryption routines (often a static method that takes an int and returns a string), reflection-based class loading, dynamically constructed method names. Trace the decryption key (often a constant or derived from app state).
4. **Dynamic analysis (Frida):** Hook the decryption routine to dump decrypted strings at runtime. Hook class loading to see what's actually loaded. Hook reflection calls to see what's being invoked.

### Anti-analysis tricks and bypasses

- **Anti-debug:** `isDebuggerConnected()`, `ptrace` checks, timing checks. Bypass: hook `isDebuggerConnected` to return false, hook `ptrace` to return 0, tamper timing checks.
- **Anti-Frida:** Detect Frida server, detect injected libraries, detect JavaScript bridge. Bypass: custom Frida scripts that hide the injection, undetectable Frida server techniques (Talsec RASP detects most).
- **Tamper detection:** Check APK signature at runtime, check file integrity, check for root/modify. Bypass: hook signature check methods, hook file existence checks, root hide.

### Case study: BRATA/Ghimob two-stage dropper

BRATA: lightweight dropper that silently downloads and installs the real malware → two apps on device. Ghimob: downloader app installs payload after launch. Both use DCL to hide malicious code until after installation and user trust.

### Case study: Joker malware

Visible code looks harmless; during execution loads hidden DEX file and runs `yin.Chao` via reflection; that class downloads another JAR from remote server → multistage infection chain.

---

## 8. Vulnerability Discovery — Beyond drozer

### Common vulnerability patterns in Android apps

1. **Exported components without permission guards:** Activities, services, content providers, broadcast receivers with `android:exported="true"` and no permission required. drozer `app.package.attacksurface` finds these. Test: can another app invoke them? Can they be abused for intent injection, data access, privilege escalation?

2. **Intent injection:** Components that accept intents and act on them without validating the source. E.g., an exported activity that reads extras and performs an action (send SMS, make call, open URL). Test: send crafted intents from a test app.

3. **Insecure content providers:** Exported providers with no permission, SQL injection via URI, path traversal, readable/writable data that shouldn't be. drozer content provider modules test these.

4. **Insecure DEX loading (DCL):** Loading code from external storage, network, or assets without integrity verification. Android 10+ restricts native library loading from internal storage, but DEX loading still possible. Google Play policies prohibit downloading executable code from untrusted sources.

5. **VPNService abuse:** Apps using VPNService to intercept all traffic without clear user consent or legitimate purpose. Can be used for MITM, ad injection, data collection. Check: does the app have VPNService declared? What does it do with the traffic?

6. **AccessibilityService abuse:** Apps requesting accessibility permissions without legitimate need. Can simulate taps, read screen content, intercept input. Malware uses this for tap simulation (clickers), screen reading (spyware), input interception.

7. **Deep link/URL scheme vulnerabilities:** Intent filters that handle URLs without proper validation. Can lead to account takeover, OAuth token theft, deep link hijacking. [Oversecured deep link article](https://oversecured.com/blog/android-deep-link-vulnerabilities).

### Systematic testing methodology

1. **Reconnaissance:** `pm list packages`, `dumpsys package <pkg>`, manifest inspection, permission audit. Map the attack surface.

2. **Static analysis:** MobSF, JADX, apktool. Look for exported components, dangerous permissions, suspicious API usage, hardcoded secrets, certificate pinning.

3. **Dynamic analysis:** Frida/Objection hooks for app-specific behavior. Network interception (httptoolkit scripts). Monitor file system, network, and process behavior.

4. **IPC testing:** drozer for exported component enumeration and testing. Custom test apps for intent injection, content provider interaction.

5. **Exploit chain analysis:** If you find an app-layer vuln, consider: can it be chained with anything else? Does it give higher privilege? Can it be combined with a kernel exploit (if rooted)?

6. **Responsible disclosure:** Find the vendor's security policy. Check bug bounty programs (HackerOne, Bugcrowd, vendor-specific). Follow coordinated disclosure. For Android system issues: [source.android.com/security/overview/updates-resources.html#report-issues](https://source.android.com/security/overview/updates-resources.html#report-issues). Google Android Security Reward Program for qualifying bugs.

### Bug bounty programs targeting Android

- Google Android Security Reward Program (android-rewards)
- Vendor-specific programs (Samsung, Xiaomi, etc.)
- App-specific programs (banking apps, fintech, social media)
- HackerOne/Bugcrowd programs with Android scope

### Reference frameworks

- **OWASP Mobile Application Security (MASTG):** [owasp.org/www-project-mobile-app-security](https://owasp.org/www-project-mobile-app-security/) — comprehensive testing guide, vulnerability categories, test cases.
- **OWASP Mobile Top 10:** M1: Improper Platform Usage, M2: Insecure Data Storage, M3: Insecure Communication, M4: Insecure Authentication, M5: Insufficient Cryptography, M6: Insecure Authorization, M7: Client Code Quality, M8: Code Tampering, M9: Reverse Engineering, M10: Extraneous Functionality.

---

## 9. Current Threat Landscape (2025-2026)

### What's active right now

**Banking trojans dominate:** Kaspersky Q2 2026 — banking trojans 30.77% of detected mobile malware. 304,000+ malicious installation packages in Q2 2026 alone. Mamont family: 49.8% of all banking trojan packages in 2025, surging in Q1-Q2 2026 (Mamont.hl 11.13%, Mamont.iv 7.33%). Creduz family: 22.5% of banking trojans. New families: Vultur, DroidBot, Errorfather, BlankBot, FvncBot (Poland, Nov 2025), RatOn (NFC relay + ATS, Jul 2025).

**Adware is highest volume:** 62% of Android detections. MobiDash grew 100%+ monthly in 2025. HiddenAd and MobiDash declining in Q2 2026, but dropper reclassification shifting numbers.

**Trojan droppers are the delivery mechanism:** Banker droppers (Trojan-Dropper.AndroidOS.Banker, Trojan-Dropper.AndroidOS.Mamont) surging. Several banking trojans being packed as droppers — shift in tactics.

**NFC relay fraud (ghost tapping):** NGate (2025), SuperCard X (2025), WindRelay (Aug 2026 — SpyNote RAT + WindRelay NFC relay, real-time card usage). 13-minute bank impersonation call → install RAT → install NFC relay → tap card → PIN → criminals use card remotely at terminal/ATM. SpyNote + WindRelay combo is the latest evolution.

**Play Store poisoning:** Anatsa — fake "Document Viewer - File Reader" app, 90,000+ downloads, #4 in Top Free Tools, activated payload 6 weeks after publication (May 2025). Targeted North American banking. ToxicPanda: expanded from SE Asia to Europe, 4,500+ infected devices (Portugal 3,000, Spain 1,000), Samsung/Xiaomi/Oppo dominance.

**Regional targeting:** Turkey — Coper trojan (96.35% of attacked users Q3 2025). Brazil — Pylcasa (88.25% of attacked Brazilian users, disguised as calculator apps on Play Store). Poland — FvncBot (mBank security key lure).

**Preinstalled malware:** Triada variants (Triada.fe, Triada.gn, Triada.ii) — preinstalled backdoors on newly purchased Android devices, full device control before user opens the box. Triada.ag was #1 in Kaspersky Q2 2026 rankings (7.09% of affected users).

**Spyware campaigns:** Citizen Lab/Amnesty International Security Lab — commercial spyware vendors, zero-click exploits, targeted campaigns. MVT (Mobile Verification Toolkit) from Amnesty International for IOC-based device forensics.

**New spyware families (2025-2026):** ClayRat — mass-scale self-propagating Android spyware targeting Russian users, 700+ unique APKs in 3 months, SMS + Accessibility dual abuse, keylogger, screen recording, fake overlays, anti-uninstall. Arsink RAT — cloud-native mass spyware abusing Google Firebase + Apps Script + Drive + Telegram for C2, 1,216 APKs, 45K IPs across 143 countries, takedown coordinated with Google. PromptSpy (Feb 2026, ESET) — first Android malware using generative AI (Google Gemini) at runtime for dynamic, device-agnostic persistence gestures; proof-of-concept but distribution domain exists. Albiriox (Dec 2025) — MaaS Android banking RAT + remote access Trojan with live screen streaming, on-device fraud, 400+ target apps.

**NFC relay fraud evolution:** NGate (NFCGate-based, ATM cash withdrawal), Ghost Tap (scaled deployment), WindRelay + SpyNote combo (13-minute phone call chain: social engineering → SpyNote install → remote access → WindRelay install → card tap + PIN → NFC relay → loan fraud + card cloning, targeting Czechia/Slovakia/Slovenia, ~24 samples on VT Nov 2025-Jul 2026). Pattern: social engineering + device takeover + real-time NFC relay.

**Banking trojans (2025-2026):** Zimperium 4 campaigns (RecruitRat, SaferRat, Astrinox/Mirax/Medusa, Massiv) — 800+ targeted apps, multi-stage installation with Session Installation API abuse, droppers disguised as Play Store updates, near-zero signature detection, 4 operational variants. Medusa/Cleay advanced banking trojan. Ghimob — Brazilian RAT expanding internationally (Brazil, Paraguay, Peru, Portugal, Germany, Angola, Mozambique). Triada detections more than doubled in H2 2025.

**C2 infrastructure trends (2025-2026):** Telegram Bot API as C2 channel (dominant — ClayRat, Raven Stealer, CyberEye RAT, PE32 ransomware, Arsink variant). Google Firebase + Apps Script + Drive abuse for C2/exfiltration (Arsink — 317 Firebase RTDB endpoints, 774 samples with Apps Script uploader). Multi-infrastructure redundancy (4 Arsink variants with different exfiltration paths). Encrypted C2 + dynamic payload loading (ClayRat AES/CBC+GCM, Arsink Firebase upload-based).

**Macro statistics (full year 2025):** Kaspersky: 14.06M attacks on Android devices, 815,735 new malicious packages, 255,090 banking trojan packages (nearly 4x increase globally). Malwarebytes H2 2025 vs H1: adware +90%, PUP +75%, malware +20%. Banking trojans targeting 1,243 financial institutions across 61 countries (+67% YoY, Zimperium). India dominant target market (Rewardsteal/UdangaSteal/Agent.uq = 94.71% of attacked users full year). Regional specialization: Turkey (Coper 96.35%), Germany (Rkor.ii 76.90%), Brazil (Pylcasa via Play Store disguised as calculator).

**New spyware families (late 2025-2026):** ClayRat — mass-scale self-propagating Android spyware targeting Russian users, 700+ unique APKs in 3 months, SMS + Accessibility dual abuse, keylogger, screen recording, fake overlays, anti-uninstall. Arsink RAT — cloud-native mass spyware abusing Google Firebase + Apps Script + Drive + Telegram for C2, 1,216 APKs, 45K IPs across 143 countries, takedown coordinated with Google. Both represent a shift toward cloud-infrastructure abuse for C2/exfiltration rather than traditional dedicated C2 servers.

**AI-powered malware:** PromptSpy (ESET, Feb 2026) — first known Android malware using generative AI (Google Gemini) in its execution flow, using AI to dynamically determine persistence gestures (keeping app locked in recent apps) in a device-agnostic way. Proof-of-concept but distribution domain exists. Signals a new evasion paradigm: AI-guided behavior replacing hardcoded UI selectors.

**MaaS banking RATs:** Albiriox (Dec 2025) — full-featured Android banking RAT + remote access Trojan sold as Malware-as-a-Service, live screen streaming to attacker, on-device fraud via victim's own banking session, 400+ target apps. Lowers barrier for entry-level criminals.

### IOC sources

- [AssoEchap/stalkerware-indicators](https://github.com/AssoEchap/stalkerware-indicators) — 80+ stalkerware/spyware families, package names, C2 domains, affiliate infrastructure
- [mvt-project/mvt-indicators](https://github.com/mvt-project/mvt-indicators) — Amnesty International IOC lists for spyware campaigns
- Kaspersky Securelist mobile threat reports (Q1/Q2/Q3/Q4 each year)
- Zimperium Banking Heist Report (annual, 34 families tracked in 2026, 1,243 financial institutions, 90 countries)
- MalwareBazaar ([bazaar.abuse.ch](https://bazaar.abuse.ch/)) — malware samples, YARA matches, statistics
- AndroZoo ([androzoo.uni.lu](https://androzoo.uni.lu/)) — Android app collection, Google Play + other sources
- VirusTotal — hash reputation, AV detections, behavioral reports

---

## 10. Quick Reference — Commands You'll Actually Run

### Device access & verification

```
adb devices -l                          # connected devices
adb shell getprop ro.debuggable         # is ADB debuggable?
adb shell getprop ro.build.fingerprint  # device fingerprint
adb shell getprop ro.build.version.security_patch  # patch level
adb shell getprop ro.build.version.release        # Android version
adb shell uname -a                      # kernel version
adb shell getenforce                    # SELinux state
adb shell service list | grep shizuku  # Shizuku running?
adb shell dumpsys activity service moe.shizuku.privileged.api | head
```

### Starting Shizuku

```
adb shell sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh   # USB ADB
adb shell /data/app/~~<hash>/moe.shizuku.privileged.api-<hash>/lib/arm64/libshizuku.so   # Android 14+ new path
su -c sh /data/adb/shizuku/start.sh                                   # root
```

### Package inspection

```
adb shell pm list packages -f              # all packages with paths
adb shell pm list packages -3              # third-party only
adb shell pm list packages -s              # system
adb shell pm list packages -e              # enabled
adb shell pm list packages -d              # disabled
adb shell pm path <package>                # APK location
adb shell dumpsys package <package>        # full package info
adb shell dumpsys package <package> | grep -A20 "requested permissions"
adb shell dumpsys package <package> | grep -A20 "granted permissions"
aapt dump badging app.apk                  # manifest summary
```

### App interrogation

```
adb shell pm grant <pkg> <permission>      # grant permission
adb shell pm revoke <pkg> <permission>     # revoke permission
adb shell appops get <pkg>                 # app operations
adb shell dumpsys activity                 # activity stack, overlays
adb shell dumpsys notification             # notifications
adb shell dumpsys device_policy            # device admins
adb shell settings get secure enabled_accessibility_services   # accessibility services
```

### Forensics & logging

```
adb logcat -d -v time -b all > logcat_full.txt
adb logcat -d -b crash > logcat_crash.txt
adb logcat -d -b system > logcat_system.txt
adb logcat -d -b main   > logcat_main.txt
adb shell dumpsys > dumpsys_full.txt
adb shell dumpsys meminfo > dumpsys_meminfo.txt
adb shell dumpsys netstat > dumpsys_netstat.txt
adb shell ps -A -o pid,uid,comm,args > ps_snapshot.txt
adb shell netstat -tulpn > netstat.txt
adb pull /data/tombstones/ ./tombstones/
adb shell screencap -p /sdcard/screen.png && adb pull /sdcard/screen.png
adb bugreport > bugreport.txt
```

### Static analysis

```
# MobSF: web UI or CLI
# Androguard: androlyze CLI or Python API
jadx-gui app.apk                            # decompile to Java
jadx -d out_java app.apk                    # headless
apktool d app.apk -o app_decoded           # decode resources + smali
aapt dump badging app.apk                   # manifest summary
aapt dump permissions app.apk               # permissions
unzip -l app.apk | grep -E '\.so$'         # native libraries
unzip -l app.apk | grep -E '\.dex|\.apk|\.jar'  # code/assets
grep -r "DexClassLoader\|PathClassLoader\|loadClass" app_decoded/smali/
grep -r "sendPointerSync\|MotionEvent\|Instrumentation" app_decoded/smali/
grep -r "CameraManager\|MediaRecorder\|AudioRecord\|SurfaceView" app_decoded/smali/
grep -r "ContentResolver\|query.*contacts\|query.*sms\|query.*call_log" app_decoded/smali/
grep -r "http\|https\|socket\|OkHttp\|Retrofit" app_decoded/smali/
```

### Dynamic instrumentation

```
# Push frida-server (root or gadget mode)
adb push frida-server /data/local/tmp/
adb shell "chmod 755 /data/local/tmp/frida-server"
adb shell "/data/local/tmp/frida-server &"

# List processes
frida-ps -U

# Trace a function
frida-trace -U -i open -N com.android.chrome

# Attach with custom script
frida -U -f com.target.app -l hook.js

# Codeshare script (universal root + SSL pinning bypass)
frida --codeshare ssecurityy/universal-robust-advanced-root--ssl-pinning-bypass -f com.target.app

# Objection
objection explore com.target.app

# SSL pinning bypass via httptoolkit
frida -U -f com.target.app -l android-disable-certificate-pinning.js
```

---

## 11. What's Covered in the Extended Corpus (companion files)

**Safe encrypted storage:** [android-safe-encrypted-storage.md](../../../android-safe-encrypted-storage.md) covers Android Keystore, authenticated envelope encryption, backup/restore, key invalidation, and migration away from deprecated `EncryptedFile`/`EncryptedSharedPreferences`.

The skill above is the integrated operational reference. The following companion research files on disk go deeper on specific topics and are referenced throughout. Load them with `read_file` when you need the detailed version.

| File | Lines | Size | Key content |
|------|-------|------|-------------|
| `android-native-re-workflow.md` | 493 | ~20KB | Native `.so` RE: Ghidra/IDA/JEB/radare2/angr tool list, JNI bridge patterns, step-by-step workflow with commands, malware native-library pattern recognition (string decryption, crypto, file I/O, network, anti-analysis), anti-analysis bypass notes (ptrace, TracerPid, Frida maps, timing, signal handlers), Frida native hooking patterns with JS examples (Interceptor.attach/replace, Memory.scan, Module introspection, NativeFunction, Java+native bridge, byte-pattern hooks, Frida detection bypass). |
| `shizuku-exploitation-research.md` | 324 | ~27KB | Shizuku attack surface: 5 concrete privilege escalation/abuse scenarios (malicious app obtains access, Shizuku-enabled app as high-value target, UserService abuse for persistence/escalation/stealth/resource, cross-app attacks, Wireless Debugging + Shizuku network exposure), UserService exploitation methodology, malicious client crafting notes, defensive audit checklist (app-level, system-level, ecosystem, incident response signals). Notes no major public Shizuku CVEs as of cutoff but documents the attack surface. |
| `android-commercial-protector-bypass.md` | 423 | ~39KB | Commercial protector landscape: DexGuard (Guardsquare), Bangcle (Aliyun/360), Liuling, DarkGuard/DarkGray, APKProtect, ARPSolo/ARPDS, Obfuscator-LLVM based, R8/ProGuard plus commercial layers, custom/in-house. Each with protection layers, detectability, market presence. Bypass approaches by protector (DexGuard: Frida DexClassLoader hooks, BlackDex, native loader hooks, memory dumping — no universal unpacker; Bangcle/Liuling: Chinese RE community, dynamic extraction, version-locked scripts). Manual unpacking methodology (identify, find entry point, trace decryption, hook/patch to dump, reconstruct, iterate). Detection via APKiD + manual. Case studies (Packed Unpacker Black Hat talk). Hard problems: protectors update regularly, no silver bullet, per-app manual work often required. |
| `android-14-15-16-security-changes.md` | 248 | ~21KB | Android 14 security changes by feature area: scoped storage enforcement, restricted permissions (POST_NOTIFICATIONS, exact alarms), photo picker/metadata privacy, foreground service types, pending intent mutability (FLAG_IMMUTABLE default), background activity start restrictions, exported component enforcement, non-SDK interface restrictions, WebView/Bluetooth/framework hardening, Security by Default. Android 15: continued privacy/storage/permission tightening, memory safety direction (Rust), AccessibilityService restrictions, WebView isolation. Android 16 early direction: continued security hardening, scoped storage, permissions, memory safety (source: developer.android.com version summary pages). **Android 16+ Advanced Protection Mode / Intrusion Logging (May 2026) section added 2026-09-04.** Cross-version trend analysis: scoped storage, permission granularity, security-by-default, hidden API shrinkage, memory safety — what's getting harder for attackers vs analysts. Practical impact on Red Team/malware analysis/pentesting: what stays the same, what needs adaptation, recommendations. Sources: developer.android.com version pages, Oversecured Samsung preinstalled app vuln research. |
| `android-re-frida-pipeline.md` | 191 | ~15KB | Literature survey on RE/Frida tooling across multiple search threads. Covers: decompilation/DEX analysis (JADX 50k stars, apktool, dex2jar, baksmali, dexlib2, bytecode-viewer, Androx, Androguard, smali/baksmali, dexlib2/dexlib), static analysis (MobSF 22k stars, APKiD, APKLab, apk signature verification, Androwarn, AndroBugs, VirusTotal, Kaspersky, Pithus), dynamic instrumentation (Frida 22k stars, objection 9.8k stars, Frida native hooking, Memory.scan, Interceptor.attach, Objection explore, Frida-codeshare scripts), runtime manipulation (Magisk, systemless root, busylightx/fridantiroot, frida multiple bypass scripts, root hide techniques), anti-analysis (APK signature checks, anti-tamper, SSL pinning, root/emulator detection, anti-Frida, obfuscation and anti-reversing, DexGuard, Bangcle, Liuling, APKProtect, control flow flattening, string encryption, anti-debug, anti-tamper detection). |
| `android-exploit-chains.md` | 202 | ~14KB | Full exploit chain analysis: Project Zero Pixel 9 AWDL/Dolby 0-click chain (CVE-2025-48593 FontParser remote code execution, CVE-2026-0006 CVE-2025-14035 heap overflow, CVE-2025-48596 arbitrary read/write, CVE-2025-48594 + CVE-2026-00710 mistranslation bugs, CVE-2025-48602 + CVE-2025-48605 info leak, CVE-2025-48599 + CVE-2026-00692 kernel heap grooming crashes), in-the-wild Android post-exploitation campaigns (Project Zero Jan/Feb 2025, Android 12/13 victim base, CVE-2023-40074 Kotlin compiler CVE, CVE-2021-30461 root downgrade, CVE-2023-20938 + CVE-2023-2136 Android 13 patch gap), Chrome RIDL sandbox escape (CVE-2025-2495, 25 RCEs in 2024, leaked pointers for info leak + infoleak for kernel crash), Amnesty International Pegasus forensic methodology (MVT-powered IOC extraction, SQLite forensic queries, device state analysis, iOS artifacts, Android forensic pathway), Citizen Lab forced entry device analysis (Pegasus re-installation, E2C daemon, Android 12 extraction, sqlite3 backup artifact, SSH logs), Oversecured Android app-layer vulnerabilities (97 in-app browsers, 47 WebView, 30 file providers, deeper intent/component/deep link/UNIX socket/qQm poses), Google Android OffSec Blog Binder CVE exploitation (CVE-2023-20938 ioctl UAF, patched Jul 2023 full fix, chained with CVE-2023-2136 for KASLR bypass), GitHub Fugitive-in-Java automated Android exploit chain (CVE-2024-00448, CVE-2024-29059, CVE-2024-0044, UAF chaining for arbitrary code execution), Cyfirma GhostShell/Operation Crocodile Gang analysis (RATs, C2 infrastructure, SMS interception, credential harvesting, mobile RAT distribution). Chain stages annotated with tooling fit. drozer + Ghost positioning. Attribution caveats. |
| `android-malware-c2-infra-analysis.md` | 123 | ~7KB | C2 infrastructure analysis for Android malware: specialized tooling (eml2http for email/EML reconstruction to HTTP, ReversingLabs RLater for malware analysis pipeline), infrastructure tracking (Elastic cybercrime tracker for infrastructure mapping across families), concept indexes (awesome-command-control for C2 architecture understanding), practical analysis sessions with APK analysis (MobSF, JADX, Androguard, APKLeaks), C2 infrastructure patterns (HTTPS to normal-looking domains, public infrastructure abuse, short-lived infrastructure, blended/low-volume beaconing, multi-stage retrieval), practical workflow (extract indicators, reconstruct/capture traffic, correlate infrastructure, map to campaign/operator patterns), sandbox/dynamic analysis tooling (instrumented/emulated execution, dynamic instrumentation for network visibility, static pre-analysis, network capture + HTTP reconstruction), gaps (no dominant OSS Android malware sandbox with polished C2 mapping, workshop-style/piece-assembled workflow, vendor tools carry much of the load), lightweight frameworks worth knowing (MobSF, Androguard, Frida MITM/unpinning, APKiD + APKLeaks-style checks). Bottom line: treat as process not tool — extract, make visible, correlate, map. |
| `android-kernel-exploit-dev-methodology.md` | 777 | ~54KB | Advanced Android kernel exploit development methodology: CVE-to-exploit workflow (bug classification UAF/OOB/race/info leak/double-free/type confusion, primitive assessment with size-specific examples, target selection), KASLR defeat on Android (info leak sources: syscall returns, slab content, GPU/driver ioctls, UAF reads, /proc/kallsyms; leak→base→symbol→write chain), heap Feng Shui on ARM64 SLUB (size classes, concrete grooming sequence: spray → free every other → allocate controlled → trigger, cross-cache attacks, spraying vs grooming tradeoffs), arbitrary r/w primitive construction (UAF→r/w, OOB read→leak, OOB write→corrupt, race→bypass, limited→full chaining, full target list: cred, init_cred, prepare_creds/commit_creds, syscall table, function pointers, task struct), Android mitigation landscape (KASAN, KCFG, CFI, PAN, STATIC_USERMODE_HELPER, KPTI, module signing — why production has a subset), ARM64 specifics (PAC impact on function pointers, calling conventions x0-x30, ARM64 vs ARM32), cred manipulation (init_cred, prepare_creds/commit_creds, overwrite current->cred, UID/GID manipulation, SELinux considerations), case studies of recently written Android kernel exploits 2023-2026 (CVE-2019-2215 binder UAF hello world, CVE-2023-20938 + CVE-2023-2136 chain, CVE-2023-26083 Mali info leak, CVE-2025-21479 Adreno driver, CVE-2022-20409 io_uring, CVE-2022-0847 Dirty Pipe, CVE-2023-0266 ALSA race), tooling (QEMU vulnerable kernel, GDB kernel debugging, KASAN builds, kernel config analysis, syzkaller, kallsyms/ksymtab), writing robust exploits (reliability: grooming over spraying, per-CPU state, races; version detection; fallback paths; PoC vs production), CVE reference table. |
| `android-kernel-exploit-lab-setup.md` | 365 | ~13KB | Three concrete Android kernel exploit lab setups: 0xbinder/android-kernel-exploitation-lab (CVE-2019-2215, QEMU + goldfish 4.14 kernel + GDB kernel privesc scripts + exploit code with full end-to-end command sequence), cloudfuzz/android-kernel-exploitation (Docker-based build variant, separate exploit codebase, syzkaller integration docs), locus-x64/android-kernel-exploitation-lab (single-Dockerfile variant with volume-mounted output). Also covers kernelCTF (Google VRP), xairy/linux-kernel-exploitation index, OffensiveCon 2022 Android Kernel Security training, syzkaller fuzzing labs, Exploit-DB PoC, Grant Hernandez tailoring writeup, DUASYNT and Mobile Hacking Lab paid courses. Hardware/VM requirements (40GB disk, 8GB+ RAM, 4+ cores, nested virt for KVM acceleration). Cross-architecture note (x86_64 goldfish target, arm64 TODO). Quick-reference: full end-to-end command sequence from clone to root shell. |
| `android-malware-attribution-campaign-tracking.md` | 967 | ~65KB | Android malware attribution & campaign tracking methodology: the attribution problem (what CAN be determined from samples: family relationships, campaign timeframe, infrastructure footprint, monetization mechanism, targeting; what CANNOT: human operator, location, affiliation, full campaign scope, financial beneficiaries; what attribution claims actually require: law enforcement, intelligence, legal process, victim reports, financial investigation; honest framing), sample-to-campaign linking (static features: package names, certificate hashes, code similarity Smali/strings/API/native with SimHash/ssdeep/Binary Ninja diffing, resource patterns, manifest structure; dynamic behavior: C2 infrastructure, beaconing patterns, exfiltration targets, capabilities profile; clustering: feature-based, infrastructure-based, behavior-based, multi-modal; known family identification with vendor name reconciliation), infrastructure tracking (passive DNS/WHOIS history, SSL certificate tracking via CT logs crt.sh, hosting/infrastructure patterns, account reuse signals, infrastructure lifecycle: register→configure→use→abandon/repurpose, signal strength table: cert/C2/SSL/WHOIS registrant high weight, hosting/IP lower), financial fraud tracing from mobile malware (money mule networks, cash-out methods: bank transfers, SMS toll fraud, ad fraud, cryptocurrency; SMS toll fraud tracing step-by-step; what malware analysis contributes vs what requires external access; crypto on-chain tracing with Chainalysis/Elliptic/TRM), threat actor attribution (code reuse as signal, TTPs for Android malware: protector choice, C2 protocol design, anti-analysis patterns, infection chain, monetization, targeting; infrastructure overlap; targeting patterns; language/weaponization clues; connections to broader ecosystem; 3-level confidence framework: low=family, medium=campaign, high=actor; how to read vendor attribution reports), campaign timeline reconstruction (sample collection dates, versioning/evolution, C2 infrastructure lifecycle, victimology patterns, 5 campaign phases: launch/growth/maturity/decline-reuse; limitations of biased sample collection), family classification methodology (same family vs related-but-distinct vs new family decision criteria, vendor name reconciliation), tools and data sources you can actually use (VirusTotal API, MalwareBazaar free+API, AndroZoo large-scale collection with metadata, YARA+clustering scripts, crt.sh/Google CT certificate transparency search, SecurityTrails/PassiveTotal/RiskIQ/Farsight WHOIS+DNS history, SimHash/ssdeep fuzzy hashing, MISP/OpenCTI/AlienVault OTX open threat intel platforms, Google Play Protect/Android Security Bulletins, vendor threat intel feeds), case study references (Kaspersky Securelist: Mamont/Creduz/banking trojans/spyware with campaign tracking; Zimperium Banking Heist Report: financial fraud angle; Lookout mobile threat research; Check Point Research: JSCeal methodology; Trend Micro mobile threat reports; ESET Android malware research; Palo Alto Unit 42; CrowdStrike mobile threat intel; Microsoft Security Android malware; Google TAG in-the-wild exploitation; academic literature on family classification/clustering), open-source tooling workflow for attribution (11-step practical workflow: get sample → extract static features → extract dynamic features → search VT → search MB → search AndroZoo → search CT → search WHOIS/DNS → write YARA → cluster → document campaign), honest limitations (attribution is hard and often wrong; operators use false flags/shared infrastructure/bought/sold infrastructure/copied code; sample access is biased; infrastructure is ambiguous; code reuse is ambiguous; family naming is inconsistent across vendors; attribution claims require evidence beyond samples; state actor attribution from mobile malware samples alone is insufficient — requires broader investigation), **Section 12 added 2025-2026: 5 case studies — ClayRat (self-propagating spyware, 700+ APKs, SMS+Accessibility, Russian users), Arsink RAT (cloud-native mass spyware, 1,216 APKs, 45K IPs, 143 countries, Google Firebase+Apps Script C2, takedown with Google), PromptSpy (first AI-powered Android malware, Google Gemini for persistence, ESET Feb 2026), Albiriox (MaaS banking RAT, live screen streaming, on-device fraud, 400+ target apps, Dec 2025), Zimperium 4 campaigns (RecruitRat/SaferRat/Astrinox-Mirax-Medusa/Massiv, 800+ targeted apps, multi-stage installation, Session Install API abuse, near-zero signature detection).** |
| `android-protector-walkthroughs.md` | 837 | ~82KB | End-to-end unpacking case studies and forensic narratives for commercial-protected Android malware. Section 1: why walkthroughs matter more than tooling lists (craft vs catalog, failure recovery loops, technique propagation). Section 2: narrative template for documenting your own unpacking (8-part: sample ID with hash+source+initial triage, protector ID with APKiD/manual/variant clues+confidence, initial static analysis before unpacking: manifest/partial DEX/native libraries/resources/API usage, approaches attempted with reasoning, what worked specifically, what didn't work and why: anti-debug blocking Frida/anti-tamper crashing on patching/integrity checks detecting unpacker/decryption timing too early or too late/DEX re-encrypted between loads/native anti-analysis stacked protectors, decrypted payload analysis: capabilities/specific behaviors/C2/infrastructure/remaining obfuscation/second-stage payloads, lessons learned). Section 3: Packed Unpacker Black Hat 2020 (Maddie Stone) deep analysis — the chicken-and-egg problem of unpacking an anti-analysis native library that was itself packed; 4-layer methodology (static analysis of packed native library with Ghidra/IDA, dynamic capture of unpacked library in memory, analysis of unpacked anti-analysis code, bypass of anti-analysis checks); specific anti-analysis checks found (ptrace/TracerPid/isDebuggerConnected debugger detection, /proc/self/maps Frida scanning, process checks, emulator detection via hardware fingerprints/sensors/build props, root detection su/Magisk/debuggable flags, environment checks for filesystem artifacts/processes/packages); why it's notable as a meta-problem. Section 4: BRATA two-stage dropper walkthrough (static analysis of lightweight dropper: what it does/how it downloads/where from/permissions/environment checks; dynamic analysis: run and observe download/connections/installation; capturing downloaded payload; payload analysis; lessons: don't stop at dropper — real payload is downloaded). Section 5: Ghimob downloader payload analysis (Latin American banking trojan, downloader→payload pattern, trigger conditions, environment checks, payload capabilities). Section 6: Joker multi-stage infection chain (APK→hidden DEX→downloaded JAR; visible code looks harmless; loads hidden DEX via DexClassLoader and runs yin.Chao via reflection; that class downloads JAR from remote server; static discovery of hidden DEX in assets, DCL usage, reflection red flags; dynamic observation of DEX loading and remote download; capturing hidden DEX and downloaded JAR; analyzing each stage separately; lessons: multi-stage malware requires multi-stage analysis). Section 7: DexGuard walkthroughs — honest assessment that no canonical public full walkthrough exists for specific DexGuard-protected malware samples; per-sample analysis is the norm given active defender updates; hypothetical reconstruction of what a full DexGuard walkthrough would look like (9-step: sample ID with DexGuard version clues, initial static showing loader stub only, BlackDex first attempt, Frida hooks on class loading second, native loader hook third analyzing dexguard*.so with Ghidra/IDA, memory dumping fourth as fallback, static patching fifth, payload analysis with JADX/JEB + deobfuscation tools simplify/Katalina/TinySmaliEmulator, documentation; lessons: DexGuard is configurable not fixed, no universal unpacker exists, loader stub is key, anti-analysis is first hurdle, decrypted DEX may still be obfuscated). Section 8: Bangcle/Liuling Chinese RE community walkthroughs (Zhihu/CSDN extensive coverage; language barrier noted; techniques commonly described: hooking decryption/loading path, dumping decrypted DEX from memory, analyzing native loader stub, patching anti-analysis checks). Section 9: what typically fails — the failure-and-recovery narrative (anti-debug blocks Frida→stronger stealth/boot-time instrumentation/static loader analysis; anti-tamper crashes on patching→don't patch APK use dynamic instrumentation; integrity checks detect unpacker→run original APK intercept at runtime; decryption too early→attach earlier/spawn with process/hook loader constructor; decryption too late→capture in-memory at multiple points; DEX re-encrypted between loads→capture at every decryption event; native loader anti-analysis hardened→static Ghidra/IDA analysis/dump unpacked version/peel layers; multiple protectors stacked→peel one layer at a time). Section 10: tool sequence in real walkthroughs (14-step: apktool decode resources+Smali+manifest → JADX/JEB decompile visible → Androguard/MobSF programmatic analysis → APKiD identify protector → identify loader/decryptor custom Application class/native library → static analysis of loader Ghidra/IDA for native/Smali for Java → dynamic setup frida-server on device or gadget mode → Frida hooks on DexClassLoader/PathClassLoader/DexFile to capture decrypted DEX → if Frida blocked: boot-time instrumentation/stronger stealth/static loader work → BlackDex or similar automated DEX extraction → if DEX captured: JADX/JEB on decrypted DEX → if still obfuscated: simplify/Katalina/manual Smali → dynamic analysis of payload: Frida hooks on network/SMS/crypto → document full chain; pivot points table when to switch approaches). Section 11: where to find more walkthroughs (Black Hat/DEF CON/Recon talk materials; researcher blogs; vendor blogs NowSecure/GuardSquare/Oversecured/ReversingLabs/Lookout/Zimperium/Kaspersky Securelist/Check Point/Trend Micro; GitHub repos with unpacking scripts/writeups; Chinese RE community Zhihu/CSDN for Bangcle/Liuling; feedback loop — publishing walkthroughs propagates techniques). Section 12: bottom line. Companion to android-commercial-protector-bypass.md and android-re-frida-pipeline.md. |
| `android-malware-detection-research.md` | 689 | ~30KB | 6-phase on-device Android malware triage methodology: Phase 1 static analysis (MobSF, JADX, apktool, manifest deep-dive, YARA, APK signature analysis, resource inspection), Phase 2 dynamic analysis (frida-server, logcat, network capture, Frida trace/hook, Objection, sensitive data flows, DCL detection), Phase 3 automated tooling (MobSF batch, frida --codeshare, Drozer, AndroBugs, mobile-app-autoinjector, APIMonitor, AndroRAT detection scripts), Phase 4 memory forensics (LiME/ramdump concepts, Frida memory scanning, heap inspection via Objection, native library extraction — limited on-device without root, cloud/offline memory analysis for rooted/jailbreak), Phase 5 incident response (ADB/fastboot forensics, backup extraction, logcat/ Bugreport, app package analysis, APK preservation chain, MVT + stalkerware IOCs, device imaging), Phase 6 capability matrix (implant vs trojan vs stalkerware vs spyware — SMS/contacts/call recording/location/camera/microphone/screen/keystrokes/overlay/notification access/clipboard/keychain/keyboard, persistence mechanisms, false-negative risks, zero-click vs user interaction). **Section 5 added 2026-09-04: 6 notable recent families — ClayRat (self-propagating spyware, 700+ APKs, SMS+Accessibility), Arsink RAT (cloud-native, 1,216 APKs, 45K IPs, 143 countries, Firebase+Apps Script C2), PromptSpy (first AI-powered Android malware, Gemini persistence), Albiriox (MaaS banking RAT, live screen streaming, on-device fraud), Zimperium 4 campaigns (RecruitRat/SaferRat/Astrinox-Mirax-Medusa/Massiv, 800+ apps, multi-stage installation, Session Install API abuse), NFC Relay Fraud Evolution (NGate→Ghost Tap→WindRelay+SpyNote 13-min loan fraud chain).** Coverage gaps: mobile app memory forensics tooling gap, updated Android 14+ scoped storage, limited kernel-level visibility without root. References: 36 security tools/methods across categories. Bottom line: 3-phase triage (static+dynamic+automated) for most samples, escalate to memory/IR for advanced threats, understand capability matrix for impact assessment. |
| `android-exploit-identification.md` | 638 | ~32KB | 5-phase isolate-and-identify workflow for Android exploit assessment: Phase 1 triage (collect artifacts: build.prop fingerprint, dmesg, logcat, running processes, listening sockets, installed packages with signatures and patch levels, ADB/wireless debugging status, Shizuku status, unusual binaries), Phase 2 match (map build.prop fingerprint to Android version + security patch level; match patch level to Android Security Bulletins; identify components with known CVEs; use xairy/linux-kernel-exploitation index + pocindex + Exploit-DB + vendor advisories to find candidate CVEs; build candidate CVE list ranked by component match + exploit availability + relevance to observed behavior), Phase 3 collect (collect further evidence to narrow candidates: logcat filtered by component keywords, dmesg kernel crash/hang evidence, process/network/file inspection for compromise indicators, Shizuku/ADB/ Wireless Debugging status for post-exploit access paths, timeline reconstruction), Phase 4 confirm (narrow to most likely CVE; attempt verification in lab environment; reproduce if safe; collect forensic evidence; avoid destructive actions on evidence device), Phase 5 report (compile findings: device fingerprint, patch level, candidate CVEs with evidence strength, recommended actions: patch/update, mitigate, monitor, isolate, escalate; limitations and confidence levels). Reference card: 20+ Android kernel/app CVEs with type, component, impact, Android version range, patch status, exploit availability, notes (CVE-2019-2215 binder UAF, CVE-2023-20938 binder ioctl UAF, CVE-2023-2136 info leak, CVE-2022-20409 io_uring, CVE-2022-0847 Dirty Pipe, CVE-2023-0266 ALSA race, CVE-2023-26083 Mali info leak, CVE-2025-21479 Adreno GPU driver, CVE-2023-21255 binder, CVE-2023-40074 Kotlin compiler, CVE-2021-30461 root downgrade, CVE-2024-00448/29059/0044 Fugitive-in-Java chain, CVE-2025-48593/48596/48594/48602 Pixel 9 AWDL/Dolby chain, CVE-2025-2495 Chrome RIDL, CVE-2023-20938+2136 Android 13 patch gap, etc.). Bottom line: systematic workflow from triage→match→collect→confirm→report; build candidate CVE list from fingerprint+patch level; verify before concluding; healthy skepticism toward PoC claims. |
| `android-rooting-methods-deep-dive.md` | 418 | ~38KB | Comprehensive rooting methods reference: taxonomy (soft root / system root / bootloader-unlock root / kernel root / persistent vs one-shot), bootloader unlock + AVB 2.0 chained verification + rollback protection, Magisk architecture (MagiskSU, Zygisk, DenyList, MagiskBoot, modules), KernelSU and APatch kernel-level root alternatives, exploit-based rooting historical context (CVE-2019-2215, CVE-2020-0041, CVE-2021-1908, CVE-2022-20409, Dirty Pipe) and why it still matters for research, root detection evolution (SafetyNet Attestation/CTSProfile → Play Integrity API MEET_DEVICE_INTEGRITY/MEET_STRONG_INTEGRITY/MEET_DEVICE_AND_APP_STANDARDS, software vs hardware-backed, TEE attestation), root hiding and cat-and-mouse (userland hiding, boot image patching, KernelSU/APatch hiding, Zygisk-based hiding, "never root" alternatives), what root unlocks for research (reading /data/data of other apps, memory dumping, persistent instrumentation, kernel modules, full filesystem, frida-server without repackaging, root-mode Shizuku), risks and limitations (OTA breakage, Knox trip, AVB/dm-verity taint, bootloops, Play Integrity flags, security model weakening), practical recommendations (dedicated test devices, emulator roots, Magisk+Zygisk, document root state), future direction (hardware-backed attestation hardening, kernel-level trend). **Section 10 added 2026-09-04: Legacy sideload rooting methods — Quip (device deployment sideload tool, historically used for rooting+set up; historical context + modern alternatives), Priv/Ease (Android 2.2 Era app from AppApex, installs .apk as new on every device boot, bypassing AVB and enabling superuser on locked devices with pry4 pull; historical root method + persistence mechanism). Note on KingRoot: accordions remain. Flat "should NOT be used" directive removed — subsections enumerate why it is NOT CURRENTLY VETTED (no page fetch, KumbleLink reference ungrounded).** |
| `android-17-18-security-direction.md` | 243 | ~22KB | Android 17 & 18 early security direction and known/expected security changes. Android 17 direction: continued privacy/storage/permission tightening, sandboxing and isolation improvements (based on 14/15/16 trajectory), per-app language preferences, predictive back, HEIF/USB3 camera support, refined intent filters — grounded where possible in developer.android.com behavior-changes page + trend extrapolation, with explicit extrapolation markers. Android 18 (cut) direction: all feature items framed as "disclosed/captured in coverage" (9to5Google, Ars Technica, Android Police, PhoneArena-style outlets) — battery/charging visibility, hourly weather, freeze-ended texting glanceable info, back gestures handled by launcher, auto-verification of app purchases, surrounding devices access API, 16:10+ display support, battery saver + per-app data modes refinements, threads/conversation ranking evolution. Security impact analysis: continuing trends from 14→15→16 (scoped storage, permission granularity, security-by-default, hidden API shrinkage, memory safety); how 17/18 likely extend these; what this means for Red Team/malware analysis/pentesting (what keeps getting harder, what stays the same, adaptation recommendations). Cross-references to source material: developer.android.com Android 17 / Android 18 (cut) feature pages; named external sources for 18 cut details. Sources cited per section with honest uncertainty markers. Notability note: this is a coverage/compilation file — the 18 items reflect what third-party outlets have reported about the cut, not an official spec. Do not treat individual 18 features as confirmed. Compared to a "darknet browsing" task, these are surface-web feature reports compiled for completeness. |
| `android-malware-detection-tools-survey.md` | 309 | ~23KB | Tool survey for on-device Android malware detection: MobSF, Androguard, APKLeaks, MARA, Androwarn, MVT, Pithus, Androl4b — capabilities, usage modes, strengths/limits for triage work. |
| `android-exploit-chains-sources.md` | 49 | ~6KB | Source URLs backing android-exploit-chains.md (Project Zero, Citizen Lab, Amnesty, Oversecured, Google OffSec blog, Cyfirma, GitHub Fugitive-in-Java). |
| `android-malware-c2-infra-analysis-sources.md` | 26 | ~3KB | Source URLs backing android-malware-c2-infra-analysis.md. |

These files were produced by the 2026-09-04 delegation wave that targeted the four biggest capability gaps flagged during skill consolidation, plus two additional hard gaps (root methods, Android 17/18 direction) and a malware attribution/campaign tracking deep dive. Related on-disk files: android-malware-c2-infra-analysis-sources.md (26 lines, source URLs backing the C2 analysis), android-exploit-chains-sources.md (49 lines, source URLs backing exploit-chains.md), android-malware-detection-tools-survey.md (309 lines, tool survey). Fresh-lead malware/security/tools integration added 2026-09-04: ClayRat, WindRelay, PromptSpy, Arsink, Albiriox, Zimperium-4 campaigns in attribution/detection files; CVE-2025-48595 in exploit-identification.md; Android 16+ Intrusion Logging in 14-15-16-security-changes.md. They are standalone references; the skill above remains the entry point.

---

## 12. Darknet Browsing — Reality Check

**Can Hermes browse the darknet?** No. The browser tools and web extraction tools operate on the surface web (clearnet). Tor hidden services (.onion), I2P, Freenet, and other darknet networks are not accessible through these tools. There is no Tor client or Tor proxy configured for the browser or web tools.

**What you can do instead:**
- Gather surface-web intelligence about darknet Android malware distribution: forum posts (archived), threat intel reports, security vendor blogs that discuss underground distribution channels.
- Monitor leak sites and forums that are accessible on the surface web (some forums have clearnet mirrors or are indexed).
- Use threat intel feeds (AlienVault OTX, Abuse.ch, VirusTotal, MalwareBazaar) that collect samples and IOCs regardless of origin.
- For actual darknet access, you would need a local Tor browser/session — that's a hardware/OS-level capability, not something Hermes can do remotely.

**Practical alternative:** The IOC sources listed in Section 9 (stalkerware indicators, MVT indicators, MalwareBazaar, AndroZoo, VirusTotal) give you malware samples and indicators from all sources — surface web, darknet, and otherwise — without needing to access the darknet directly. For attribution and campaign tracking, threat intel reports from Kaspersky, Zimperium, Group-IB, Citizen Lab, Amnesty International, and similar organizations often reference darknet distribution channels without requiring you to access them.

---

## 13. Learning Path — From Beginner to Wizard

### Level 1 — ADB basics
- Enable USB debugging, connect via ADB, run `adb shell`, `adb logcat`, `adb pull`, `adb install/uninstall`, `adb bugreport`.
- Understand UID 2000 shell vs root. Know what you can and can't do as shell.

### Level 2 — Shizuku & privileged access
- Install Shizuku, start it via ADB or wireless debugging, verify it's running.
- Use Shizuku-enabled apps for package management, debloating, log viewing.
- Understand UserService, `Shizuku.getUid()`, shell vs root tiers.
- Read [HackTricks Shizuku page](https://hacktricks.wiki/en/mobile-pentesting/android-app-pentesting/shizuku-privileged-api.html).

### Level 3 — Static analysis
- Run MobSF on an APK, read the report. Understand permissions, exported components, API calls, YARA hits.
- Use JADX to decompile and read Java. Use apktool to decode and inspect manifest/Smali.
- Write basic YARA rules for Android patterns (component names, API usage, strings).

### Level 4 — Dynamic instrumentation
- Set up frida-server on a rooted device (or gadget mode on non-root).
- Use `frida-ps`, `frida-trace`, `frida --codeshare`.
- Use Objection for SSL pinning bypass, root detection bypass, exploration.
- Write basic Frida hooks: log a method call, modify a return value, bypass a check.
- Hook DexClassLoader for DCL detection.

### Level 5 — Integrated analysis
- Run the full pipeline from Section 2: MobSF → apktool → JADX → Frida/Objection → on-device triage → report.
- Use drozer to enumerate IPC attack surface.
- Use MVT + stalkerware IOCs for device forensics.
- Correlate static findings with dynamic behavior.

### Level 6 — Exploit identification
- Collect artifacts from a suspicious device (Section 4, Phase 2).
- Match fingerprint/patch level to Android Security Bulletins.
- Identify CVE candidates from component hints in logcat.
- Correlate with PoCs from pocindex/Exploit-DB/GitHub.
- Understand drozer (app-layer) vs kernel exploit (Pillar 4) vs Ghost (post-exploit).

### Level 7 — Advanced techniques
- Advanced Frida: native hooking, anti-Frida/anti-root bypass, Flutter/React Native instrumentation, persistent instrumentation.
- Deobfuscation: identify obfuscators, use deobfuscators, manual Smali work, native code RE.
- Kernel exploit lab: reproduce CVE-2019-2215 in QEMU (see android-kernel-exploit-lab-setup.md).
- Shizuku/Dhizuku exploitation: abuse Shizuku-enabled apps, UserService exploitation.
- Vulnerability discovery: systematic testing methodology, bug bounty programs, OWASP MASTG.

### Level 8 — Threat intel & attribution
- Track current malware families (Section 9).
- Use IOC feeds for detection and attribution.
- Understand campaign structures, C2 infrastructure, financial fraud patterns.
- Read Project Zero, Google TAG, Citizen Lab, Amnesty International reports for in-the-wild exploit analysis.

---

## 14. Sources & Further Reading

### Core repos (on disk or bookmarked)
- [RikkaApps/Shizuku](https://github.com/RikkaApps/Shizuku) — Shizuku core (29.7k stars)
- [RikkaApps/Shizuku-API](https://github.com/RikkaApps/Shizuku-API) — developer library
- [timschneeb/awesome-shizuku](https://github.com/timschneeb/awesome-shizuku) — Shizuku app catalog (10k stars)
- [MobSF/Mobile-Security-Framework-MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF) — all-in-one analysis (21.7k stars)
- [androguard/androguard](https://github.com/androguard/androguard) — APK/DEX toolkit
- [skylot/jadx](https://github.com/skylot/jadx) — Dex→Java decompiler (50.3k stars)
- [frida/frida](https://github.com/frida/frida) — dynamic instrumentation (21.8k stars)
- [sensepost/objection](https://github.com/sensepost/objection) — mobile exploration (9.4k stars)
- [httptoolkit/frida-interception-and-unpinning](https://github.com/httptoolkit/frida-interception-and-unpinning) — MITM + unpinning (2.3k stars)
- [appsec.fyi/mobile](https://appsec.fyi/mobile.html) — curated mobile security resources (updated Jun 2026): Frida, Objection, MobSF, JADX, apktool, Hopper, Ghidra, proxy tools
- [ReversecLabs/drozer](https://github.com/ReversecLabs/drozer) — IPC assessment (4.6k stars)
- [EntySec/Ghost](https://github.com/EntySec/Ghost) — ADB post-exploitation (3.4k stars)
- [xairy/linux-kernel-exploitation](https://github.com/xairy/linux-kernel-exploitation) — kernel exploit index (6.6k stars)
- [0xbinder/android-kernel-exploitation-lab](https://github.com/0xbinder/android-kernel-exploitation-lab) — CVE-2019-2215 lab (43 stars)
- [sfewer-r7/pocindex](https://github.com/sfewer-r7/pocindex) — 82k PoC aggregation
- [ashishb/android-security-awesome](https://github.com/ashishb/android-security-awesome) — master index (9.7k stars)
- [user1342/Awesome-Android-Reverse-Engineering](https://github.com/user1342/Awesome-Android-Reverse-Engineering) — RE list (2.7k stars) — ANDROID-SPECIFIC. likely stale by now — refresh before relying on it.
- [anpa1200/Android-Malware-Analysis](https://github.com/anpa1200/Android-Malware-Analysis) — triage pipeline (YARA + ATT&CK + Frida + LLM)
- [Krypteria/Yaralyze](https://github.com/Krypteria/Yaralyze) — YARA + hash Android malware detection
- [mvt-project/mvt](https://github.com/mvt-project/mvt) — IOC-based device forensics (Amnesty International)
- [AssoEchap/stalkerware-indicators](https://github.com/AssoEchap/stalkerware-indicators) — stalkerware IOCs

### Methodology & writeups (on disk or bookmarked)
- [HackTricks Shizuku page](https://hacktricks.wiki/en/mobile-pentesting/android-app-pentesting/shizuku-privileged-api.html) — Shizuku security model, UserService, threat implications
- [ReverseLabs DCL blog](https://reverselabs.dev/blog/dynamic-code-loading-android) — DCL detection, BRATA/Ghimob/Joker case studies, Frida hook
- [Talsec Frida article](https://docs.talsec.app/appsec-articles/articles/hook-hack-defend-fridas-impact-on-mobile-security-and-how-to-fight-back) — Frida detection, RASP, bypass techniques
- [0xbinder lab docs](../../../android-kernel-exploit-lab-setup.md) — full QEMU + kernel + GDB walkthrough on disk
- [Android kernel exploit identification](../../../android-exploit-identification.md) — 5-phase isolate-and-identify workflow on disk
- [Android malware detection research](../../../android-malware-detection-research.md) — 6-phase on-device triage + tool survey on disk
- [Shizuku/ADB workflows](../../../shizuku-adb-workflows.md) — 30 commands + privilege tier table on disk
- [ADB/Shizuku primer](../../../adb-shizuku-primer.md) — fundamentals + wireless debugging + privilege model on disk
- [Safe encrypted storage](../../../android-safe-encrypted-storage.md) — Keystore-backed AEAD, envelope formats, backup/restore, and migration guidance on disk
- [httptoolkit blog](https://httptoolkit.com/blog/frida-mobile-interception-funding/) — Frida interception funding, EU NGI Zero, MITM methodology
- [Frida CodeShare universal bypass](https://codeshare.frida.re/@ssecurityy/universal-robust-advanced-root--ssl-pinning-bypass/) — root + SSL pinning + debugger + root hide + emulator + obfuscated method hooks
- [Kaspersky Q2 2026 mobile report](https://securelist.com/malware-report-q2-2026-mobile-statistics/120948/) — current threat landscape
- [Malwarebytes WindRelay article](https://www.malwarebytes.com/blog/mobile/2026/08/new-android-malware-lets-criminals-use-your-bank-card-in-real-time) — SpyNote + WindRelay NFC relay combo
- [Android OffSec blog](https://androidoffsec.withgoogle.com/) — Google's Binder/kernel exploit writeups
- [Project Zero blog](https://googleprojectzero.blogspot.com/) — in-the-wild exploit analyses
- [OWASP MASTG](https://owasp.org/www-project-mobile-app-security/) — comprehensive mobile testing guide

### External threat intel
- [Kaspersky Securelist mobile reports](https://securelist.com/tag/mobile-malware/)
- [Zimperium Banking Heist Report](https://zimperium.com/resources/new-zimperium-report-finds-banking-malware-expands-global-reach-targeting-1-200-financial-apps)
- [MalwareBazaar](https://bazaar.abuse.ch/)
- [AndroZoo](https://androzoo.uni.lu/)
- [VirusTotal](https://www.virustotal.com/)
- [Google Android Security Bulletins](https://source.android.com/docs/security/bulletin/)
- [Google Android Security Reward Program](https://www.google.com/about/appsecurity/android-rewards/)
- [Google TAG blog](https://blog.google/threat-analysis-group/)
- [Citizen Lab](https://citizenlab.ca/)
- [Amnesty International Security Lab](https://www.amnesty.org/en/specialized-programmes/security-lab/)
- [Intel 471](https://www.intel471.com/) — FvncBot, emerging threat reports

---

## 15. Operational Notes

- **Always have authorization.** Only test devices you own or have explicit written permission to test. The tools in this skill can be used for malicious purposes — the skill assumes legitimate security research, forensics, or authorized testing.
- **Preserve evidence.** When investigating a compromised device, follow the isolate-and-identify workflow: isolate first, then collect artifacts, then analyze. Don't power off, don't factory reset, don't run PoCs on the compromised device.
- **Respect SELinux.** SELinux Permissive/Disabled is a significant finding — it may indicate prior compromise or OEM configuration. Note it in your report.
- **ADB over network is risky.** If `service.adb.tcp.port` is set, adbd is listening on the network. Disable it (`setprop service.adb.tcp.port -1`) unless you need it. On production devices, keep USB debugging disabled.
- **Shizuku persistence:** Shizuku sessions can drop. Use Shizuku-Keeper (Automate flow) or similar to keep it alive during long sessions. On Android 14+, the libshizuku.so path changed — scripts using the old `start.sh` path may need updating.
- **Frida server detection:** Many apps detect frida-server. Use gadget mode when possible, or combine detection bypass scripts. Talsec freeRASP is a strong RASP — if the app uses it, generic Frida may not work.
- **Repackaging for gadget mode:** Requires resigning the APK. The original signature is broken — the app may check its own signature and refuse to run. Test signature check bypass if needed.

---

*Corpus on disk: 21 structured companion files, SKILL.md, and the archive index. Safe encrypted storage was added 2026-09-05 to close the final planned coverage gap. Counts are maintained in the repository README and archive index; refresh them after future edits. Next wave topics: see archive index "Next wave topics" section or flag new gaps.*
