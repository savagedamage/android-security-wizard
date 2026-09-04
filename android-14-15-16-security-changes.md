# Android 14/15/16 Security Model Changes & Attack-Surface Evolution

*Synthesized from developer.android.com version summary pages (Android 14/15/16), Oversecured Samsung preinstalled app vuln research, and OWASP MASTG testing methodology. Written 2026-09-04.*

*Sources: https://developer.android.com/about/versions/15/summary, https://developer.android.com/about/versions/16/summary, https://developer.android.com/about/versions/17/behavior-changes-17, https://oversecured.com/blog/176-vulnerabilities-in-samsung-preinstalled-apps*

## 1. Android 14 — Security changes by feature area

### Scoped storage (continuation/enforcement)

Android 13 introduced scoped storage; Android 14 continues enforcing it more strictly. External storage access is more restricted. Apps that previously relied on broad file-system access to shared storage must use the MediaStore APIs, Storage Access Framework, or app-specific directories.

Attack-surface impact: Malware that previously exfiltrated files from shared storage or other apps' public directories has fewer direct paths. Forensics and analysis tools that rely on broad storage access may lose visibility unless they have appropriate permissions or use the Storage Access Framework.

### Restricted permissions — runtime model tightening

Android 14 continues the runtime-permission tightening from Android 13:

- **POST_NOTIFICATIONS** is a runtime permission (introduced Android 13, enforced in 14). Apps that send notifications without the permission are silently blocked.
- **Exact alarms** (SCHEDULE_EXACT_ALARM / USE_EXACT_ALARM) require explicit justification. Android 14 makes it harder to obtain and use exact alarms without clear need.
- More granular permission groups; some permissions that were implicit or one-time grants become more explicitly controlled.

Attack-surface impact: Malware that abuses notifications (push fraud, disguised alerts) or exact alarms (scheduled tasks, timer-based callbacks) faces additional hurdles. Analysts testing notification abuse must respect the runtime permission model.

### Photo picker and photo metadata privacy

Android 14 advances the photo picker as a privacy-preserving alternative to broad READ_MEDIA_IMAGES / READ_MEDIA_VIDEO / READ_MEDIA_AUDIO access. Photo metadata (EXIF) access is limited to protect location privacy — apps get reduced or no location metadata by default unless they have strong justification.

Attack-surface impact: Spyware that relied on reading full-resolution photos with embedded location metadata has a weaker default path. Analysts hunting for photo exfiltration need to check whether the app uses the photo picker, MediaStore, or attempted broad storage access.

### Foreground service types

Android 14 requires apps to declare foreground service types. If an app starts a foreground service without declaring the appropriate type, the app may crash or be restricted. This makes it harder for apps to run background services under the guise of foreground services without declaring their purpose.

Attack-surface impact: Malware that uses foreground services to maintain persistence or hide background activity must now declare a type that matches its behavior. Analysts can use the declared type as a signal — a mismatch between declared type and actual behavior is suspicious.

### Pending intent mutability (FLAG_IMMUTABLE default)

Android 14 moves to FLAG_IMMUTABLE as the default for pending intents. This reduces the attack surface for intent-mutability-based abuse, where an attacker modifies a pending intent after creation to change its target or extras.

Attack-surface impact: A class of intent injection and pending-intent manipulation attacks becomes harder. If an app relies on mutable pending intents, it must explicitly opt in — which is a signal worth auditing.

### Background activity starts — further restrictions

Android 14 further restricts background activity starts. Apps that try to start activities from the background without an appropriate exemption are blocked or limited.

Attack-surface impact: Apps that use background activity launches for UI-based fraud (overlays, fake login screens, phishing-style flows) face additional constraints. Analysts checking for background UI abuse need to verify whether the activity start is allowed under Android 14 rules.

### Exported component enforcement

Android 14 enforces explicit `android:exported` declaration for components that have intent filters. Components with intent filters that are meant to be internal must be explicitly marked `exported="false"`.

Attack-surface impact: This reduces a class of accidental exposure where a component with an intent filter was left unexported-by-omission but actually reachable or misread. It also makes the exported/non-exported boundary clearer for pentesters and malware analysts.

### Non-SDK interface restrictions (hidden API evolution)

Android 14 continues the trend of restricting greylist/blacklist access to non-SDK interfaces. Apps using hidden APIs get warnings and, in more cases, errors. The reachable hidden API surface shrinks over time.

Attack-surface impact: Malware and advanced analysis tooling that relied on hidden APIs for deeper access, reflection-based hooks, or framework internals faces a shrinking toolkit. Analysts who use non-SDK interfaces for instrumentation or forensics may need to adapt to supported APIs or accept reduced visibility.

### WebView, Bluetooth, and other framework areas

- **WebView**: Updated security defaults, mixed-content handling, Safe Browsing integration. Reduces some MITM and content-injection attack surface in apps that embed WebView.
- **Bluetooth**: BLUETOOTH_SCAN and related permissions have location implications and more granular control. Apps that use Bluetooth scanning must be more explicit about location usage.
- **Security by Default**: Android 14 increases the default security posture for apps in several areas, meaning apps have fewer capabilities by default and must opt into riskier behaviors.

Attack-surface impact: WebView-based attacks (JavaScript interfaces, mixed content, file access from WebView) are partially mitigated by defaults. Bluetooth-based attacks must contend with more explicit permission and location handling.

## 2. Android 15 — Security and privacy changes

### Privacy refinements

Android 15 continues privacy improvements from Android 13/14:

- Further photo and video access refinements — apps have fewer defaults for broad media access.
- Storage and media access continue moving toward scoped, purpose-limited APIs.
- Location and sensor privacy continue tightening.

Attack-surface impact: The media-access and location-access attack surface shrinks further. Malware that relied on broad media or location access must work harder or target older devices.

### Security hardening

- Scoped storage enforcement continues; more APIs require explicit media/focus permissions or Storage Access Framework usage.
- More APIs are restricted to system or privileged apps.
- Intent filtering and handling continue to be refined for security.
- Background execution and activity start restrictions continue.

Attack-surface impact: The gap between what a normal app can do and what a privileged/system app can do widens. This is good for defense but means analysts working without privileged access may lose some inspection paths.

### Memory safety direction

Android 15 continues the platform's push toward memory-safe languages (Rust) in platform code. The goal is fewer memory-corruption vulnerabilities in system components over time.

Attack-surface impact: In the long run, this reduces the number of exploitable memory-safety bugs in the platform. In the near term, the transition is incomplete — legacy C/C++ components remain attackable, and many third-party apps are still written in Java/Kotlin/C++.

### AccessibilityService and sensitive API restrictions

Android 15 continues refining what AccessibilityService and other sensitive APIs can do, and under what conditions. Some capabilities that were broadly available become more constrained.

Attack-surface impact: Accessibility-based malware (tap simulation, screen reading, input interception) faces additional constraints. Analysts hunting for accessibility abuse must check not just whether the service is enabled, but what it is allowed to do under the current Android version's rules.

### WebView isolation and platform component hardening

Improvements to WebView isolation and other platform components reduce some cross-component attack paths.

Attack-surface impact: WebView-as-attack-surface remains relevant (especially in apps that expose JavaScript interfaces or load untrusted content), but the default posture is stronger.

## 3. Android 16 — Early direction (from developer.android.com summary)

Android 16 is the version under active development at the time of this writing (2026). The developer documentation summary page lists security changes; the headline visible in search results is:

- **Security — Change (all apps): Improved s...** (truncated in search result — full text on the version summary page)

Based on the trajectory from Android 13 through 15, the likely Android 16 themes are:

- Continued scoped storage enforcement and storage-access hardening.
- Further permission-model tightening and runtime permission clarity.
- Continued restrictions on non-SDK/hidden interfaces.
- Continued memory-safety push (Rust) in platform components.
- Further anti-abuse improvements for sensitive APIs (notifications, alarms, accessibility, networking, background execution).
- Security-by-default posture continuing to strengthen.

For concrete, version-specific Android 16 changes, read the full Android 16 version summary page directly:

- https://developer.android.com/about/versions/16/summary

Android 14 behavior changes for apps targeting Android 17 or higher are documented at:

- https://developer.android.com/about/versions/17/behavior-changes-17

## 4. Cross-version trend analysis

### Trend 1 — Scoped storage is the dominant theme

Across Android 13, 14, 15, and 16, scoped storage is the single most consistent theme. External/shared storage access is increasingly restricted, purpose-limited, and mediated through MediaStore / Storage Access Framework / app-specific directories.

What's getting harder for attackers: Bulk file exfiltration from shared storage, reading other apps' public files without appropriate APIs, using storage as a free-form drop zone.

What's getting harder for analysts: Forensics tools and manual inspection that previously read broad storage paths may lose access unless they have appropriate permissions or use the supported APIs.

### Trend 2 — Permission model is increasingly granular and runtime

More capabilities require runtime permissions. More permissions require explicit justification. More behaviors that were free are now gated.

What's getting harder for attackers: Silent access to notifications, exact alarms, media, location, Bluetooth scanning with location implications, and similar capabilities.

What's getting harder for analysts: Testing behaviors that require permissions means testing with the permission model in place, not assuming free access.

### Trend 3 — Security by Default is strengthening

Apps get fewer capabilities by default. Risky behaviors require opt-in. Pending intents are immutable by default. Background starts are more restricted. Exported components must be explicitly declared.

What's getting harder for attackers: Accidental or lazy attack surface — apps that didn't carefully configure components, intents, or permissions are less likely to be exploitable by default.

What's getting harder for analysts: Some older testing assumptions (e.g., "this component is probably reachable because it has an intent filter") become less reliable. Analysts must verify explicit export declarations and permission guards.

### Trend 4 — Hidden API surface is shrinking

Non-SDK interface access continues to be restricted. Greylist/blacklist enforcement tightens. Apps using hidden APIs get more warnings and errors.

What's getting harder for attackers: Malware that relied on hidden APIs for deep inspection, framework manipulation, or evasion has a shrinking toolkit.

What's getting harder for analysts: Instrumentation, hook targets, and forensic inspection paths that relied on hidden APIs may become less reliable. Analysts may need to rely more on supported APIs, explicit instrumentation points, and dynamic analysis.

### Trend 5 — Memory safety is the long-term structural trend

The platform's move toward memory-safe languages (Rust) in system components is a structural, long-term trend. It does not eliminate all memory-safety risk — legacy C/C++ remains, and third-party apps are varied — but it reduces the pool of exploitable memory-corruption bugs in platform code over time.

What's getting harder for attackers: Exploiting memory-corruption bugs in newly written platform components.

What stays the same: Legacy components, device-specific drivers (GPU, modem, DSP, etc.), and third-party native code remain attackable. The kernel, GPU drivers, and vendor-specific code are still the richest sources of serious memory-safety bugs.

### Net assessment

For attackers and malware authors: the environment is getting harder in the app layer — fewer free capabilities, more runtime permissions, stronger defaults, shrinking hidden API surface. The enduring attack surface is in privileged components, vendor drivers, kernel, GPU/modem/DSP, and the gaps between app-level restrictions and system-level access.

For analysts and pentesters: the fundamentals remain — exported components, intent injection, content providers, DCL, accessibility abuse, VPNService abuse, and deep link handling are still the core app-layer attack surface. The change is that more of these now operate under tighter runtime constraints, and some older inspection shortcuts (broad storage access, hidden APIs) are less available.

## 5. Practical impact on Red Team / malware analysis / pentesting workflows

### What's still the same

- Exported component enumeration and testing (drozer, manual manifest inspection)
- Intent injection testing against exported components
- Content provider testing (permissions, SQLi, path traversal, read/write access)
- DCL detection (DexClassLoader, PathClassLoader, assets/OBB loading)
- AccessibilityService abuse detection
- VPNService abuse detection
- Deep link / intent filter URL handling testing
- Network interception and pinning bypass (Frida, httptoolkit)
- Dynamic instrumentation of app-specific behavior
- Static analysis (MobSF, JADX, apktool, Androguard)
- On-device triage and artifact collection

These are still the core workflow. The Android version changes the constraints, not the fundamental methodology.

### What needs adaptation

- **Scoped storage**: When analyzing malware or testing apps, don't assume broad file-system access to shared storage. Check which storage APIs the app uses and whether it has the appropriate permissions or uses the Storage Access Framework.
- **Runtime permissions**: When testing notification abuse, exact alarms, media access, Bluetooth scanning, or similar capabilities, test with the runtime permission model in place. An app that appears to have a capability may actually be blocked at runtime without the permission.
- **Foreground service types**: When analyzing background behavior, check the declared foreground service types. A mismatch between declared type and observed behavior is a signal.
- **Pending intent mutability**: When testing intent-based attacks, remember that mutable pending intents require explicit opt-in on Android 14+. Test whether the app actually uses mutable pending intents.
- **Exported component enforcement**: Verify the explicit `android:exported` declaration, not just the presence of an intent filter. A component with an intent filter and no explicit export declaration may be non-exported on Android 14+.
- **Hidden API reliance**: If your instrumentation or analysis relies on non-SDK interfaces, verify it still works on the target Android version. Some hidden API access that worked on older versions may be blocked or warned on newer versions.
- **WebView testing**: Continue testing WebView JavaScript interfaces, file access, and mixed content, but recognize that defaults are stronger. Focus on apps that explicitly weaken the defaults or expose dangerous interfaces.

### What's getting harder for malware authors

- Silent notification abuse (POST_NOTIFICATIONS runtime permission)
- Exact alarm abuse (justification required)
- Broad media/photo exfiltration (scoped storage, photo picker, metadata limits)
- Background activity launches for UI fraud (background start restrictions)
- Pending intent manipulation (immutable default)
- Accidental exposure of components with intent filters (explicit export enforcement)
- Reliance on hidden APIs (shrinking surface)

### What's getting easier for malware authors (or staying viable)

- Targeting older Android versions where restrictions are weaker
- Using supported-but-sensitive APIs in abusive ways (accessibility, VPNService, notifications with permission)
- DCL and dynamic code loading (still possible, though Android 10+ restricts native library loading from internal storage)
- Vendor-specific and platform-component bugs (driver/kernel/GPU/modem) — these are version- and device-specific, not uniformly mitigated by app-layer restrictions
- Social engineering to obtain runtime permissions from the user

### Recommendations for security workflows

1. **Version-aware testing**: Know the target Android version. Test against the permission model, scoped storage rules, foreground service type requirements, and exported-component enforcement of that version.
2. **Permission-aware analysis**: When you see a capability in static analysis, verify whether it's actually available at runtime under the target Android version's permission model.
3. **Default-aware WebView testing**: Test WebView attack surface with the understanding that defaults are stronger; focus on apps that opt out of safe defaults or expose dangerous interfaces.
4. **Non-SDK fallback planning**: If your analysis or instrumentation relies on hidden APIs, have a fallback plan for when those are restricted on newer Android versions.
5. **Remember the enduring attack surface**: App-layer restrictions do not eliminate the kernel, driver, GPU, modem, and vendor-specific attack surface. For root-level compromise, those remain the high-impact targets — and they are not uniformly mitigated by app-layer security changes.
6. **Cross-reference Samsung/vendor research**: Vendor preinstalled apps remain a rich source of vulnerabilities. Oversecured's research on Samsung preinstalled app vulnerabilities (176 vulnerabilities over three years) is a reminder that OEM/customizations can introduce attack surface that generic Android version notes don't cover.

### Android 16+ Advanced Protection Mode — Intrusion Logging (May 2026)

- **What:** New opt-in Android feature in Advanced Protection Mode, announced May 12, 2026. Records security events: phone unlock times, app install/uninstall, websites/servers connected to, ADB connection attempts, log deletion attempts.
- **Storage:** Logs created daily, stored encrypted in user's Google account. Only the user can access/share; Google cannot.
- **Availability:** Android 16 December update + newer, Pixel devices only, requires Google account link + Advanced Protection Mode enabled. Co-developed with Amnesty International.
- **Why it matters for security work:** First phone-maker feature designed to help researchers and high-risk users investigate spyware attacks. Addresses Android's historical forensic limitations vs. iOS. For anyone investigating compromised devices, this provides a tamper-resistant audit trail that survives app-side tampering (log deletion attempts are themselves logged).
- **Limitations:** Pixel-only. Opt-in — not enabled by default. Android 16 Dec update or newer required. Doesn't replace device forensics; complements it.
- **Source:** TechCrunch (techcrunch.com/2026/05/12/google-launches-new-android-security-feature-to-help-uncover-spyware-attacks)
- **Confidence:** HIGH

## Bottom line

Android 14, 15, and 16 continue the same structural direction: scoped storage, granular runtime permissions, stronger defaults, explicit component export, shrinking hidden API surface, and a long-term push toward memory safety in platform code. The app-layer environment is getting harder for attackers and more constrained for analysts without privileged access — but the core app-layer attack surface (exported components, intents, content providers, DCL, accessibility, VPNService, deep links) remains, and the high-impact kernel/driver/vendor attack surface is largely unaffected by these app-layer changes.

For practical security work, test version-aware, permission-aware, and default-aware — but don't lose sight of the enduring attack surface that these changes don't address.
