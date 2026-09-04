# ADB & Shizuku for Android Security Testing — Primer

**Target audience:** Anyone with basic Android knowledge who wants to start interrogating apps and doing device forensics.

**Scope:** ADB fundamentals, Shizuku's privilege model, wireless debugging setup, practical commands, accessing protected APIs, limitations, and curated learning resources.

---

## 1. What ADB Is

**ADB (Android Debug Bridge)** is a client-server CLI tool included in the Android SDK Platform Tools. It lets a host computer communicate with an Android device over USB or Wi-Fi to install/uninstall apps, run shell commands, read logs, copy files, and inspect system state.

Three components make it work:

| Component | Where it runs | Role |
|-----------|---------------|------|
| **Client** | Your computer | The `adb` binary you invoke from the terminal |
| **Server** | Your computer | Background process that manages connections (TCP port 5037) |
| **adbd** (daemon) | The device | Listens for commands; runs as the `shell` UID (2000) when USB debugging is enabled |

ADB gives you a Unix shell on the device with the privileges of the `shell` user — more powerful than a normal app, but **not root**. Some commands (e.g., reading another app's `/data/data/` directory) are still blocked.

**Where to get it:** Google distributes the standalone [SDK Platform Tools](https://developer.android.com/studio/releases/platform-tools). On Linux you can also pull it via `sdkmanager` or your distro's package manager.

---

## 2. What Shizuku Is

**Shizuku** is an Android app (package `moe.shizuku.privileged.api`, originally by RikkaApps, now also maintained as forks like Shizuku+ by thejaustin) that lets other apps call Android system APIs with **ADB (shell UID 2000) or root (UID 0) privileges** — without requiring those apps to be installed as system apps or to have root themselves.

### How it works (simplified)

1. **Shizuku server** is a Java process started via `app_process` that runs with either shell (ADB) or root privileges.
2. It exposes a **Binder service** (`moe.shizuku.privileged.api`) that acts as a proxy: a client app sends a Binder transaction → Shizuku server forwards it to the real system service (e.g., `PackageManagerService`) → result comes back.
3. Client apps use the **Shizuku-API** library to wrap system-service Binder interfaces with `ShizukuBinderWrapper`, so calls are executed with elevated UID without the app needing special OS privileges.
4. Shizuku also offers a **UserService** API: you can run your own Java/JNI code in a separate process as UID 2000 (shell) or UID 0 (root), which is useful for long-lived tooling or complex operations that don't fit a single Binder call.

**Key distinction:** Shizuku backed by ADB gives you **shell UID 2000** — the same as `adb shell`. Root mode (via Magisk module "Sui" or `su`) gives you **UID 0** with full filesystem and kernel access. They are **not** the same.

**Relevant repos:**
- Shizuku core: [`RikkaApps/Shizuku`](https://github.com/RikkaApps/Shizuku) (29.7k stars)
- Shizuku API & developer guide: [`RikkaApps/Shizuku-API`](https://github.com/RikkaApps/Shizuku-API)
- Shizuku+ (enhanced fork): [`thejaustin/ShizukuPlus`](https://github.com/thejaustin/ShizukuPlus)
- Sui (Magisk module for root-mode Shizuku): [`RikkaApps/Sui`](https://github.com/RikkaApps/Sui)

---

## 3. Wireless Debugging & Device Pairing

### Android 11+ (native Wireless Debugging)

Android 11 (API 30) introduced built-in Wireless Debugging with pairing codes and TLS encryption. This is the method Shizuku uses when starting without a computer.

**Setup (one-time pairing):**

1. **Enable Developer Options** — tap "Build number" 7 times in Settings > About phone.
2. **Enable USB Debugging** in Developer Options.
3. **Enable Wireless Debugging** in Developer Options.
4. In Wireless Debugging, tap **"Pair device with pairing code"** — note the IP address, port, and 6-digit pairing code displayed.
5. On your computer: `adb pair <IP>:<port>` then enter the pairing code when prompted.
6. After pairing, you can connect wirelessly: `adb connect <IP>:<port>` (the IP:port shown *after* closing the pairing dialog, not the pairing one).

Once paired, the device remembers your computer. On trusted networks, ADB can auto-connect. Android 17 / ADB 37.0.0 added "adb Wi-Fi 2.0" with automatic reconnection on trusted networks.

**For Shizuku specifically:** Shizuku's own UI handles the pairing flow internally — you tap "Start via Wireless debugging," it prompts you to enable Wireless Debugging and enter the pairing code within the app. No computer needed after initial setup.

### Android 10 and below (legacy TCP/IP mode)

This method requires an initial USB connection:

```bash
# With device connected via USB and USB debugging enabled
adb tcpip 5555          # Puts adbd into TCP mode on port 5555
adb disconnect          # Disconnect USB cable
adb connect <device-ip>:5555
```

No pairing code, no encryption (plain-text traffic). Less secure; a device on the same network can attempt to connect and get a yes/no dialog. You can only turn this off by rebooting.

**Caveats:**
- Android 11+ Wireless Debugging is preferred — it encrypts traffic and requires explicit pairing.
- Some OEMs (MIUI/HyperOS, ColorOS, EMUI) add extra security layers that can break Shizuku; see OEM-specific tweaks in the Shizuku FAQ.
- Wireless debugging requires the device and computer to be on the same network (or reachable via routed network).

**References:**
- [Android docs: Connect to a device over Wi-Fi](https://developer.android.com/tools/adb#connect-to-a-device-over-wi-fi)
- [LineageOS Wiki: ADB over Wi-Fi](https://wiki.lineageos.org/how-to/adb-over-wifi/)

---

## 4. Shizuku's Privilege Model: Shell UID vs Root

### The two modes at a glance

| | ADB/Shell mode | Root mode |
|---|---|---|
| **UID** | 2000 (shell) | 0 (root) |
| **How started** | `adb shell sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh` (or wireless equivalent) | `su -c sh /data/adb/shizuku/start.sh` (or Sui Magisk module) |
| **Access level** | Same as `adb shell`; can call most system APIs visible to shell | Full root; can read/write almost anything, mount filesystems, etc. |
| **Can read `/data/data/<pkg>`?** | No — shell cannot access another app's private directory | Yes (subject to SELinux, which may still block some paths) |
| **Can use hidden APIs freely?** | No — hidden API restrictions apply; use AndroidHiddenApiBypass or move logic to UserService | Better, but still constrained by Android version and SELinux |
| **Persistence** | Must be restarted after every reboot (on non-rooted devices) | Can be set to start on boot with Magisk |

### What shell (UID 2000) can and cannot do

Shell permissions come from the [`Shell` app's AndroidManifest.xml](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/packages/Shell/AndroidManifest.xml) in AOSP. They **change across Android versions** and OEMs can trim them further.

**Generally accessible:**
- `pm` commands (list packages, grant/revoke permissions, install/uninstall)
- `dumpsys` (query system service state)
- `appops` (query/modify app operations)
- `settings` (read/write system settings — within shell's allowed namespace)
- `cmd` (activity manager, package manager, connectivity, etc.)
- `logcat` (reading logs)
- `am` (start activities, broadcast intents, start services)
- `screencap`, `screenrecord`
- `ps`, `top`, `pidof`

**Not accessible from shell:**
- Reading another app's private data directory (`/data/user/0/<pkg>/`)
- Modifying secure/system settings that require `WRITE_SECURE_SETTINGS` (though ADB shell does get some of these via its manifest; Shizuku also acquires `WRITE_SECURE_SETTINGS` via its own setup)
- Accessing certain hardware-level APIs
- Bypassing hidden API restrictions enforced at the framework level

**The `WRITE_SECURE_SETTINGS` permission:** Shizuku requests this permission during setup. It allows modifying highly privileged settings (including toggling USB/Wireless debugging). It is considered sensitive — granted only to system apps by default, but ADB/shell can be granted it through the normal permission mechanism when authorized by the user.

**Check which mode you're in:**
```bash
# Inside Shizuku's settings or via API: Shizuku.getUid() returns 2000 (shell) or 0 (root)
adb shell dumpsys activity service moe.shizuku.privileged.api | head
adb shell service list | grep shizuku
```

---

## 5. Practical Commands for Security Testing & Forensics

### Listing apps

```bash
# All packages
adb shell pm list packages

# With APK path
adb shell pm list packages -f

# Third-party only
adb shell pm list packages -3

# System packages only
adb shell pm list packages -s

# Enabled/disabled
adb shell pm list packages -e    # enabled
adb shell pm list packages -d    # disabled

# List associated users
adb shell pm list users
```

### Inspecting a specific app's metadata

```bash
# Dump full package info (very verbose)
adb shell dumpsys package <package.name>

# Grep for specific fields
adb shell dumpsys package <package.name> | grep -E 'userId|permission|granted|enabled'

# Path to APK
adb shell pm path <package.name>

# App uid
adb shell dumpsys package <package.name> | grep userId
```

### Permissions

```bash
# List all known permissions
adb shell pm list permissions

# List permissions with details
adb shell pm list permissions -g

# Permissions granted to a specific app
adb shell dumpsys package <package.name> | grep -A 20 'granted=true'

# Grant a permission
adb shell pm grant <package.name> <permission.name>

# Revoke a permission
adb shell pm revoke <package.name> <permission.name>

# Clear all app data (resets app to fresh-install state)
adb shell pm clear <package.name>
```

### App operations (AppOps)

```bash
# List app ops for a package
adb shell appops list <package.name>

# Set an app op
adb shell appops set <package.name> <op> <mode>
# modes: allow, ignore, deny, grant, revoke
```

### Reading logs

```bash
# Live logcat
adb logcat

# Filter by tag or package
adb logcat -s <Tag>
adb logcat | grep <package.name>

# Clear logs
adb logcat -c

# Save to file
adb logcat -d > /tmp/logcat.txt

# Full bug report (dumpsys + logcat + dumpstate)
adb bugreport > /tmp/bugreport.zip
```

### App data (where accessible)

```bash
# Note: shell UID cannot read /data/data/<pkg> on modern Android.
# You need root (adb root on emulator/userdebug builds, or full root via Magisk)
# or Shizuku in root mode.

# With root access:
adb shell su -c "ls -la /data/data/<package.name>/"
adb shell su -c "cat /data/data/<package.name>/shared_prefs/*.xml"

# SQLite databases (requires root for other apps' data)
adb shell sqlite3 /data/data/<package.name>/databases/<db_name>.db ".dump"

# Pull APK from device
adb pull $(adb shell pm path <package.name> | cut -d ':' -f 2) ./app.apk

# Pull app data (requires root; use adb backup as fallback on older Android)
adb backup -f backup.ab -apk <package.name>
```

**Note on backups:** `adb backup` is deprecated on Android 12+ and increasingly restricted. On modern devices, root or specialized forensic tools (e.g., SQLite 직접 접근 via root) are the realistic path.

### System state inspection (dumpsys)

```bash
# Activity manager state
adb shell dumpsys activity

# Current foreground activity
adb shell dumpsys activity activities | grep -E 'mCurrentFocus|mFocusedApp'

# Running services
adb shell dumpsys activity services

# Package manager state (all packages)
adb shell dumpsys package packages

# Memory info for a process
adb shell dumpsys meminfo <package.name>

# Battery stats
adb shell dumpsys batterystats --charged <package.name>

# Connectivity / network
adb shell dumpsys connectivity

# Window / display state
adb shell dumpsys window

# Sensor state
adb shell dumpsys sensorservice

# CPU, procstats, power, etc.
adb shell dumpsys cpuinfo
adb shell dumpsys procstats
adb shell dumpsys power
```

### Launching, binding, and broadcasting

```bash
# Start an activity
adb shell am start -a android.intent.action.VIEW -d https://example.com

# Start a service
adb shell am startservice ...

# Send a broadcast
adb shell am broadcast -a <action> [--es key value ...]

# Force stop an app
adb shell am force-stop <package.name>

# Simulate incoming call
adb shell am start-activity -a android.intent.action.CALL -d tel:+1234567890
```

### Screen capture / recording

```bash
adb shell screencap -p /sdcard/screenshot.png
adb pull /sdcard/screenshot.png

adb shell screenrecord /sdcard/rec.mp4   # stop with Ctrl+C; max ~3 minutes
adb pull /sdcard/rec.mp4
```

---

## 6. Using Shizuku to Access Protected APIs

The real power of Shizuku for security testing is letting **apps** (or scripts) call system APIs with shell/root identity without requiring the app itself to be a system app or have root.

### The basic flow (for a Shizuku-enabled app)

1. App declares `<uses-permission android:name="moe.shizuku.manager.permission.API"/>` (or `rikka.shizuku.manager.permission.API` for v11+).
2. App includes a `ShizukuProvider` in its manifest.
3. App waits for the Shizuku Binder to be available, then requests permission from the user (runtime-style dialog).
4. Once granted, the app wraps the target system service Binder with `ShizukuBinderWrapper` and makes calls as if it had shell/root UID.

**Minimal example** (from HackTricks / Shizuku-API docs):

```java
Shizuku.addBinderReceivedListenerSticky(() -> {
    if (Shizuku.checkSelfPermission() != PackageManager.PERMISSION_GRANTED) {
        Shizuku.requestPermission(1000);
        return;
    }
    IPackageManager pm = IPackageManager.Stub.asInterface(
        new ShizukuBinderWrapper(SystemServiceHelper.getSystemService("package"))
    );
    // Now use pm as if you were the system — enumerate packages, grant/revoke, etc.
});
```

### UserService: running your own code with elevated UID

For operations beyond simple Binder transactions (stateful tooling, JNI helpers, repeated operations), Shizuku offers **UserService**. You define a service component; Shizuku starts it in a separate process as UID 2000 (shell) or UID 0 (root):

```java
Shizuku.UserServiceArgs args = new Shizuku.UserServiceArgs(
    new ComponentName(this, AuditService.class))
    .daemon(false)
    .version(1)
    .processNameSuffix("audit");

Shizuku.bindUserService(args, conn);
```

**Use cases relevant to security testing:**
- Enumerate all installed apps with hidden API access
- Modify app permissions en masse (debloating)
- Collect logs, network state, running processes programmatically
- On-rooted devices: dump app databases, inspect SELinux contexts, read protected files
- Run `rish` — a remote shell that gives you a privileged command line inside Termux or another terminal app

### `rish`: privileged shell from a terminal app

Shizuku's settings include **"Use Shizuku in terminal apps"**, which installs `rish` — a helper that lets terminal apps (like Termux) execute commands through Shizuku's shell/root session. This effectively gives you a `shizuku-shell` prompt inside Termux without needing external ADB.

```bash
# Inside Termux (after enabling rish via Shizuku settings)
pkg install wget
wget https://rikka.app/rish/latest -O rish && chmod +x rish
# Then use rish to run commands with Shizuku's privileges
```

**Relevant documentation:**
- [Shizuku-API developer guide](https://github.com/RikkaApps/Shizuku-API) — the canonical source for integrating with Shizuku
- [HackTricks: Shizuku Privileged API](https://hacktricks.wiki/en/mobile-pentesting/android-app-pentesting/shizuku-privileged-api.html) — security-focused overview with threat-model notes

---

## 7. Known Limitations & Caveats

### ADB shell limitations
- **Not root.** Shell UID 2000 cannot read other apps' private data, cannot modify all secure settings, cannot mount filesystems.
- **Permissions vary by Android version.** What shell can do on Android 10 may differ from Android 14. OEMs can further restrict shell.
- **`adb root` only works on userdebug/eng builds** (emulators and engineering devices). Production ROMs (user builds) do not allow `adb root`.
- **Hidden API restrictions** (Android 9+) affect apps using non-SDK interfaces. Moving code to a UserService helps but doesn't fully bypass framework-level enforcement; for heavy hidden-API work, dedicated bypass modules (e.g., [AndroidHiddenApiBypass](https://github.com/LSPosed/AndroidHiddenApiBypass)) are recommended.

### Shizuku-specific
- **Must be restarted after every reboot** on non-rooted devices (ADB-backed mode). Root mode with Sui/Magisk can persist.
- **OEM quirks:** MIUI/HyperOS requires "USB debugging (Security options)" in addition to normal USB debugging. ColorOS/OnePlus may need "Permission monitoring" disabled. Flyme (Meizu) has "Flyme payment protection" to disable. Check the [Shizuku FAQ](https://shizuku.rikka.app/guide/setup/#faq) for device-specific workarounds.
- **Background killing:** Some OEM ROMs aggressively kill background apps, which can stop Shizuku. Allow Shizuku to run in the background.
- **ADB authorization timeout:** On Android 11+, enable "Disable adb authorization timeout" to prevent random disconnections during long test sessions.
- **`WRITE_SECURE_SETTINGS`:** Shizuku requests and holds this permission to manage USB/Wireless debugging toggles. This is powerful — treat the Shizuku app with trust.
- **Security surface:** Shizuku keeps USB debugging enabled while running and briefly toggles Wireless Debugging on/off. Traffic on loopback (127.0.0.1) is plain text (for legacy TCP mode); Wireless Debugging uses TLS. There's a brief window during setup where Wireless Debugging is enabled before Shizuku disables it. Understand the trade-off: more convenience, larger attack surface.
- **You cannot access `/data/data/<pkg>` from shell-backed Shizuku.** For that you need root mode.
- **Shizuku is not a root substitute.** It provides *some* of root's capabilities through elevated shell identity but is intentionally limited by design.

### Forensic limitations
- App data is encrypted at rest (Android's File-Based Encryption) on modern devices; without the lockscreen credential or root/backup access, some data is inaccessible.
- `adb backup` is deprecated and increasingly restricted; it won't capture everything.
- Some apps use the Android Keystore for cryptographic keys that are hardware-backed and non-extractable — you won't get those keys out via ADB/Shizuku.
- SELinux enforcement can block even root from accessing certain paths at runtime.

---

## 8. Learning Resources (curated)

### Documentation
- **[Android Debug Bridge (adb) — official docs](https://developer.android.com/tools/adb)** — authoritative reference for all ADB commands, wireless debugging setup, and shell commands.
- **[dumpsys — official docs](https://developer.android.com/tools/dumpsys)** — how to use dumpsys for system inspection.
- **[Shizuku user manual](https://shizuku.rikka.app/guide/setup/)** — setup, start methods, FAQ, OEM-specific troubleshooting.
- **[Shizuku-API developer guide](https://github.com/RikkaApps/Shizuku-API)** — integrating with Shizuku as an app developer; also useful for understanding the privilege model, Binder wrapping, and UserService.

### GitHub repos
- [`RikkaApps/Shizuku`](https://github.com/RikkaApps/Shizuku) — the core Shizuku app (29.7k stars). Read the README for the architecture explanation.
- [`RikkaApps/Shizuku-API`](https://github.com/RikkaApps/Shizuku-API) — the API library and demo; essential for understanding how client apps talk to Shizuku.
- [`thejaustin/ShizukuPlus`](https://github.com/thejaustin/ShizukuPlus) — enhanced fork with QoL improvements and backported features.
- [`timschneeb/awesome-shizuku`](https://github.com/timschneeb/awesome-shizuku) — curated list of 100+ apps that use Shizuku; great for finding tools in categories like privacy, debloating, file management, network auditing, and terminals. See especially the "Command-line utilities" and "Terminal" sections for security-relevant tools.
- [`timschneeb/changelog-awesome-shizuku`](https://github.com/timschneeb/changelog-awesome-shizuku) — daily changelog for the awesome list.
- [`LSPosed/AndroidHiddenApiBypass`](https://github.com/LSPosed/AndroidHiddenApiBypass) — for when you need to go beyond Shizuku's hidden-API limits.

### Tutorial / security-focused writeups
- **[HackTricks: Shizuku Privileged API](https://hacktricks.wiki/en/mobile-pentesting/android-app-pentesting/shizuku-privileged-api.html)** — the best single security-focused overview: privilege model, setup, API usage, threat-model implications, defensive notes.
- **[Kitsumed Blog: What Is Shizuku?](https://kitsumed.github.io/blog/posts/what-is-shizuku_how-does-it-work_security-implications/)** — detailed walkthrough of Shizuku's internal behavior, the Wireless Debugging toggle dance, WRITE_SECURE_SETTINGS implications, TCP mode vs Wireless Debugging, and security critique.
- **[TalSec: Shizuku's Perils and Root Detection](https://docs.talsec.app/appsec-articles/articles/how-to-achieve-root-like-control-without-rooting-shizukus-perils-and-talsecs-root-detection)** — security perspective on how Shizuku can be used by malicious apps and how developers can detect it.
- **[HighOn.coffee: ADB Commands Cheat Sheet](https://highon.coffee/blog/adb-command-cheat-sheet/)** — a solid pen-test-oriented ADB command reference.
- **[Android StackExchange: How does Shizuku work?](https://android.stackexchange.com/questions/258214/how-does-shizuku-work)** — community Q&A clarifying what Shizuku can and cannot do vs root.

### ADB command references
- **[Pulimet's ADB gist](https://gist.github.com/Pulimet/5013acf2cd5b28e55036c82c91bd56d8)** — community-maintained, frequently updated command list.
- **[Neupaneniraj: Common ADB Shell Commands](https://neupaneniraj.com.np/posts/Common-ADB-Shell-Commands/)** — organized breakdown of pm, dumpsys, and system interaction commands.
- **[42Gears: List of all widely used ADB commands](https://techblogs.42gears.com/list-of-all-widely-used-adb-commands/)** — 200+ commands including fastboot.

### Forensics-focused
- **[SJDC Forensics: Capturing Android Dumpsys/Logcat Logs](https://www.sjdcforensics.com/android-crash-logs/)** — forensic collection workflow.
- **[Digital Forensics blog: Triaging Modern Android Devices](https://blog.digital-forensics.it/2021/03/triaging-modern-android-devices-aka.html)** — android_triage script and methodology.

---

## 9. Quick Start Checklist

1. **Install Platform Tools** on your computer (or use Termux with `pkg install android-tools`).
2. **On the device:** enable Developer Options → USB Debugging → Wireless Debugging.
3. **Pair** your computer with the device via `adb pair` (or use Shizuku's built-in pairing flow).
4. **Install Shizuku** from [shizuku.rikka.app](https://shizuku.rikka.app/download/) (or IzzyOnDroid).
5. **Start Shizuku** via Wireless Debugging (no computer needed after first pairing) or via `adb shell sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh`.
6. **Verify:** `adb shell service list | grep shizuku` should show the Shizuku Binder service.
7. **Start interrogating:** use the commands in Section 5. For app-level access to protected APIs, either use an existing Shizuku-enabled tool from awesome-shizuku or integrate Shizuku-API into your own tooling.
8. **If you need root-level access** (reading app private data, full filesystem), use a rooted device with Shizuku in root mode or Sui Magisk module — or use an emulator/userdebug build where `adb root` works.

---

*End of primer. For anything not covered here, the Shizuku user manual and the HackTricks Shizuku page are the best next stops.*
