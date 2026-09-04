================================================================================
ADB + Shizuku Operational Workflows for Android Security Work
Research summary by Solar Pro4 — 2026-09-04
================================================================================

SECTION 1 — MOST USEFUL TOOL REPOS
================================================================================

1. RikkaApps/Shizuku (29.7k stars, Apache 2.0)
   https://github.com/RikkaApps/Shizuku
   The foundational privileged-process server. Starts a Java process via
   app_process with ADB/shell UID (2000) or root UID (0), exposes selected
   Android system APIs over Binder. This is the core infrastructure every
   other Shizuku-based tool builds on. Without it, none of the downstream
   apps or workflows function.

2. timschneeb/awesome-shizuku (10k stars, CC BY-SA 3.0)
   https://github.com/timschneeb/awesome-shizuku
   Curated catalog of Shizuku-using apps organized by category (terminals,
   file managers, debloaters, device-owner tools, automation flows).
   Essential for finding the right tool for a specific task. Also lists
   Automate flows for keeping Shizuku alive, and the rish shell client.

3. iamr0s/Dhizuku (3.8k stars, GPL-3.0)
   https://github.com/iamr0s/Dhizuku
   Shares Device Owner (DPM) permissions to third-party apps via Binder.
   Different privilege axis from Shizuku — grants uninstall, install,
   verify, and managed-profile powers rather than shell UID. Android allows
   only ONE device owner, so Dhizuku and Shizuku are complementary, not
   drop-in replacements.

4. ahmed-alnassif/AndroSH (240 stars, GPL-3.0)
   https://github.com/ahmed-alnassif/AndroSH
   No-root multi-distro Linux (Arch, Fedora, Alpine, Debian, Ubuntu, Kali,
   Void, Manjaro, openSUSE, Chimera) running via Shizuku/ADB + proot.
   Bundles Termux:X11 GUI. Useful when you need actual Linux tooling (binwalk,
   strings, foremost, tcpdump) on a non-rooted device for forensics or pentest
   workloads.

5. thejaustin/ShizukuPlus (fork, Samsung UID 1000 exploit)
   https://github.com/thejaustin/ShizukuPlus
   Enhanced Shizuku fork that adds a Samsung-specific UID 1000 system
   execution path on top of the standard ADB/shell mode. Relevant when
   working on Samsung devices where the vendor exploit gives you more than
   shell but less than full root.

6. RikkaApps/Shizuku-API (official API library)
   https://github.com/RikkaApps/Shizuku-API
   The developer library apps link against. Provides ShizukuBinderWrapper,
   UserService binding, permission grant flow, and UID detection. Any
   custom tooling or in-house audit app you build will use this.

7. DP-Hridayan/aShellYou (Shizuku-based terminal)
   https://github.com/DP-Hridayan/aShellYou
   Terminal app that uses Shizuku to execute shell commands locally on the
   device without PC or USB. Practical for on-device incident response when
   you can't connect a workstation.

8. HackTricks Shizuku page (reference methodology)
   https://hacktricks.wiki/en/mobile-pentesting/android-app-pentesting/shizuku-privileged-api.html
   Not a repo, but the best consolidated writeup on Shizuku architecture
   (Binder wrapper, UserService, shell vs root UID, security implications,
   detection ideas, abuse chains). Worth reading alongside the code.

---

SECTION 2 — CONCRETE COMMAND / WORKFLOW EXAMPLES
================================================================================

A. DEVICE ACCESS — starting and verifying Shizuku
--------------------------------------------------------------------------------

# 1. Start Shizuku via USB ADB (non-root)
adb shell sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh

# 2. Start Shizuku via Wireless Debugging (Android 11+, no USB)
#    First pair once: Settings > Developer Options > Wireless debugging > Pair device
#    Then in Shizuku app: "Start via Wireless debugging"
adb connect <DEVICE_IP>:5555
adb shell sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh

# 3. Start Shizuku via root (Magisk/KernelSU, persists across reboot as daemon)
su -c sh /data/adb/shizuku/start.sh

# 4. Verify Shizuku is running
adb shell dumpsys activity service moe.shizuku.privileged.api | head
adb shell service list | grep shizuku
# Expect to see moe.shizuku.privileged.api in the service list

# 5. Check effective privilege tier from a Shizuku-enabled app / rish shell
#    Shizuku.getUid() returns 2000 (shell) or 0 (root)
adb shell sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh
#    then open rish shell via Shizuku app "Use Shizuku in terminal apps"

# 6. Restart ADB server if Shizuku won't attach (stale auth)
adb kill-server && adb start-server
adb shell sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh

B. APP INTERROGATION — package, permission, and component enumeration
--------------------------------------------------------------------------------

# 7. List all packages (shell-equivalent via Shizuku proxy)
adb shell pm list packages
adb shell pm list packages -3                  # third-party only
adb shell pm list packages -s                  # system only
adb shell pm list packages -f                  # with APK path
adb shell pm list packages | grep -i camera    # filter by keyword

# 8. Inspect a single package in depth
adb shell dumpsys package com.example.target
#    Look for: userId, declared permissions, intents, activities, providers,
#    receivers, services, signatures, installLocation, sharedUserId

# 9. Find the UID and GID for a package
adb shell dumpsys package com.example.target | grep -E 'userId|gid'

# 10. Enumerate app permissions and compare granted vs requested
adb shell dumpsys package com.example.target | grep -A20 'granted=true'
adb shell dumpsys package com.example.target | grep permission

# 11. Grant / revoke permissions (works via Shizuku shell, no root needed)
adb shell pm grant com.example.target android.permission.READ_CONTACTS
adb shell pm revoke com.example.target android.permission.READ_CONTACTS
adb shell pm reset-permissions                  # reset all to defaults

# 12. Enable / disable (debloat) system and user apps
adb shell pm disable --user 0 com.google.android.apps.tips
adb shell pm enable com.google.android.apps.tips
adb shell pm uninstall --user 0 com.bloat.package   # remove for current user

# 13. Launch an activity or component for interaction testing
adb shell am start -n com.example.target/.MainActivity
adb shell am start -a com.example.target.SOME_ACTION -d "https://example.com"

# 14. Dump activity service state for a package
adb shell dumpsys activity services com.example.target
adb shell dumpsys activity providers com.example.target
adb shell dumpsys activity broadcasts              # recent broadcast history

# 15. Inspect app ops (granular runtime permission behavior)
adb shell appops get com.example.target
adb shell appops set com.example.target READ_CONTACTS deny
adb shell appops set com.example.target READ_CONTACTS allow

C. FILE SYSTEM FORENSICS — what shell can and cannot reach
--------------------------------------------------------------------------------

NOTE: Shell UID (2000) CANNOT read /data/user/0/<package> private sandbox
directly. You get: /sdcard (shared storage), app cache dirs you own,
Android/data/<package> (Android 11+ scoped storage visibility), and world-
readable paths. Root unlocks everything below /data.

# 16. Pull shared storage for triage
adb pull /sdcard/Download/ ./evidence/Download/
adb pull /sdcard/Pictures/ ./evidence/Pictures/
adb pull /sdcard/DCIM/ ./evidence/DCIM/

# 17. List Android/data contents (visible since Android 11 to shell)
adb shell ls -la /sdcard/Android/data/
adb shell ls -la /sdcard/Android/data/com.example.target/
adb shell ls -la /sdcard/Android/obb/

# 18. Pull known APK path from pm list packages -f
adb shell pm path com.example.target
#    Returns: package:/data/app/~~/base.apk
adb pull /data/app/~~/com.example.target-==/base.apk ./evidence/

# 19. Copy app cache/logs if permissions allow (depends on OEM)
adb shell ls -la /data/local/tmp/
adb shell cat /data/local/tmp/some_shared_log  # if world-readable

# 20. AndroSH: drop into a Kali/Ubuntu proot for Linux forensics tooling
#    Prerequisite: Shizuku running, AndroSH installed in Termux
cd ~/AndroSH && python main.py launch <env_name>
#    Then inside the proot:
binwalk image.jpg
strings -n 8 suspicious.apk
foremost -i evidence.img -o output/
tcpdump -i any -w capture.pcap              # if network interface accessible

D. LOG COLLECTION — live and post-incident
--------------------------------------------------------------------------------

# 21. Dump full logcat to file (live device)
adb logcat -d > logs/logcat_$(date +%Y%m%d_%H%M%S).txt

# 22. Filter logcat by tag, PID, or priority
adb logcat -d -s MainActivity              # specific tag
adb logcat -d | grep -i "com.example.target"
adb logcat -d -b all                       # all buffers (main, system, crash, radio)

# 23. Live tail for interactive monitoring
adb logcat | grep -E "FATAL|Exception|com.example.target"
adb logcat -v threadtime                    # high-resolution timestamps

# 24. Full bugreport (heavy, captures logs + dumpsys + traces + last_kmsg)
adb bugreport > evidence/bugreport_$(date +%Y%m%d).zip

# 25. Targeted dumpsys for specific subsystems
adb shell dumpsys batterystats --checkin
adb shell dumpsys meminfo com.example.target
adb shell dumpsys netstats
adb shell dumpsys connectivity
adb shell dumpsys window windows            # view hierarchy, focused window

# 26. Collect crash traces
adb shell cat /data/system/dropbox/        # if accessible (often restricted)
adb logcat -d -b crash > evidence/crash.txt
adb shell dumpsys dropbox                  # dropbox manager state

E. SHIZUKU-SPECIFIC TERMINAL WORKFLOW (rish shell, no PC)
--------------------------------------------------------------------------------

# 27. Enable rish shell in Shizuku app: Settings > "Use Shizuku in terminal apps"
#     Shizuku downloads rish binary. Then from a terminal app on-device:

shizuku shell                               # interactive shell with shell UID 2000
shizuku exec "pm list packages -3"         # single command
shizuku open                                # launch Shizuku app GUI

# 28. Termux + Shizuku integration (no PC at all)
#     In Termux:
pkg install wget
wget https://rikka.app/rish/latest -O rish && chmod +x rish
#     Then use rish or the Shizuku Termux integration to run shell commands
#     directly on-device with ADB-equivalent privileges.

F. ADB BACKUP / RESTORE (legacy, limited Android 12+)
--------------------------------------------------------------------------------

# 29. Backup a single app with APK (works best on Android 9-11; limited post-12)
adb shell pm list packages -f -3
adb backup -f evidence/com.example.target.ab -apk com.example.target
adb restore evidence/com.example.target.ab

# 30. Full device backup (noisy, often restricted on modern Android)
adb backup -apk -shared -system -all -f evidence/full_device.ab

---

SECTION 3 — SHIZUKU vs DHIZUKU vs ROOT, AND PRIVILEGE TIERS
================================================================================

A. THE THREE MODES OF ELEVATED ACCESS
--------------------------------------------------------------------------------

SHIZUKU (ADB/shell mode, UID 2000):
  - How: User enables USB or Wireless debugging, Shizuku app runs
    start.sh via adbd, which launches a Java process via app_process
    running as the shell user (uid=2000).
  - What you get: Everything adb shell can do — pm, dumpsys, am, appops,
    settings (some), cmd connectivity, logcat, shared storage access.
    Binder proxy: Shizuku receives requests from authorized apps, forwards
    them to system services, returns results. The app sees the same Binder
    interface as if it were calling the system service directly.
  - What you DON'T get: Reading another app's private data directory
    (/data/user/0/<pkg>), modifying system partitions, loading kernel mods,
    accessing hardware that requires SYSTEM or ROOT uid, bypassing hidden-
    API restrictions from a normal app process (use UserService instead).
  - Persistence: No root = no auto-start at boot. Session dies on reboot
    unless you re-run start.sh. Root mode can install as a daemon at /data/adb/
    so it survives reboot.
  - OEM quirks: MIUI/HyperOS often needs "USB debugging (Security settings)"
    beyond normal USB debugging. ColorOS/OxygenOS may need permission-monitor
    disabled. Android 11+ "Disable adb authorization timeout" helps stability.
  - Detection: service list | grep shizuku, dumpsys activity service
    moe.shizuku.privileged.api. Malware may mimic with its own shell-owned
    Binder helper.

DHIZUKU (Device Owner mode, DPM powers, no UID elevation):
  - How: User wipes device (or uses QR provisioning), makes Dhizuku the
    Device Owner via Android's DevicePolicyManager. Dhizuku then exposes
    Device Owner permissions to other apps via its API.
  - What you get: Install/uninstall apps system-wide, verify apps, manage
    managed profiles, lock task mode, set global/hidden settings, disable
    safe boot. This is a DIFFERENT privilege axis from Shizuku — it's about
    MDM/enterprise control, not shell command execution.
  - What you DON'T get: shell UID, direct command execution, dumpsys, pm
    internals beyond what DPM exposes, file system access beyond normal app.
  - Relationship to Shizuku: Dhizuku's own README notes it can use Shizuku
    as a backend for some operations ("Dhizuku Mode"), and ShizukuPlus lists
    Dhizuku as a component. They are complementary, not competing. Android
    only allows one Device Owner, so you pick Dhizuku or another DO app.
  - Use case: Enterprise device management, kiosk mode, remote app control,
    sideloading without Play Store, managed-profile security testing.

ROOT (UID 0, full Linux root on device):
  - How: Magisk, KernelSU, or kernel exploit. Shizuku can run in root mode
    via `su -c sh /data/adb/shizuku/start.sh`, which gives the Shizuku
    server process UID 0 instead of 2000.
  - What you get: Everything shell can do PLUS: read/write any file
    (including /data/user/0/<pkg>), remount system/vendor partitions,
    load kernel modules, full logcat including kernel logs, ptrace any
    process, access hardware directly, bypass most permission checks.
  - Trade-offs: Detected by SafetyNet/Play Integrity (unless hidden with
    Magisk Hide / Zygisk), wipes on some OTA updates, higher risk of bricking,
    not available on locked bootloader devices without an exploit.

B. SHELL (UID 2000) vs ROOT (UID 0) — PRACTICAL DIFFERENCE TABLE
--------------------------------------------------------------------------------

Capability                          Shell (UID 2000)     Root (UID 0)
----------------------------------- --------------------  -------------------
pm list/enable/disable/uninstall    YES (user scope)     YES (system-wide)
dumpsys (most services)             YES                  YES
appops get/set                      YES                  YES
am start/instrument                  YES                  YES
logcat -d                           YES                  YES (all buffers)
pull /sdcard/*                      YES                  YES
pull /data/user/0/<pkg>             NO (sandbox)         YES
pull /system /vendor /product       NO (read-only fs)    YES (if not A/B locked)
install APKs (PackageInstaller)     YES (with Shizuku)   YES (with Shizuku + more flags)
load native libraries / JNI         via UserService only YES (direct)
bypass hidden API restrictions      NO (in app process)  YES (or via UserService)
SELinux                              Enforced             Enforced (unless setenforce 0)
kernel logs (dmesg, pstore)         NO                   YES
ptrace / process injection          NO                   YES

C. KEY PRACTICAL NOTES
--------------------------------------------------------------------------------

- Shizuku.getUid() tells you which tier you're running. 2000 = shell,
  0 = root. Write tooling to check this and degrade gracefully.

- Even with shell UID, Android permissions, SELinux, and OEM-specific
  restrictions apply. A command that works via `adb shell` on a Pixel may
  fail through Shizuku on a Samsung because the OEM trimmed shell permissions.

- UserService is the modern way to run long-lived Java/JNI code in the
  Shizuku server process. Prefer it over spawning repeated shell commands
  for offensive tooling that needs state or repeated Binder operations.

- Shizuku's Binder wrapper (ShizukuBinderWrapper) makes the app believe it's
  calling the system service directly. The actual transaction runs in the
  Shizuku server process, not the app process. This matters for hidden API
  restrictions — code in the app process is still limited; code in the
  UserService / server process is not.

- Wireless debugging (Android 11+) simplified Shizuku setup but also creates
  a security concern: if an attacker can re-enable Wireless debugging (e.g.,
  via Accessibility + WRITE_SECURE_SETTINGS abuse) and has the ADB pairing
  keys, they can reconnect and recreate a shell session post-boot without
  root. Treat the Accessibility + Wireless debugging + shell-Backed Binder
  chain as a high-risk post-exploitation path.

- For persistence on a non-rooted device, Automate flows exist (Better Shizuku
  Starter, Shizuku Keeper, Shizuku Keeper Lite — listed in awesome-shizuku).
  These watch for Shizuku dying and restart it via wireless debugging on
  events like screen on, boot completed, or periodic checks. Only works if
  the device is already authorized for wireless debugging.

---

SECTION 4 — QUICK-REFERENCE WORKFLOW DECISION TREE
================================================================================

DO YOU HAVE ROOT?
  YES → Shizuku in root mode (`su -c sh /data/adb/shizuku/start.sh`) gives
        UID 0, survives reboot as daemon, can read all /data.
  NO  → Shizuku in ADB/shell mode (UID 2000) gives most pm/dumpsys/am/logcat
        primitives without root.

DO YOU NEED TO READ ANOTHER APP'S PRIVATE DATA?
  YES → Need root, or need to use a Shizuku-enabled file manager that uses
        scoped-storage APIs (e.g., X-plore with Shizuku for Android/data).
        Shell UID cannot read /data/user/0/<pkg> directly.
  NO  → Shell mode is sufficient for most auditing, debloating, and log work.

DO YOU NEED TO INSTALL/UNINSTALL SYSTEM-WIDE OR MANAGE DEVICE POLICY?
  YES → Consider Dhizuku for Device Owner powers (install, uninstall, verify,
        lock task, managed profile). Or get root for full pm control.
  NO  → Shizuku shell for per-user disable/uninstall and permission changes.

DO YOU NEED LINUX TOOLING (binwalk, foremost, tcpdump)?
  YES → AndroSH: Shizuku + proot + Termux:X11 gives you a Kali/Ubuntu/Arch
        environment without root. Good for forensics and pentest on a handset.
  NO  → Standard Shizuku shell + adb pull is enough.

ARE YOU BUILDING CUSTOM TOOLING?
  YES → Use Shizuku-API (ShizukuBinderWrapper + UserService) for Binder-level
        access. Add the Shizuku API permission and ShizukuProvider to your
        manifest. Check Shizuku.getUid() at runtime to know your tier.
  NO  → Use existing Shizuku-enabled apps from awesome-shizuku list.

---

END OF REPORT
================================================================================
