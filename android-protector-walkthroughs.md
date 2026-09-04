# Android Commercial Protector Unpacking — End-to-End Walkthroughs & Case Studies

> Forensic narratives of taking specific commercial-protected Android malware samples from APK → decrypted DEX → understanding the payload. This is the craft documentation — *how* researchers actually unpack real samples, what fails, what they do when it fails, how they chain approaches. Not a tooling landscape (that's in `android-commercial-protector-bypass.md`); not a Frida survey (that's in `android-re-frida-pipeline.md`). This is the story of the analysis itself.

---

## 1. Why Walkthroughs Matter More Than Tooling Lists

A tooling list tells you what exists. A walkthrough tells you how it actually gets used on real samples — what the researcher tried first, what blocked them, what they did when the first approach didn't work, and what the final chain of extraction looked like.

The difference is the difference between a catalog and a craft. You can read a list of every Android unpacking tool and still not know:

- Which approach to try first on *this* sample, given *this* protector and *this* anti-analysis profile.
- What to do when Frida gets blocked mid-extraction and the decrypted DEX is only in memory for 200 milliseconds.
- How to recognize that you're dealing with a packed unpacker and not just a normal protector.
- What the decrypted payload typically looks like after extraction — and what signals told the researcher they'd gotten to the real malware.

Walkthroughs encode the failure recovery loops that tooling lists omit. They show the pivot points: when to switch from dynamic to static, when to move from Java-level analysis to native, when to abandon an automated tool and write a targeted hook, when to accept that the sample requires per-app manual work.

They also document the payload — what the malware actually did after extraction. The unpacking is the means; understanding the malware is the end. A walkthrough that stops at "here's the decrypted DEX" without analyzing what the malware does is only half a case study.

Finally, walkthroughs are how techniques propagate. A researcher who solves a novel protector configuration publishes the narrative so the next person doesn't have to rediscover the same approach. The Black Hat/DEF CON/Recon talk circuit, vendor blogs, researcher blogs, and the Chinese RE community all serve this function — they're where the craft gets taught.

---

## 2. Narrative Template — What a Good Unpacking Case Study Looks Like

When documenting your own unpacking, follow this template. It forces you to capture the decision points, not just the outcome.

### 2.1 Sample Identification

- **Hash and source:** SHA-256 of the APK, where you got it (VirusTotal, AndroZoo, malware feed, client submission).
- **Initial triage:** What made this sample interesting? AV detections? Suspicious permissions? Network indicators? A protector that looked unusual? The trigger that made you invest time in unpacking it rather than treating it as a routine scan.

### 2.2 Protector Identification

- **What protector was detected, and how:** APKiD output, manual inspection of DEX structure (encrypted secondary DEX files, invalid headers), native library names, manifest entries (custom Application class, protector-specific components).
- **Confidence level:** Is this a clear APKiD hit, a probable manual identification, or an unknown protector that you inferred from behavior?
- **Variant/version clues:** Some protectors leave version markers, configuration fingerprints, or library naming patterns that hint at which variant you're dealing with. Note them.

### 2.3 Initial Static Analysis (Before Unpacking)

What could you see before you extracted the decrypted DEX?

- Manifest: permissions, exported components, custom Application class, services/receivers/activities of interest.
- Partial DEX: if the primary DEX isn't fully encrypted, what classes are visible? Is there a loader stub? A decryption initialization class?
- Native libraries: how many, what sizes, what names, what export symbols are visible, what anti-analysis patterns are apparent from strings and function names.
- Resources/assets: encrypted containers, unusual file names, high-entropy files.
- API usage: what dangerous APIs are referenced even in the encrypted/partial code (SMS, telephony, accessibility, network, package installation)?
- This section establishes the baseline — what you knew before you invested in unpacking, and what hypotheses you formed about what the malware does.

### 2.4 The Unpacking Approach(es) Attempted

List what you tried, in order, with the reasoning for each:

1. **First approach and why:** e.g., "Ran BlackDex first because it's fast, automated, and has coverage across many protectors. Hoped for a one-shot extraction."
2. **What you hoped to capture:** decrypted DEX, strings, resources, native library, memory dump.
3. **What the protector's anti-analysis profile suggested:** Did you expect anti-Frida? Anti-debug? Integrity checks? Early decryption? This informed your sequencing.

### 2.5 What Worked

The approach that actually produced the decrypted payload. Be specific:

- Which tool or technique (Frida hook on which API, BlackDex, memory dump, patched APK, static loader analysis).
- What point in the app lifecycle you targeted (early decryption, class load time, native loader function).
- What you captured and in what form (full DEX file, partial DEX, string dump, memory dump that required carving).
- Any intermediate steps that were necessary (anti-analysis bypass first, then extraction).

### 2.6 What Didn't Work and Why

This is the most valuable section for the next researcher. List the approaches that failed and the specific blocker:

- **Anti-debug blocked Frida:** app detected the debugger and crashed/quit before extraction.
- **Anti-tamper crashed on patching:** patched APK failed integrity check and wouldn't run.
- **Integrity checks detected the unpacker:** signature or checksum verification caught the modification.
- **Decryption timing:** DEX decrypted before Frida could attach, or the window was too narrow.
- **DEX re-encrypted between loads:** captured DEX was only valid for one class load; subsequent loads got re-encrypted data.
- **Native anti-analysis:** the native library that performs decryption also has anti-debug/anti-Frida/control flow flattening that blocked dynamic analysis.
- **Stacked protectors:** two protectors layered — DexGuard for DEX plus a separate native protector — and the approaches for one didn't carry over to the other.

For each failure, note what you tried to recover and whether it worked.

### 2.7 The Decrypted Payload Analysis

What was in the decrypted DEX, and what did the malware do?

- **Capabilities:** banking fraud, SMS fraud, ad fraud, spyware, downloader/dropper, RAT, credential theft, etc.
- **Specific behaviors:** what APIs were called, what data was collected, where was it sent, what was the C2 infrastructure (domains, IPs, protocols).
- **Obfuscation remaining after extraction:** was the decrypted DEX still obfuscated (string encryption, control flow flattening, virtualization)? Did it require further deobfuscation?
- **Second-stage payloads:** was there a downloaded JAR, a secondary installation, a remote payload? What did it do?

### 2.8 Lessons Learned

What would you do differently next time?

- Did you start with the right tool, or would a different first approach have saved time?
- Did you spend too long on a blocked approach before pivoting?
- Was there an earlier signal you missed that would have pointed you to the right extraction path?
- What was surprisingly hard? What was surprisingly easy?
- What do you now know about this protector/variant that you didn't know at the start?

---

## 3. Packed Unpacker — Black Hat 2020 (Maddie Stone) Deep Analysis

### 3.1 Sample / Protector

- **Sample:** An Android app whose native library performed anti-analysis — anti-debug, anti-Frida, anti-dumping, environment checks — and that native library was itself packed and obfuscated.
- **Protector/pattern:** This is the "Packed Unpacker" pattern — the anti-analysis layer is itself the first layer you have to defeat. Not a specific commercial protector product, but a structural pattern that appears across well-protected apps (and malware using such protection).
- **Source:** Maddie Stone, Black Hat USA 2020, "Unpacking the Packed Unpacker: Reverse Engineering an Android Anti-Analysis Native Library." Also published as a Virus Bulletin 2019 paper.

### 3.2 Methodology

**The chicken-and-egg problem.** The native library is the thing that does the decryption and anti-analysis. You need to understand the anti-analysis to bypass it. But the anti-analysis is designed to prevent analysis of the native library. And the native library itself is packed — encrypted or obfuscated — so you can't statically read it without first unpacking it.

The methodology Stone documented followed a layered approach:

**Layer 1 — Static analysis of the packed native library.** Even packed, the native library has structure. Ghidra/IDA can show the packing stub, the entry points, the strings that are visible before unpacking, the control flow of the loader. The goal at this layer is to understand *what the packing looks like* — is it XOR-based, is it a custom format, does it have a recognizable stub structure, where does the unpacking code live?

**Layer 2 — Dynamic analysis to capture the unpacked library in memory.** The packed native library decrypts/unpacks itself at runtime (presumably during `JNI_OnLoad` or library constructor). Running the app under Frida and dumping process memory at the right moment can capture the unpacked native library in its decrypted form. This requires timing the dump after unpacking but before any anti-analysis checks that might detect the dumping. Alternatively, if the library writes the unpacked version to a file (even temporarily), that file can be captured.

**Layer 3 — Analysis of the unpacked anti-analysis code.** Once the native library is unpacked, you can read the anti-analysis logic. This is where you learn what the library checks for: debuggers (ptrace, TracerPid, `isDebuggerConnected`), Frida (maps scanning, process checks, library checks), emulators (hardware fingerprints, sensor presence), rooted devices (su binaries, Magisk, debuggable flags), specific processes, specific filesystem artifacts. Each check is a barrier you need to bypass to continue analysis.

**Layer 4 — Bypass of the anti-analysis checks.** With the checks understood, you can bypass them — either by patching the native library (NOPping the checks, returning early), by using Frida to hook the check functions and force them to return "clean" results, or by running the app in an environment that satisfies the checks (non-rooted, non-debugged, non-emulated). Only then can you proceed to analyze the rest of the app (the decrypted DEX, the malware logic).

### 3.3 What Made It Notable

This is a **meta-problem.** The defender's tool (anti-analysis) is the very thing you need to defeat to analyze the defender's payload. And defeating it requires understanding it, which requires unpacking it, which requires bypassing the anti-analysis... The recursion is the point. It's a deliberately constructed obstacle where each layer protects the next.

This pattern shows up in well-protected apps (legitimate apps with strong security requirements) and in malware that borrows commercial protector features. It's not the most common protection pattern, but when you encounter it, it dominates the analysis timeline.

### 3.4 What Was Found (Anti-Analysis Techniques)

From the unpacked native library, the anti-analysis checks included:

- **Debugger detection:** `ptrace` self-attachment to prevent other debuggers, `TracerPid` reading from `/proc/self/status`, `isDebuggerConnected()` checks via JNI.
- **Frida detection:** scanning `/proc/self/maps` for Frida signatures, checking for `frida-server` processes, looking for known Frida strings in memory, detecting the Frida JavaScript bridge.
- **Emulator detection:** hardware fingerprint checks, device name verification, sensor presence checks, build property inspection.
- **Root detection:** checking for `su` binaries, Magisk indicators, debuggable build flags, test-key signatures.
- **Environment checks:** specific filesystem artifacts, process lists, installed package lists that would indicate a analysis environment.

These are the standard anti-analysis battery, but the notable finding is that they were implemented in a native library that was itself packed — so a naive analyst who tried to statically analyze the `.so` would hit the packing layer first, and a naive analyst who tried to dynamically analyze the app would hit the anti-analysis layer first. Both layers had to be defeated in order.

### 3.5 What Worked

- **Static analysis of the packing stub** gave enough understanding of the native library's structure to inform the dynamic approach.
- **Memory dumping at the right moment** captured the unpacked native library, which could then be analyzed statically without the packing layer.
- **Understanding the anti-analysis checks** from the unpacked library allowed targeted bypass, which then enabled analysis of the rest of the app.

### 3.6 What Didn't Work (Implied)

- **Direct static analysis of the packed `.so`** was unproductive — the packing obscured the logic.
- **Naive Frida attachment** would have hit the anti-analysis checks before any useful extraction.
- **Automated unpacking tools** would not have handled the dual-layer (packed native + anti-analysis native) structure — this required manual work at each layer.

### 3.7 Payload

The payload was the anti-analysis layer itself — the malware's primary capability was making analysis difficult. In the broader context of the app (beyond what Stone's talk focused on), the app likely contained additional malicious logic that was protected by this anti-analysis layer. The point of the case study was the unpacking challenge, not the final payload analysis.

### 3.8 Lessons Learned

- **When the native library is the protector, analyze it as a first-class target, not as a side note.** The decryption logic and the anti-analysis logic both live there. Understanding the `.so` is often the key to the whole app.
- **Packed protectors create recursive analysis problems.** Don't assume the first unpacker you run will handle a packed unpacker — you may need to unpack the unpacker itself.
- **Anti-analysis is not a monolith.** The checks are individual functions with individual bypass paths. Understanding them one by one (from the unpacked library) is more effective than trying to generically bypass "anti-analysis."
- **The ordering matters.** Static analysis of the packing stub → dynamic capture of the unpacked library → static analysis of the unpacked library → bypass → analyze the rest. Each step enables the next.

---

## 4. BRATA — Two-Stage Dropper Walkthrough

### 4.1 Sample / Protector

- **Sample:** BRATA (BRute ARAndo Transfer Application / "BRATA" — a mobile banking trojan dropper). Multiple variants have been analyzed by Lookout, Zimperium, and other vendors over 2020–2025.
- **Protector:** Not necessarily a commercial protector in all variants; some versions use light obfuscation, some use commercial protectors. The walkthrough focus here is the two-stage dropper structure, which is independent of the protector layer.
- **Source:** Lookout, Zimperium, and other vendor analyses of BRATA as a banking trojan dropper. Also referenced in the reverse-engineering community as a classic two-stage dropper pattern.

### 4.2 Methodology

**Stage 1 — Static analysis of the dropper APK.** The dropper APK is deliberately lightweight. It may request minimal permissions, contain little visible malicious code, and look like a generic utility or a benign-looking app. The key static finding is typically a `DexClassLoader` or `PathClassLoader` usage, or a reference to downloading and installing a package. The dropper's manifest may show a small set of components and a minimal permission set.

**Stage 2 — Dynamic analysis of the dropper.** Run the dropper on a controlled device/emulator with Frida and network capture. Observe:

- What URL the dropper connects to for the payload download.
- What it downloads (a DEX file, a JAR, a full APK).
- Where it stores the downloaded payload (internal storage, external storage, a specific path).
- What it does with the payload — does it load it via DexClassLoader? Does it install it as a separate package? Does it extract and execute it in-process?

**Stage 3 — Capture the downloaded payload.** The payload is not in the original APK. It must be captured from the network (MITM the download) or from the filesystem after download (pull the file from the device). If the payload is downloaded and then loaded into the same process via DexClassLoader, a Frida hook on the class loading can capture the decrypted/loaded DEX. If it's installed as a separate APK, pull the installed APK from `/data/app/` or the install path.

**Stage 4 — Analyze the payload separately.** The payload is the real malware — the banking trojan logic, the SMS fraud, the overlay attacks, the credential harvesting. Analyze it with standard tools (JADX, apktool, Frida hooks on the payload's behavior).

### 4.3 What Worked

- **Network capture of the download.** MITM or proxy capture of the payload download URL gave researchers the payload without needing to extract it from the device. This is the most direct path when the dropper downloads over HTTP/HTTPS (HTTPS requires SSL pinning bypass first).
- **Filesystem capture after download.** Pulling the downloaded file from the device's storage gave the payload APK/DEX/JAR directly.
- **Frida DexClassLoader hooks.** When the payload was loaded into the same process, hooking the class loader captured the payload DEX in memory.

### 4.4 What Didn't Work (Common Failure Points)

- **Stopping at the dropper APK.** Static analysis of the dropper alone showed little — researchers who didn't proceed to dynamic analysis would miss the payload entirely.
- **SSL pinning blocking MITM.** If the dropper used HTTPS with pinning, a naive MITM proxy would fail. The pinning had to be bypassed (Frida hook on TrustManager, universal bypass script) before the download could be captured.
- **Payload deleted after loading.** Some droppers delete the downloaded file after loading it into memory, leaving no filesystem trace. In that case, the payload had to be captured from memory (Frida dump at load time) or from the network (MITM during download).
- **Payload encrypted in transit.** If the downloaded payload was encrypted and decrypted after download, capturing the encrypted file from the network or filesystem gave you an encrypted payload that still needed unpacking.

### 4.5 Payload

The BRATA payload is a banking trojan. Capabilities typically include:

- **Overlay attacks:** fake login screens overlaid on legitimate banking apps to capture credentials.
- **SMS interception:** reading SMS messages for 2FA codes, banking alerts.
- **Accessibility abuse:** using AccessibilityService to read screen content, simulate taps, capture input.
- **Contact/data exfiltration:** reading contacts, call logs, device information, sending to C2.
- **Remote control:** some variants include remote command execution, allowing the attacker to control the device.
- **Persistence:** installing additional components, surviving reboots, hiding from the user.

The exact capabilities vary by variant and version.

### 4.6 Lessons Learned

- **The dropper is not the malware.** The APK on disk is the delivery mechanism; the malware is what gets downloaded. Analyzing only the dropper gives you a partial and misleading picture.
- **Capture the network traffic.** The download URL is the bridge between the dropper and the payload. Capturing it (with pinning bypass if needed) is often the fastest path to the real payload.
- **Capture the installed payload.** If the dropper installs a separate package, pull that package from the device. It's a standalone APK that can be analyzed independently.
- **Two apps on the device after infection.** A hallmark of the two-stage dropper: the user installs one app, and after running it, there are two apps on the device — the harmless-looking dropper and the hidden malware. Detecting this on a device (via `pm list packages`, comparing pre- and post-install package lists) is a forensic signal.

---

## 5. Ghimob — Downloader Payload Analysis

### 5.1 Sample / Protector

- **Sample:** Ghimob — a mobile banking trojan downloader. Documented by Kaspersky (Securelist) and other vendors in Latin American threat reports.
- **Protector:** Varies by sample; some Ghimob samples use light obfuscation, some use commercial protectors. The walkthrough focus is the downloader → payload structure.
- **Source:** Kaspersky Securelist reports on Latin American banking trojans, vendor analyses of Ghimob as a downloader family.

### 5.2 Methodology

The methodology mirrors BRATA's — Ghimob is also a downloader/dropper, and the analysis approach is the same pattern:

**Stage 1 — Static analysis of the downloader.** The downloader APK looks lightweight. Its manifest shows a minimal permission set and a small number of components. Static analysis may reveal network code (HTTP/HTTPS client, URL construction), file I/O code (writing the download to storage), and possibly package installation code (if the payload is installed as a separate APK). The downloader may also include environment checks (root, emulator, specific device models) to decide when to download and install.

**Stage 2 — Dynamic analysis of the downloader.** Run the downloader on a controlled device with Frida and network capture. Observe:

- What URL the downloader connects to.
- What it downloads.
- When it downloads (on first launch? after a delay? triggered by a specific event?).
- What it does with the downloaded payload.

**Stage 3 — Capture the payload.** Same as BRATA: network capture, filesystem capture, or memory capture at load time.

**Stage 4 — Analyze the payload.** The payload is the banking trojan — analyze its capabilities, C2, and behavior.

### 5.3 What Worked

- **Dynamic observation of the download trigger.** Understanding when and how the downloader initiates the download (e.g., on first launch, after a specific time, when a banking app is detected) helped researchers time their capture.
- **Network capture of the payload download.** The same MITM approach as BRATA.
- **Payload pulled from device storage.** If the payload was written to a known path, pulling it directly was straightforward.

### 5.4 What Didn't Work

- **Static-only analysis.** The downloader's static footprint is intentionally minimal. Without dynamic analysis, the payload URL and the payload itself remain invisible.
- **Environment checks blocking the download.** If the downloader checks for root, emulator, or specific device conditions and refuses to download in an analysis environment, the payload never gets downloaded. Bypassing the environment checks (Frida hooks, environment spoofing) is necessary to trigger the download in the lab.
- **Download delayed or conditional.** Some downloaders wait for a specific trigger (a banking app being launched, a specific time of day, a remote command). Researchers who didn't trigger the condition would never see the download.

### 5.5 Payload

The Ghimob payload is a banking trojan targeting Latin American banks. Capabilities include:

- **Banking app detection:** the payload may check for specific banking apps installed on the device and activate when they are launched.
- **Overlay attacks:** fake login screens for banking apps.
- **SMS/2FA interception:** reading SMS for banking codes.
- **Credential theft:** capturing login credentials, personal data.
- **Remote C2:** receiving commands, exfiltrating data.
- **Accessibility abuse:** reading screen content, simulating input.

The specific banks targeted and the specific techniques vary by campaign and version.

### 5.6 Lessons Learned

- **Downloaders are delivery mechanisms, not the malware itself.** Treat the downloader as the installer, not the payload. The malware is what gets installed.
- **Trigger conditions matter.** If the downloader waits for a condition, you need to understand and replicate that condition in your analysis environment, or bypass the check.
- **Environment checks are a common blocker.** Downloaders often check for analysis environments (root, emulator, specific device models) and refuse to operate. Bypassing these checks is often the prerequisite for capturing the payload.
- **The payload is a standalone APK.** Once captured, the payload can be analyzed independently with standard tools. You don't need the downloader anymore.

---

## 6. Joker — Multi-Stage Infection Chain Walkthrough

### 6.1 Sample / Protector

- **Sample:** Joker — a malware family that uses a multi-stage infection chain. Variants have been analyzed by Zimperium, Check Point, Trend Micro, and others. The family is known for being initially misdetected as adware or benign because the visible APK code looks harmless.
- **Protector:** Joker samples have been found with various levels of protection. Some use light obfuscation; some rely on the multi-stage structure itself as the primary evasion — the malicious code is simply not visible in the APK.
- **Source:** Zimperium, Check Point Research, Trend Micro, and other vendor analyses of Joker as a subscription fraud / malware family. Also referenced in the reverse-engineering community as a classic multi-stage chain.

### 6.2 Methodology

**Stage 1 — Static analysis of the APK.** The visible Java/Smali code in the APK looks harmless. It may be a simple utility, a game, or a generic app with minimal functionality. Standard static analysis (JADX, apktool, manifest inspection) shows little of interest — few dangerous permissions, no obvious malicious API usage, benign-looking components.

**Stage 2 — Finding the hidden DEX.** The key discovery is that the APK contains a hidden DEX file (in `assets/`, `res/`, or another container) and that the app uses `DexClassLoader` (or `PathClassLoader`) to load this hidden DEX at runtime. Static analysis reveals:

- A `DexClassLoader` or `PathClassLoader` instantiation with a path pointing to a file inside the APK (assets, res, or a path constructed at runtime).
- The hidden DEX file itself — visible in the APK's file listing but not in the standard `classes.dex`.
- Possibly a reflection-based invocation of a class from the loaded DEX (e.g., `yin.Chao` or similar class name).

**Stage 3 — Dynamic analysis to observe the loading and the next stage.** Run the app with Frida hooks on `DexClassLoader`/`PathClassLoader` constructors and `loadClass`. Observe:

- The hidden DEX being loaded from its path inside the APK.
- The class from the hidden DEX being loaded (the `yin.Chao` class or equivalent).
- The loaded class downloading another file from a remote server — typically a JAR file.
- The downloaded JAR being loaded or executed as the next stage.

**Stage 4 — Capture each stage.** The multi-stage chain means you need to capture each stage separately:

- **Stage 1 (the APK):** already have it on disk.
- **Stage 2 (the hidden DEX):** capture it from the APK's assets (it's already on disk, just not in `classes.dex`), or capture it from memory at load time via Frida.
- **Stage 3 (the downloaded JAR):** capture it from the network (MITM the download) or from the filesystem after download. This is the stage that contains the real malicious logic.

**Stage 5 — Analyze each stage.** Each stage adds capability:

- The APK may contain environment checks, download logic, and the loader mechanism.
- The hidden DEX may contain the class that performs the download and the loading of the next stage.
- The downloaded JAR contains the actual malicious payload — subscription fraud, SMS fraud, ad fraud, or other capabilities.

### 6.3 What Worked

- **Static discovery of the hidden DEX.** Finding the hidden DEX file in the APK and the `DexClassLoader` reference gave researchers the trigger to investigate further. Without this static discovery, the dynamic analysis might not have been targeted at the right point.
- **Frida DCL hooks to observe the loading sequence.** Hooking `DexClassLoader`/`PathClassLoader` showed exactly when and how the hidden DEX was loaded, and what happened immediately after (the JAR download).
- **Network capture of the JAR download.** MITM or proxy capture of the downloaded JAR gave researchers the final payload without needing to extract it from the device.

### 6.4 What Didn't Work

- **Static-only analysis stopping at "looks harmless."** The APK's visible code is designed to look harmless. A researcher who did only static analysis and concluded "nothing interesting here" would miss the entire infection chain.
- **Missing the hidden DEX in assets.** The hidden DEX is in the APK but not in the standard DEX slot. A researcher who only looked at `classes.dex` and the decompiled Java would miss it. Inspecting the full APK file listing (all files in `assets/`, `res/`, etc.) is necessary.
- **Missing the JAR download.** The downloaded JAR is the real payload. If the researcher didn't capture network traffic or didn't notice the download happening, they'd miss the final stage.
- **SSL pinning on the JAR download.** If the download used HTTPS with pinning, the MITM capture would fail without a pinning bypass.

### 6.5 Payload

The Joker payload (in the downloaded JAR) is typically subscription fraud and/or SMS fraud:

- **Subscription fraud:** tricking the user into subscribing to paid services via hidden or obfuscated consent flows, often using SMS or USSD.
- **SMS fraud:** sending SMS messages to premium numbers, intercepting SMS for 2FA or banking codes.
- **Ad fraud:** loading ads in the background, clicking ads, generating fraudulent ad revenue.
- **Data exfiltration:** sending device information, contacts, or other data to C2.
- **Additional capabilities:** some Joker variants include overlay attacks, accessibility abuse, or other techniques depending on the version.

The exact payload varies by variant and campaign. The defining characteristic is the multi-stage delivery — the APK looks harmless, the hidden DEX adds the download capability, and the downloaded JAR delivers the actual fraud.

### 6.6 Lessons Learned

- **Multi-stage malware requires multi-stage analysis.** Each stage of the chain must be captured and analyzed separately. Static analysis gives you Stage 1; dynamic analysis + network capture gives you Stages 2 and 3.
- **The hidden DEX is on disk but not in the standard decompilation.** Always inspect the full APK contents — assets, res, every file — not just `classes.dex`.
- **"Looks harmless" is not a conclusion; it's a signal to look deeper.** When an APK looks too benign, check for hidden DEX files, hidden APK files, and downloader behavior. The harmless appearance may be intentional.
- **The downloaded payload is the real malware.** Capture it from the network or from the device. It's a standalone file that can be analyzed independently.
- **Reflection is a red flag.** If the app uses reflection to invoke a class that isn't visible in the standard decompilation (like `yin.Chao`), that class is likely in a hidden DEX or loaded dynamically. Follow the reflection.

---

## 7. DexGuard-Protected Malware — Public Walkthroughs and the Honest Assessment

### 7.1 The State of Public DexGuard Walkthroughs

DexGuard is actively maintained by Guardsquare, and the company actively researches and counters reverse-engineering techniques. As a result, **there is no single canonical public walkthrough of a DexGuard-protected malware sample** that covers the full chain from APK → anti-analysis bypass → decrypted DEX → payload analysis in the level of detail that a researcher would want to replicate.

The public material on DexGuard comes from several sources, each with different depth and focus:

- **Guardsquare's own content:** Guardsquare publishes on DexGuard features, countermeasures, and the protector's capabilities. This content is from the defender's perspective — it describes what DexGuard does and how it's configured, not how to unpack a specific sample. It's useful for understanding what you're up against, but it's not a walkthrough.
- **Vendor analyses of DexGuard-protected malware:** Some vendor blogs (NowSecure, Oversecured, ReversingLabs, Lookout, Zimperium, Kaspersky Securelist, Check Point, Trend Micro) have analyzed malware samples that happen to use DexGuard or similar protectors. These analyses sometimes describe the unpacking approach at a high level ("the DEX was decrypted using Frida hooks on the class loader" or "the native loader was analyzed to extract the decrypted DEX"), but they rarely document the full failure-and-recovery narrative that a walkthrough requires. The focus is typically on the malware's behavior, not on the unpacking process.
- **Researcher blogs and talks:** Individual Android RE researchers sometimes publish detailed writeups of interesting samples. A DexGuard-protected sample that was particularly challenging or that revealed novel techniques might get a detailed writeup. These are valuable but sporadic — there's no guarantee that a given DexGuard version or configuration has a published walkthrough.
- **GitHub:** Repos with unpacking scripts, Frida hooks, and writeups exist, but they're often version-specific or partial. A DexGuard-unpacking Frida script on GitHub may work on one version and fail on the next.

### 7.2 Why Per-Sample Analysis Is the Norm for DexGuard

DexGuard is not a monolithic protector with a single decryption mechanism. It's a configurable product where the customer (the app developer) selects which layers to enable:

- Class encryption (entire DEX files encrypted, or individual classes encrypted).
- String encryption.
- Control flow obfuscation.
- Anti-tamper (signature verification, checksums, runtime integrity checks).
- Anti-debug (ptrace, TracerPid, timing checks).
- Anti-Frida (maps scanning, library checks, process checks).
- Root detection, emulator detection.
- SSL/TLS pinning.
- Native library protection (anti-analysis native code).
- Virtualization (sensitive methods translated to custom bytecode).

Two apps protected by DexGuard can have different configurations — different layers enabled, different methods virtualized, different anti-analysis checks included. A Frida hook that works on one DexGuard-protected app may not work on another because the decryption happens at a different point, through a different API, or is blocked by different anti-analysis.

Additionally, Guardsquare updates DexGuard regularly. A technique that worked on DexGuard v7 may not work on DexGuard v8. Public scripts and tools become stale and need updating.

The practical consequence: **there is no universal DexGuard unpacker.** Each DexGuard-protected sample requires some amount of analysis, even if it's just identifying the right hook point. The general approaches (Frida hooks on class loading, BlackDex, native loader hooks, memory dumping, static analysis of the loader stub) cover the common cases, but they're not guaranteed, and the failure-and-recovery loop is often sample-specific.

### 7.3 What Public Material Does Exist

- **The protector bypass methodology in `android-commercial-protector-bypass.md`** covers the general DexGuard bypass approaches — Frida hooks, BlackDex, native loader hooks, memory dumping, string decryption hooks, static analysis of the loader stub, patching anti-analysis, and the key limitations.
- **The Packed Unpacker case study (Section 3 above)** is the closest thing to a meta-analysis of a well-protected native layer, and it applies conceptually to DexGuard-protected apps that include anti-analysis native libraries.
- **The tool sequence in Section 10 below** is the practical workflow that researchers use when they encounter DexGuard-protected samples.
- **The deobfuscation section in the Android Security Wizard skill (Section 7)** covers the obfuscation tiers and the detection/bypass approach, with DexGuard as tier 3.

### 7.4 What a DexGuard Walkthrough Would Look Like (Hypothetical Reconstruction)

If a researcher were to publish a full DexGuard walkthrough today, it would likely follow this shape, based on the known techniques and failure modes:

1. **Sample identification:** DexGuard detected via APKiD (encrypted secondary DEX files, `dexguard*.so` native library, custom Application class). Note the DexGuard version clues from library names, DEX container structure, or manifest entries.
2. **Initial static analysis:** The visible DEX contains only the loader stub — a small set of classes with the custom Application class, the DexGuard initialization, and possibly a minimal set of classes that aren't encrypted. The real app logic is in the encrypted secondary DEX files.
3. **First attempt — BlackDex:** Try BlackDex for automated extraction. If it succeeds, proceed to payload analysis. If it fails (common with newer DexGuard versions, or with specific configurations), proceed to manual approaches.
4. **Second attempt — Frida hooks on class loading:** Write a Frida script that hooks `DexClassLoader.loadClass`, `PathClassLoader.loadClass`, or the `DexFile` constructor to capture decrypted DEX bytes as they are loaded. If anti-Frida blocks this, bypass the anti-Frida first (patch the checks, use stealth options, or use boot-time instrumentation).
5. **Third attempt — native loader hook:** If Java-level hooks are blocked or don't capture the DEX, analyze the `dexguard*.so` native library with Ghidra/IDA to find the decryption function. Hook the native decryption function or the JNI calls that feed decrypted bytes back to the Java layer.
6. **Fourth attempt — memory dumping:** If hooks are fully blocked, dump process memory at the right moment and search for DEX magic bytes. This is less reliable but can work as a fallback.
7. **Fifth attempt — static patching:** If dynamic approaches all fail, patch the loader stub (smali or native) to dump decrypted DEX to a file, disable anti-analysis, or simplify the decryption. Rebuild, sign, and run in a controlled environment.
8. **Payload analysis:** Once decrypted DEX is captured, analyze it with JADX/JEB. Apply deobfuscation tools (simplify, Katalina, TinySmaliEmulator) if the decrypted DEX is still obfuscated. Analyze the malware's capabilities.
9. **Documentation:** Document the full chain — what DexGuard features were detected, what approaches were tried, what worked, what didn't, what the payload did.

This is the expected shape, but the specifics — which approach works, which anti-analysis blocks which step, what the payload is — are sample-specific.

### 7.5 Lessons

- **DexGuard is a configurable product, not a fixed target.** The approach that works depends on the specific configuration of the specific sample.
- **No universal unpacker exists, and the defender actively updates.** Expect to do some amount of per-sample analysis even if you use automated tools first.
- **The loader stub is the key.** Whether you extract dynamically or statically, understanding the DexGuard loader stub (the custom Application class and/or the `dexguard*.so` native library) is often necessary. Static analysis of the stub informs the dynamic approach.
- **Anti-analysis is the first hurdle.** DexGuard's anti-debug, anti-Frida, and integrity checks must be bypassed before extraction. These checks are concentrated in the loader stub and native libraries.
- **The decrypted DEX may still be obfuscated.** DexGuard doesn't just encrypt the DEX — it also obfuscates the code inside it (string encryption, control flow flattening, virtualization). After extraction, further deobfuscation may be needed.

---

## 8. Bangcle/Liuling — Chinese RE Community Walkthroughs

### 8.1 The Chinese RE Community and Its Coverage

The Chinese reverse-engineering community (Zhihu, CSDN, and related forums) has produced extensive writeups on Bangcle and Liuling — two of the dominant Android protectors in the Chinese ecosystem. This body of work covers:

- **File format analyses** of encrypted DEX containers — the structure of the encrypted DEX, the headers, the magic values, the layout of encrypted and plain-text sections.
- **Native library function analyses** — the decryption routines in the `.so` files, the anti-analysis checks (ptrace, TracerPid, root checks, emulator checks, Frida detection), and the control flow of the loader stub.
- **Frida scripts and hook approaches** for specific Bangcle/Liuling variants — targeting the decryption function, the class loading path, or the native loader.
- **Unpacker scripts** — version-specific scripts that automate the extraction for a particular Bangcle or Liuling version. These are often published on GitHub or in forum posts.

This literature is valuable because Bangcle and Liuling are less thoroughly covered in English-language tooling and documentation than DexGuard, and the Chinese RE community has been analyzing them longer and in more depth.

### 8.2 The Language Barrier

A significant practical issue: **most of this writeup material is in Chinese.** Researchers who don't read Chinese face a language barrier when searching for Bangcle/Liuling-specific techniques. Options for bridging this include:

- **Machine translation** of Zhihu posts, CSDN articles, and GitHub README files. Technical content generally translates reasonably well, though some nuance may be lost.
- **Searching for English-language coverage** of the same techniques. The same unpacking approaches (Frida hooks on class loading, native loader hooking, memory dumping) are protector-agnostic and appear in English-language writeups as well.
- **Looking for code over prose.** Frida scripts, Python unpacker scripts, and Ghidra/IDA analysis scripts on GitHub may be version-specific but are language-neutral — the code itself is the technique, regardless of the surrounding documentation language.
- **Searching GitHub for Bangcle/Liuling unpacker repos.** Some repos have English READMEs even if the original forum discussion was in Chinese.

### 8.3 Technique Overlap with the General Approach

The Chinese RE community's Bangcle/Liuling techniques overlap heavily with the general dynamic extraction approach:

- **Hooking the decryption/loading path.** Frida hooks on `DexClassLoader`/`PathClassLoader` or on the native decryption function. This is the same approach described for DexGuard and other protectors.
- **Dumping decrypted DEX from memory.** Capturing the decrypted DEX at runtime, either via hooks or via memory dumps.
- **Analyzing the native loader stub.** Static analysis of the `.so` that performs decryption, to understand the decryption flow and identify hook points.
- **Patching anti-analysis checks.** When dynamic analysis is blocked by anti-debug or anti-Frida, patching the checks in the loader stub (smali or native) to enable dynamic extraction.

What the Chinese RE community adds is depth on the **file format** and the **specific native library structures** of Bangcle and Liuling — knowledge that helps identify the protector, identify the decryption function, and understand what the encrypted DEX looks like before and after decryption.

### 8.4 What a Bangcle/Liuling Walkthrough Typically Looks Like

A typical walkthrough from the Chinese RE community (translated) would follow this shape:

1. **Identify the protector:** APKiD or manual inspection identifies Bangcle or Liuling. File structure fingerprints (encrypted DEX container format, native library names, manifest entries) confirm the identification.
2. **Analyze the native library:** Load the `.so` in Ghidra/IDA. Identify the decryption function, the anti-analysis checks, and the control flow of the loader stub. Understand where the encrypted data comes from, how the key is derived, and where the decrypted DEX goes.
3. **Attempt dynamic extraction:** Use Frida to hook the decryption function or the class loading path. Dump the decrypted DEX as it's loaded. If anti-Frida blocks this, patch the anti-Frida checks or use stealth options.
4. **Fallback to static patching:** If dynamic is blocked and can't be bypassed, patch the native loader stub to dump decrypted DEX to a file, or to disable anti-analysis. Rebuild and run.
5. **Analyze the decrypted payload:** Once the DEX is extracted, analyze it with standard tools. The payload may be a legitimate app (Bangcle/Liuling are used by legitimate apps too) or malware (some malware authors use Bangcle/Liuling for protection).
6. **Document version-specific findings:** Note which version of Bangcle/Liuling, which specific functions were hooked, which anti-analysis checks were present, and what the decrypted DEX contained.

### 8.5 Lessons

- **The Chinese RE community has deep Bangcle/Liuling knowledge.** Searching that literature is productive when English-language sources are insufficient, even with the language barrier.
- **The techniques overlap with the general approach.** The same dynamic extraction, native analysis, and anti-analysis bypass techniques apply. The community-specific knowledge is primarily about file formats and native library structures.
- **Version-specific scripts exist but are fragile.** Public unpacker scripts target specific Bangcle/Liuling versions and break when the protector updates. They're useful starting points but not universal solutions.
- **Malware using Bangcle/Liuling is analyzed with the same approach as legitimate apps.** The protector doesn't care whether the payload is benign or malicious. The analysis approach is identical.

---

## 9. What Typically Fails — The Failure-and-Recovery Narrative

This section catalogs the common failure modes researchers hit during commercial protector unpacking, and the recovery approaches. These are the obstacles that show up repeatedly across samples and protectors.

### 9.1 Anti-Debug Blocks Frida

**The failure:** The app detects the debugger (Frida attaches as a debugger) and refuses to run, crashes, or behaves differently. The Frida script never gets a chance to run, or runs but the app exits before any useful extraction.

**Why it happens:** Commercial protectors include anti-debug checks — `ptrace` self-attachment to block other debuggers, `TracerPid` checks in `/proc/self/status`, `isDebuggerConnected()` checks, timing-based detection. These checks are often in the loader stub and run early in the app lifecycle.

**Recovery approaches:**

- **Frida stealth options:** Use Frida's `--no-pause`, `--memory`, and other options to reduce the debugger footprint. Some protectors detect the default Frida attachment pattern; stealth options can evade basic detection.
- **Patching the anti-debug checks:** Analyze the loader stub (smali or native) to find the anti-debug checks, then patch them (NOP, early return). This requires static analysis of the stub but enables dynamic extraction afterward.
- **Boot-time instrumentation:** Use Frida gadget mode (repackage the APK with `libfrida-gadget.so`) so the instrumentation is loaded before the anti-debug checks run. The gadget initializes early enough to hook the checks before they fire.
- **Work statically on the loader:** If dynamic is fully blocked, analyze the loader stub statically (Ghidra/IDA for native, Smali for Java) to understand the decryption flow without running the app. This is slower but doesn't require bypassing anti-debug.
- **Run on a non-rooted, non-debuggable device:** Some anti-debug checks only fire when the device is rooted or the app is debuggable. Running on a clean emulator without root or without debuggable flags can bypass some checks.

### 9.2 Anti-Tamper Crashes on Patching

**The failure:** You patch the APK (to dump decrypted DEX, disable anti-analysis, or simplify the decryption) and rebuild it, but the rebuilt APK crashes on launch or sabotages itself. The patching triggered an integrity check.

**Why it happens:** Commercial protectors include anti-tamper checks — signature verification (checking that the APK is signed by the expected key), checksums on critical files (DEX, native libs, manifest), runtime integrity checks that verify the app hasn't been modified. When you patch the APK, you break these checks.

**Recovery approaches:**

- **Don't patch the APK; use dynamic instrumentation instead.** Run the original, unmodified APK and intercept at runtime with Frida. The APK's integrity is preserved, so integrity checks pass.
- **Patch the integrity checks instead of the APK.** If you must patch (because dynamic is blocked), patch the integrity check functions to always return "clean" rather than patching the DEX or native libs.
- **Re-sign the APK with the original signing key** (if you have it — rarely the case for malware samples) or with a key that the app accepts. Some integrity checks verify the signature against a specific key; if you can provide that key, the check passes. This is uncommon for malware analysis but possible for legitimate apps where you have the source.
- **Memory patching vs. APK patching.** Patch at runtime in memory (via Frida) rather than patching the APK file. Memory patches don't change the file on disk, so file-based integrity checks pass. Runtime integrity checks that re-verify memory content are harder to bypass.

### 9.3 Integrity Checks Detect the Unpacker

**The failure:** The protector verifies the APK signature or file integrity at runtime and detects that it's been tampered with (or that the app is running in an analysis environment). The app crashes, exits, or refuses to decrypt.

**Why it happens:** Integrity checks are a standard anti-tamper layer. They verify that the APK is the original, unmodified version. When you run the APK through an unpacker (which may modify the APK or run it in a modified environment), the checks detect the modification.

**Recovery approaches:**

- **Run the original APK without modification.** Use dynamic instrumentation (Frida) on the unmodified APK. The file integrity is preserved.
- **Bypass the integrity check functions.** Patch or hook the functions that perform the integrity check to return success regardless of the actual state.
- **Use Frida gadget mode.** The gadget is loaded into the original APK without modifying the APK's file structure (just adding a native library). Some integrity checks may not flag this, depending on what they check.
- **Understand what the integrity check verifies.** Is it the APK signature? A checksum on a specific file? A runtime check on decrypted content? Each type has a different bypass path. Static analysis of the check function tells you what it verifies and how to bypass it.

### 9.4 Decryption Timing — Too Early or Too Late to Hook

**The failure:** The DEX is decrypted at a point in the app lifecycle where your Frida script can't intercept it. Either the decryption happens before Frida attaches (early decryption), or the decryption window is so narrow that the hook misses it.

**Why it happens:** Some protectors decrypt the DEX very early — during the custom Application class's `attachBaseContext` or `onCreate`, before the Frida gadget is fully initialized or before a debugger can attach. Others decrypt on-demand (per-class-load) with a narrow window between decryption and the next protection step (re-encryption, checksum verification).

**Recovery approaches:**

- **Earlier attachment.** Use Frida spawn mode (`frida -f`) to attach as early as possible in the process lifecycle. The spawn mode starts the process and attaches before `main`/equivalent, which can be early enough to catch early decryption.
- **Frida gadget mode.** The gadget initializes early in the app lifecycle (loaded as a native library by the APK's loader), which can be earlier than a dynamically attached Frida client.
- **Hook the loader/constructor, not the load point.** Instead of hooking the point where the DEX is loaded (which may be after decryption), hook the decryption function itself (in the native library or the Java loader stub) or the constructor that initializes the decryption. This captures the decrypted data at the source.
- **Capture at multiple points.** If the DEX is decrypted and then re-encrypted between loads, capture it at every point where it's in memory in decrypted form. Multiple hooks, multiple dumps, and then compare/correlate.
- **Static analysis to understand the timing.** Analyze the loader stub to understand exactly when decryption happens, what triggers it, and what the window is. This informs where to place the hook.

### 9.5 DEX Re-Encrypted Between Loads

**The failure:** You capture the decrypted DEX, but it's only valid for one class load. After that load, the protector re-encrypts it. The captured DEX works for analysis of the loaded class but you can't get the full DEX — only the classes that were loaded during your capture window.

**Why it happens:** Some protectors decrypt DEX on-demand — decrypt a class, load it, then re-encrypt it. This reduces the exposure window and makes capture harder. The decrypted DEX is in memory only briefly and only for the classes being loaded.

**Recovery approaches:**

- **Capture during the window.** Hook at the exact point where the DEX is decrypted and in memory, before it's re-encrypted. This may require precise timing or hooking the decryption function to capture the output immediately.
- **Capture multiple times.** If different classes are decrypted at different times, capture at each decryption event. Over multiple runs or with a comprehensive hook, you may collect all the decrypted classes.
- **Hook the decryption function, not the load point.** The decryption function produces the decrypted bytes. If you hook that function and capture its output every time it's called, you get every decrypted class regardless of when it's loaded.
- **Understand the re-encryption trigger.** What causes the re-encryption? Is it time-based, load-based, or check-based? Understanding the trigger lets you predict when the window is open.

### 9.6 Native Loader Is Anti-Analysis Hardened

**The failure:** The native library that performs the decryption (and/or the anti-analysis) is itself heavily protected — anti-debug, anti-Frida, control flow flattening, string encryption, junk code. You can't hook it dynamically because it detects the instrumentation, and you can't analyze it statically because the obfuscation is too dense.

**Why it happens:** Well-protected apps invest in hardening the native loader. This is the "Packed Unpacker" pattern (Section 3) — the protector's own code is the thing you need to analyze and bypass, and it's designed to resist exactly that.

**Recovery approaches:**

- **Static analysis of the native library with Ghidra/IDA.** Even with obfuscation, static analysis can reveal structure — function boundaries, string references, control flow patterns, calls to known APIs (decryption APIs, class loading APIs, anti-analysis APIs). The goal is to identify the decryption function and the anti-analysis checks, even if the full logic is obscured.
- **Dynamic analysis with stronger bypass.** If the native library detects Frida, use stronger Frida stealth, boot-time instrumentation, or patch the native anti-analysis checks. This is the recursive problem — you may need to bypass the native anti-analysis to analyze the native library that contains the anti-analysis.
- **Memory dump of the unpacked native library.** If the native library unpacks itself at runtime (the "Packed Unpacker" pattern), dump the process memory after unpacking to capture the unpacked native library. Analyze the unpacked version statically — it's easier to read than the packed version.
- **Peel the layers one at a time.** First, bypass the outer anti-analysis to enable dynamic analysis. Then, analyze the native library to find the decryption function. Then, hook the decryption function to extract the DEX. Each layer requires its own approach.

### 9.7 Multiple Protectors Stacked

**The failure:** The APK uses more than one protector — e.g., DexGuard for DEX encryption plus a separate native protector for the native library plus additional obfuscation (string encryption, control flow flattening). The approach that works for one layer doesn't work for another, and the layers interact in unexpected ways.

**Why it happens:** Some apps use multiple protection products (or one product with multiple layers enabled) to defense in depth. The layers may be from different vendors, may protect different artifacts (DEX vs. native vs. resources), and may have different anti-analysis profiles.

**Recovery approaches:**

- **Peel one layer at a time.** First, get the decrypted DEX (which may require bypassing the DEX protector's anti-analysis). The decrypted DEX may reveal the logic for the next layer (e.g., the native library's interface, the string decryption routine, the resource decryption path). Then, tackle the next layer.
- **Identify each layer independently.** APKiD may flag multiple protectors. Manual inspection of the APK structure (multiple encrypted DEX files, multiple native libraries with different anti-analysis patterns, multiple obfuscation signatures) helps identify what layers are present.
- **Don't assume one tool covers all layers.** BlackDex may extract the DEX but not bypass the native protector's anti-analysis. Frida hooks on class loading may capture the DEX but not the strings (which are still encrypted in the DEX). Be ready to use different tools for different layers.
- **The decrypted DEX may be the key to the next layer.** Once you have the decrypted DEX, you can analyze the Java-level logic for how the native library is called, what it expects, what it returns. This informs the approach for the native layer.

---

## 10. Tool Sequence in Real Walkthroughs

This is the typical sequence researchers use when unpacking a commercial-protected sample. It's not a rigid checklist — the sequence varies by sample, protector, and anti-analysis profile. The key is knowing the full toolbox and being ready to pivot.

### 10.1 Step 1 — apktool (Decode Resources and Smali)

`apktool d app.apk -o decoded/`

Decode the APK to get the manifest, resources, and Smali. This is the first step because it gives you the full file listing (every file in the APK, including assets, native libraries, and auxiliary DEX files) and the Smali code for the visible DEX.

What you're looking for at this step:

- **Manifest:** custom `Application` class, exported components, permissions, protector-specific entries.
- **File listing:** encrypted secondary DEX files (high entropy, invalid headers), native libraries (`.so` files), files in `assets/` that could be hidden DEX/JAR/APK, encrypted resource containers.
- **Smali:** the loader stub (custom Application class, decryption initialization), references to `DexClassLoader`, `PathClassLoader`, `loadClass`, reflection, and any suspicious API usage.

### 10.2 Step 2 — JADX / JEB (Decompile What's Visible)

`jadx-gui app.apk` or `jadx -d out_java app.apk`

Decompile the visible DEX to Java (or use JEB for a more powerful commercial decompiler). This gives you readable Java for the parts of the app that aren't encrypted.

What you're looking for:

- **The loader stub logic:** how the custom Application class initializes, what it calls, what native libraries it loads, what decryption setup it performs.
- **Visible API usage:** even in the encrypted app, the loader stub and any non-encrypted classes may reference dangerous APIs (SMS, telephony, network, package installation, accessibility).
- **Class loading references:** `DexClassLoader`, `PathClassLoader`, `loadClass`, `DexFile` — these indicate dynamic code loading and point to where the hidden DEX is loaded.
- **Reflection usage:** dynamically constructed class/method names, reflection-based invocation — these may point to hidden code.

### 10.3 Step 3 — Androguard / MobSF (Programmatic Analysis)

Run MobSF for an automated static scan (manifest extraction, permissions, API calls, hardcoded secrets, certificate pinning detection, YARA hits). Use Androguard for programmatic inspection of the manifest, components, and API usage.

What you're looking for:

- **Dangerous permission combos:** SMS + READ_CONTACTS + ACCESS_FINE_LOCATION + CAMERA together, or other suspicious combinations.
- **Exported components:** activities, services, receivers, content providers with `exported=true` and no permission guards.
- **API usage patterns:** SMS sending, phone calls, location access, camera/microphone, network exfiltration, package installation.
- **Certificate pinning:** detected by MobSF — relevant if you plan to MITM the network traffic.
- **YARA hits:** known malware family signatures or IOC matches.

### 10.4 Step 4 — APKiD (Identify the Protector)

`apkid app.apk`

Run APKiD to identify the protector. This is the pivot point — knowing which protector you're dealing with informs the rest of the approach.

What you're looking for:

- **Protector identification:** APKiD flags DexGuard, Bangcle, Liuling, APKProtect, and others based on signatures, YARA rules, and ML-assisted detection.
- **Confidence level:** high-confidence hits (clear signatures) vs. low-confidence or unknown (requires manual inspection).
- **If no protector is identified:** treat the app as having an unknown or custom protector and proceed with manual inspection of the APK structure (encrypted DEX, native libraries, manifest, resources).

### 10.5 Step 5 — Identify the Loader/Decryptor

Based on the APKiD results and manual inspection, identify the component that performs the decryption:

- **Custom Application class:** the class referenced in the manifest's `android:name` attribute. This class typically initializes the protector and performs decryption setup.
- **Native library:** the `.so` file that performs decryption and/or anti-analysis. For DexGuard, this is often `dexguard*.so`. For Bangcle/Liuling, it's a characteristic native library with the decryption logic.
- **Java-level decryptor:** in some configurations, the decryption happens in Java (a custom class with a decryption method) rather than native. This is less common for strong protectors but possible for lighter configurations.

### 10.6 Step 6 — Static Analysis of the Loader

Analyze the loader stub to understand the decryption flow:

- **For Java-level loaders (Smali/Java):** trace the decryption logic — where the encrypted data comes from, how the key is derived, where the decrypted data goes, what class loading API is called with the decrypted data.
- **For native libraries (Ghidra/IDA):** analyze the `.so` to find:
  - The decryption function (the function that takes encrypted data and produces decrypted data).
  - The anti-analysis functions (ptrace, TracerPid, Frida detection, root checks, emulator checks).
  - The JNI bridge functions (the functions that receive decrypted data from native and pass it to Java, or that call Java class loading APIs).
  - The library initialization (JNI_OnLoad, constructors) — where the decryption setup happens.
  - String references and data sections — clues about what the library does and what APIs it uses.

The goal of this step is to understand *how* the decryption works well enough to target it with dynamic hooks or static patches. Even if you plan to use BlackDex or a generic Frida script first, understanding the loader gives you a fallback path when the generic approach fails.

### 10.7 Step 7 — Dynamic Setup (frida-server or Gadget)

Set up the dynamic analysis environment:

- **Rooted device/emulator:** push `frida-server` to `/data/local/tmp/`, chmod 755, run in background.
- **Non-rooted device/emulator:** use Frida gadget mode — decompile the APK with apktool, add `libfrida-gadget.so` to the APK, re-align with zipalign, re-sign, install. The gadget initializes on app launch and connects to the Frida client.
- **Consider the anti-analysis profile:** if the app has strong anti-Frida, the default frida-server deployment may be detected. Plan for stealth options, gadget mode, or boot-time instrumentation.

### 10.8 Step 8 — Frida Hooks on Class Loading / Decryption

Write and deploy Frida hooks targeting the decryption/loading path:

- **DexClassLoader/PathClassLoader hooks:** hook the constructors or `loadClass` to log and capture loaded DEX paths and (if possible) the decrypted DEX bytes.
- **DexFile constructor hooks:** hook the `DexFile` constructor to capture the DEX file path and (if the DEX is decrypted in memory before being passed to the constructor) the decrypted bytes.
- **Native decryption function hooks:** if the native library's decryption function has been identified in Step 6, hook it to capture the decrypted bytes as they're produced.
- **String decryption hooks:** if the string decryption method is identified, hook it to log decrypted strings. This is useful even without full DEX extraction.

The Frida script should:

- Intercept the function that receives or produces decrypted DEX bytes.
- Write the bytes to a file on the device or send them over the Frida connection to the host.
- Handle anti-Frida checks (by patching them first, using stealth options, or using gadget mode).

### 10.9 Step 9 — If Frida Is Blocked

If the app detects and blocks Frida (crashes, exits, or refuses to decrypt):

- **Stronger Frida stealth:** try `--no-pause`, `--memory`, gadget mode with hiding options, or a different Frida deployment (e.g., the gadget built into the APK rather than a separate frida-server process).
- **Boot-time instrumentation:** use the gadget mode so the instrumentation is loaded before the anti-Frida checks run.
- **Patch the anti-Frida checks:** use static analysis of the loader stub (from Step 6) to identify the anti-Frida checks, then patch them in the APK (smali or native). Rebuild, sign, and run the patched APK with Frida attached. The patched APK won't detect Frida, so the dynamic extraction can proceed.
- **Fallback to static-only analysis of the loader:** if dynamic is fully blocked and patching the anti-Frida is not feasible (e.g., the checks are too numerous or too well-protected), analyze the loader stub statically to understand the decryption flow. This is slower and may not yield the decrypted DEX, but it can yield enough understanding to inform other approaches or to analyze the loader's logic directly.

### 10.10 Step 10 — BlackDex or Similar (Automated Extraction)

Try BlackDex (or similar automated DEX extraction tools) as a parallel or fallback approach:

- BlackDex targets Android 5.0–12 and works by hooking the DEX loading path and dumping decrypted DEX files. It has coverage across many protectors but is not universal.
- If BlackDex succeeds, you have the decrypted DEX without writing custom Frida scripts. Proceed to payload analysis.
- If BlackDex partially succeeds (extracts some but not all DEX files, or extracts an incomplete DEX), examine what's missing and target the missing parts manually.
- If BlackDex fails entirely, proceed to manual Frida hooks or static patching.

### 10.11 Step 11 — Decrypt DEX Analysis (JADX/JEB on the Decrypted DEX)

Once decrypted DEX is captured:

- **Analyze with JADX/JEB:** decompile the decrypted DEX to Java for readable analysis.
- **If multiple DEX files were encrypted:** each must be extracted. Some protectors encrypt `classes.dex` and additional DEX files (`classes2.dex`, `classes3.dex`, etc.) separately. Each may need its own extraction.
- **If resources/assets were encrypted:** they may need separate extraction (often through the same decryption pipeline or through dedicated resource decryption hooks). Encrypted resources may contain additional code, configuration, or payload data.

### 10.12 Step 12 — If Payload Is Still Obfuscated

The decrypted DEX may still be obfuscated — string encryption, control flow flattening, virtualization, junk code. Apply deobfuscation tools:

- **simplify:** Smali deobfuscator with virtual execution, constant propagation, dead code removal, and unreflection. Useful for cleaning up the decrypted code.
- **Katalina / TinySmaliEmulator:** string deobfuscators. Useful if the decrypted DEX still has encrypted strings.
- **Deoptfuscator:** targets control-flow-obfuscated code patterns. Useful if the decrypted DEX has CFG flattening.
- **Manual Smali work:** when automated tools don't fully clean up the code, read the Smali, rename obfuscated classes/methods, trace string decryption, and reconstruct control flow manually.

### 10.13 Step 13 — Dynamic Analysis of the Payload

Once the payload is readable (or even while it's still partially obfuscated), do dynamic analysis of its behavior:

- **Frida hooks on the payload's behavior:** network (HTTP, OkHttp, Socket), SMS (SmsManager), telephony (PhoneAccount, CallManager), crypto (KeyStore, KeyGenerator, Cipher), accessibility (AccessibilityService, AccessibilityManager), package installation (PackageInstaller, Intent), file I/O (File, FileOutputStream), and any other suspicious APIs.
- **Network capture:** MITM the payload's traffic (with SSL pinning bypass if needed) to capture C2 communications, downloaded payloads, and exfiltrated data.
- **Behavioral observation:** what does the payload do when it runs? What APIs does it call? What data does it collect? Where does it send it?

### 10.14 Step 14 — Document the Full Chain

Document everything:

- **What was protected:** which protector, which layers, which artifacts were encrypted (DEX, strings, resources, native libs).
- **What protector:** APKiD identification, manual confirmation, version clues.
- **How it was unpacked:** which tools and techniques were used, in what order, what the hook points were, what was captured.
- **What worked and what didn't:** the failure-and-recovery narrative.
- **What the malware did:** the payload analysis — capabilities, C2, behaviors, IOC.
- **Lessons learned:** what you'd do differently next time.

This documentation is what turns your analysis into a walkthrough that helps the next researcher.

### 10.15 The Pivot Points

The sequence is not linear. The key pivot points where researchers switch approaches:

| Pivot | From | To | Trigger |
|-------|------|----|---------|
| Automated → Manual | BlackDex / generic Frida script | Custom Frida hooks, static loader analysis | Automated tool fails or partially succeeds |
| Dynamic → Static | Frida hooks | Ghidra/IDA on native, Smali analysis of loader | Anti-Frida / anti-debug blocks dynamic |
| Java → Native | Java-level hooks | Native library hooks (Ghidra/IDA + Frida native hooks) | Decryption is in native code, Java hooks miss it |
| Hooking → Patching | Frida dynamic hooks | APK patching (smali/native) | Dynamic is blocked and anti-analysis can be patched |
| DEX extraction → String extraction | Full DEX dump | String decryption hooks only | Full DEX extraction fails but string decryption is hookable |
| Static → Dynamic | Static analysis of loader | Frida hooks, dynamic observation | Static analysis gives enough understanding to target dynamic hooks |
| Single capture → Multiple captures | One dump | Multiple dumps at different points | DEX is re-encrypted between loads, or different classes decrypt at different times |

The researcher who knows these pivot points — and knows when to take them — is the researcher who successfully unpacks samples that resist the first attempt.

---

## 11. Where to Find More Walkthroughs

### 11.1 Conference Materials (Black Hat, DEF CON, Recon)

Conference talks are a primary source of detailed walkthroughs. Search for:

- **Black Hat USA / Europe / Asia** — Android unpacking, protector bypass, anti-analysis, malware analysis talks. Maddie Stone's "Packed Unpacker" is the standout example. Other years have had talks on DexGuard bypass, Bangcle analysis, malware unpacking, and Frida-based extraction.
- **DEF CON** — Android security, reverse-engineering, and malware analysis talks. DEF CON villages (IoT, mobile) often have relevant content.
- **Recon** — Recon is a security conference with a strong reverse-engineering focus. Android unpacking and protector analysis talks appear here.
- **Virus Bulletin** — Academic/industry conference with papers on Android malware and unpacking (e.g., the Packed Unpacker paper).

Talk materials typically include slides, whitepapers, and sometimes recordings. Slides often contain the detailed methodology and failure-and-recovery narrative that written reports may omit.

### 11.2 Researcher Blogs

Individual Android RE researchers publish detailed writeups of interesting samples on their blogs. These are often the most detailed public walkthroughs available — a researcher who solved a challenging sample may publish the full narrative.

Search approaches:

- Search for the specific malware family or protector name + "walkthrough," "analysis," "unpacking," "reverse engineering."
- Follow known Android RE researchers and security vendors who publish detailed sample analyses.
- Look for writeups that go beyond "here's what the malware does" to include "here's how we extracted the decrypted DEX, here's what blocked us, here's how we got around it."

### 11.3 Vendor Blogs

Security vendors publish detailed analyses of malware samples, often including the unpacking approach:

- **NowSecure:** Android security research, including protector analysis and malware unpacking.
- **Guardsquare:** DexGuard feature documentation and countermeasure content (defender's perspective, but useful for understanding what you're up against).
- **Oversecured:** Android vulnerability analyses, including some protector-related content and deep dives into specific app vulnerabilities.
- **ReversingLabs:** Malware analysis, including Android samples and unpacking approaches.
- **Lookout:** Mobile malware research, including BRATA and other dropper families.
- **Zimperium:** Mobile threat research, including banking trojans and subscription fraud families (Joker, etc.).
- **Kaspersky Securelist:** Detailed mobile threat reports, including Android malware analyses with sample-level detail (Ghimob, banking trojans, droppers). Kaspersky's reports often include the delivery mechanism and the payload analysis.
- **Check Point Research:** Advanced malware analysis, including JSCeal (V8 bytecode deobfuscation) and other complex samples. Check Point's research often goes deep on the deobfuscation/unpacking methodology.
- **Trend Micro:** Mobile malware research, including multi-stage infection chains and downloader families.

Vendor blogs vary in how much unpacking detail they include. Some focus on the malware's behavior and IOCs; others include the extraction methodology. Look for the ones that document the process, not just the result.

### 11.4 GitHub

GitHub hosts a variety of unpacking-related content:

- **Unpacking scripts:** Frida scripts for DEX extraction, native library hooking, anti-analysis bypass. These are often version-specific but demonstrate the technique.
- **Writeups and documentation:** Some repos include detailed writeups of specific samples or protectors.
- **Unpacker tools:** BlackDex, simplify, Katalina, TinySmaliEmulator, Deoptfuscator, and other tools. Reading the tool's documentation and issues can reveal what protectors it covers, what failure modes are common, and what the community has learned about using it.
- **Sample analyses:** Some researchers publish their analysis of specific samples as GitHub repos, including the Frida scripts they wrote, the decrypted DEX they extracted, and the documentation of the process.

Search for the specific protector name + "unpack," "Frida," "hook," "extract," or the malware family name + "analysis," "walkthrough."

### 11.5 Chinese RE Community (Zhihu, CSDN)

The Chinese RE community has extensive writeups on Bangcle, Liuling, and other Chinese protectors, as covered in Section 8. These writeups include file format analyses, native library function analyses, Frida scripts, and unpacker scripts.

Search approaches:

- **Zhihu:** Search for Bangcle, Liuling, APK unpacking, DEX extraction. Zhihu posts often include detailed technical writeups with code snippets.
- **CSDN:** Search for the same terms. CSDN articles often include step-by-step walkthroughs with code.
- **Language barrier:** Most content is in Chinese. Use machine translation for technical content. Look for code (Frida scripts, Python scripts, Ghidra/IDA annotations) which is language-neutral. Search for English-language coverage of the same techniques as a complement.

### 11.6 The Feedback Loop

Walkthroughs propagate techniques. When a researcher publishes a detailed walkthrough of a challenging sample, the next researcher who encounters a similar sample benefits from the published approach. This is how the craft advances — not through tooling lists, but through documented narratives of what worked, what didn't, and what the researcher did when it didn't work.

When you publish your own walkthrough (following the template in Section 2), you contribute to this feedback loop. The details that feel obvious to you after the analysis — the specific hook point, the specific anti-analysis check that blocked you, the specific recovery that worked — are exactly the details that help the next researcher who encounters the same protector or the same failure mode.

---

## 12. Bottom Line

Walkthroughs teach the craft. They show you how researchers actually unpack real samples — not just what tools exist, but how the tools are used in sequence, what fails, what the researcher does when it fails, and how they chain approaches to get from an encrypted APK to a understood payload.

The Packed Unpacker case study is the standout meta-problem: unpacking the anti-analysis layer to understand the anti-analysis layer. It illustrates why some samples require significant manual effort even when the general approach (dynamic extraction) is known — because the protector's own code is the first thing you have to defeat, and that code is designed to prevent you from defeating it.

Two-stage droppers (BRATA, Ghimob) teach that static analysis of the APK is often just the first stage. The dropper is the delivery mechanism; the real payload is downloaded or loaded at runtime and must be captured dynamically — from the network, from the filesystem, or from memory. Stopping at the dropper means missing the malware.

Joker teaches that multi-stage malware requires multi-stage analysis. The APK → hidden DEX → downloaded JAR chain means each stage must be captured and analyzed separately. Static analysis alone gives you the first stage; dynamic analysis and network capture give you the later stages.

DexGuard and Bangcle/Liuling teach that per-sample analysis is the norm for actively defended protectors. No universal unpacker exists. DexGuard is updated by Guardsquare to counter known bypass techniques; Bangcle and Liuling are updated by their maintainers. Each sample requires some amount of analysis — identifying the right hook point, bypassing the specific anti-analysis checks, adapting to the specific configuration. Public scripts and tools are starting points, not guarantees.

The failure-and-recovery narrative (Section 9) is the core practical content. Anti-debug blocks Frida → patch the checks or use stronger stealth. Anti-tamper crashes on patching → don't patch the APK, use dynamic instrumentation. Integrity checks detect the unpacker → run the original APK. Decryption is too early or too late → attach earlier or hook the decryption function. DEX is re-encrypted between loads → capture at every decryption event. Native loader is anti-analysis hardened → analyze it statically, dump the unpacked version, peel layers one at a time. Multiple protectors stacked → peel one layer at a time.

The tool sequence (Section 10) is the practical workflow. apktool → JADX/JEB → Androguard/MobSF → APKiD → loader identification → static analysis of the loader → Frida dynamic hooks → BlackDex → decrypted DEX analysis → payload analysis → documentation. The sequence varies by sample; the key is knowing the full toolbox and the pivot points where you switch approaches.

The sources are everywhere — Black Hat/DEF CON/Recon talks, researcher blogs, vendor blogs (NowSecure, GuardSquare, Oversecured, ReversingLabs, Lookout, Zimperium, Kaspersky Securelist, Check Point, Trend Micro), GitHub, and the Chinese RE community (Zhihu/CSDN). The best walkthroughs are the ones that document the full chain — what was protected, what protector, how it was unpacked, what worked, what didn't, what the payload did, and what the researcher learned.

---

*This document is the companion to `android-commercial-protector-bypass.md` (the protector landscape and bypass approaches catalog) and `android-re-frida-pipeline.md` (the RE/Frida tooling survey). Together, the three documents cover the Android commercial protector analysis craft: what the protectors are, how to bypass them, and what the actual unpacking of real samples looks like.*
