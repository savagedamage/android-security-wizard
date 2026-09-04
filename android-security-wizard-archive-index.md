# Android Security Wizard — Research Archive Index

*Created: 2026-09-04 | Purpose: long-term reference index for the Android security research corpus*

## Skill (master document)
- `/home/lzrk/.hermes/skills/android-security-wizard/SKILL.md` (80KB) — the working skill. 15 sections, 4 pillars, integrated pipeline, quick reference, learning path, sources, operational notes. This is the entry point for future sessions.

## Structured research files (on disk, in /home/lzrk/)
| File | Lines | KB | Purpose |
|------|-------|-----|---------|
- adb-shizuku-primer.md | 457 | 23.8 | ADB + Shizuku fundamentals, wireless debugging, privilege model
- shizuku-adb-workflows.md | 388 | ~19.5 | 30 concrete commands, privilege tier table, decision tree
- android-safe-storage-encryption-sources.md | 39 | 3.7 | Source URLs backing safe-storage-encryption.md
- android-safe-storage-encryption.md | 30 | 3.6 | AES-256-GCM key derivation + IV handling replace pattern
- android-malware-detection-research.md | 570 | 24.8 | 6-phase on-device triage, 9 tools, YARA sources, stalkerware IOCs
- android-malware-detection-tools-survey.md | 309 | ~15 | Tool survey (MobSF, Androguard, APKLeaks, MARA, Androwarn, MVT, Pithus, Androl4b)
- android-exploit-identification.md | 635 | 31.6 | 5-phase isolate-and-identify workflow, CVE reference card, drozer+Ghost fit, saelo replacement
- android-kernel-exploit-lab-setup.md | 365 | ~15 | QEMU + vulnerable kernel + GDB walkthrough for CVE-2019-2215
- android-commercial-protector-bypass.md | 423 | 39.1 | Commercial protector landscape: DexGuard/Bangcle/Liuling/DarkGuard/APKProtect/ARPSolo/LLVM/custom; bypass approaches per protector; manual unpacking methodology; detection/identification; case studies + 8 hard problems; 9-step practical workflow.
|| android-14-15-16-security-changes.md | 248 | 20.8 | Android 14/15/16 security changes by feature area, cross-version trend analysis, practical impact on Red Team/malware/pentest. Includes Android 16+ Advanced Protection Mode / Intrusion Logging (May 2026).
|| android-native-re-workflow.md | 493 | 19.9 | Native .so RE workflow: tools (Ghidra/IDA/JEB/radare2/angr), JNI patterns, step-by-step commands, malware native-library patterns, anti-analysis bypass notes, Frida native hooking patterns with JS examples.
|| shizuku-exploitation-research.md | 324 | 27.2 | Shizuku attack surface: 5 privilege escalation/abuse scenarios, UserService exploitation methodology, malicious client crafting, 37-item defensive audit checklist.
|| android-re-frida-pipeline.md | 191 | 15.6 | RE/Frida tooling survey across multiple search threads.
|| android-exploit-chains.md | 202 | 14.8 | Full exploit chain analysis: Project Zero Pixel 9 AWDL/Dolby 0-click chain, in-the-wild post-exploitation campaigns, Chrome RIDL sandbox escape, Amnesty/Citizen Lab spyware forensics, Oversecured app-layer vulns, Google Android OffSec Binder CVE, GitHub Fugitive-in-Java chain, Cyfirma GhostShell/Crocodile Gang.
|| android-malware-c2-infra-analysis.md | 123 | 7.5 | C2 infrastructure analysis process: DNS/WHOIS/passive DNS/cert transparency/WEBINT/lateral movement/fraud/attribution.
|| android-kernel-exploit-dev-methodology.md | 777 | 54.4 | Advanced kernel exploit dev: CVE-to-exploit, KASLR defeat, heap Feng Shui/grooming on ARM64, arbitrary r/w primitives, Android mitigations (KASAN/KCFG/CFI/PAN/STATIC_USERMODE_HELPER/KPTI/module signing), ARM64 specifics (PAC, calling conventions), cred manipulation, case studies (2023-2026), tooling, robust exploit engineering, CVE reference table.
|| android-malware-attribution-campaign-tracking.md | 967 | 65.4 | Android malware attribution & campaign tracking: sample-to-campaign linking, infrastructure tracking, financial fraud tracing, threat actor attribution, campaign timeline, family classification, tools, case study references, open-source workflow, honest limitations. Section 12 adds 2025-2026 campaign case studies (ClayRat, Arsink, PromptSpy, Albiriox, Zimperium 4 campaigns).
|| android-protector-walkthroughs.md | 837 | 81.8 | End-to-end unpacking case studies: narrative template, Packed Unpacker (Maddie Stone BH 2020) deep analysis, BRATA/Ghimob/Joker walkthroughs, DexGuard/Bangcle/Liuling-specific walkthroughs with honest gaps, 14-step tool sequence with pivot-points table, where to find more.
|| android-rooting-methods-deep-dive.md | 418 | 38.6 | Comprehensive rooting reference: taxonomy, bootloader unlock + AVB 2.0, Magisk, KernelSU/APatch, exploit-based rooting historical context, root detection, root hiding cat-and-mouse, what root unlocks for research, risks, practical recommendations, future direction.
|| android-17-18-security-direction.md | 243 | 21.9 | Android 17 known direction + Android 18 (cut) early security direction. Security trend analysis 14→15→16→17→18, Red Team/malware/pentest impact, adaptation recommendations.
|| android-safe-storage-encryption.md | 30 | 3.6 | AES-256-GCM key derivation + IV handling replace pattern for Android EncryptedFile, SQLiteDatabase, MasterKey, EncryptedSharedPreferences.
|| android-malware-c2-infra-analysis-sources.md | 26 | 3.1 | Source URLs backing C2 infrastructure analysis.
|| android-malware-attribution-campaign-tracking.md | 891 | 58.4 | Android malware attribution & campaign tracking: sample-to-campaign linking, infrastructure tracking (passive DNS/WHOIS/CT logs/hosting patterns/account reuse/lifecycle), financial fraud tracing (money mules/cash-out/crypto/SMS toll fraud/ad fraud), threat actor attribution (TTPs/code reuse/infrastructure overlap/targeting/confidence levels), campaign timeline reconstruction, family classification, tools (VirusTotal/MalwareBazaar/AndroZoo/YARA/ct.sh/WHOIS history/MISP/OpenCTI/OTX), case study references, open-source workflow, honest limitations. |
|| android-protector-walkthroughs.md | 837 | 81.8 | End-to-end unpacking case studies: narrative template, Packed Unpacker (Maddie Stone BH 2020) deep analysis, BRATA/Ghimob/Joker walkthroughs, DexGuard/Bangcle/Liuling-specific walkthroughs with honest gaps, 14-step tool sequence with pivot-points table, where to find more. |
|| android-rooting-methods-deep-dive.md | 418 | 38.6 | Comprehensive rooting reference: taxonomy, bootloader unlock + AVB 2.0, Magisk (MagiskSU/Zygisk/DenyList/modules), KernelSU/APatch, exploit-based rooting historical context, root detection (SafetyNet→Play Integrity, software vs hardware-backed, TEE), root hiding cat-and-mouse, what root unlocks for research, risks (OTA/Knox/AVB/bootloops/integrity flags), practical recommendations, future direction.
- android-17-18-security-direction.md | 243 | 21.9 | Android 17 known direction + Android 18 (cut) early security direction: privacy/storage/permission tightening, per-app language, predictive back, HEIF/USB3 camera, refined intent filters (17); battery/charging visibility, hourly weather, freeze-ended texting glanceable info, back gestures via launcher, auto-verify app purchases, surrounding devices API, 16:10+ displays, battery saver refinements, threads/conversation ranking (18). Security trend analysis 14→15→16→17→18, Red Team/malware/pentest impact, adaptation recommendations. Sources cited with honest uncertainty markers.
- android-safe-storage-encryption.md | 30 | 3.6 | AES-256-GCM key derivation + IV handling replace pattern for Android EncryptedFile, SQLiteDatabase, MasterKey, EncryptedSharedPreferences. Replace patterns for getFile(), getDatabase(), getSharedPreferences(), commit().

## Web cache (source pages, /home/lzrk/.hermes/cache/web/)
~50+ cached source pages, ~3.9MB total. Key ones:
- github.com-d4e1eaab70.md — JADX full repo read (50.3k stars, CLI/GUI, plugins, Smali input)
- github.com-0be287f578.md — httptoolkit/frida-interception-and-unpinning (2.3k stars, MITM + unpinning scripts)
- github.com-d60b72c802.md — anpa1200/Android-Malware-Analysis (YARA + ATT&CK + Frida + LLM triage)
- github.com-1b9828350b.md — sensepost/objection (9.4k stars, Frida-powered mobile exploration)
- github.com-6d4b591c0e.md — MobSF full repo read (21.7k stars, v4.5.2)
- github.com-bc1e889e6a.md — user1342/Awesome-Android-Reverse-Engineering (2.7k stars, canonical RE list)
- github.com-cacc26863b.md — timschneeb/awesome-shizuku (10k stars, full app catalog)
- github.com-a23679946d.md — shizuku-android topic page (27 repos, security tools slice)
- github.com-4fdca5ede6.md — ReversecLabs/drozer (4.6k stars, full repo read)
- codeshare.frida.re-fa8dc1aa83.md — universal root + SSL pinning bypass (108KB, full script)
- docs.talsec.app-ff0d373215.md — Frida impact + RASP detection article
- hacktricks.wiki-1410f9547e.md — Shizuku privileged API security methodology
- androidoffsec.withgoogle.com-3f5860df65.md — Android Binder exploitation (CVE-2023-20938)
- projectzero.google-*.md — Project Zero in-the-wild exploit analyses (multiple)
- frida.re-ff62626d61.md — Frida documentation
- research.checkpoint.com-3838c8b514.md — Android security research
- blog.quarkslab.com-84eaa59a50.md — Quarkslab Android RE
- blog.ostorlab.co-4f168aa86e.md — Ostorlab Android security
- redfoxsec.com-e0591dad68.md — Android security
- highon.coffee-641746ef50.md — Android security
- arxiv.org-36eff8d601.md — academic paper on deobfuscation
- httptoolkit.com-0495aebec8.md — EU-funded mobile interception project

## Delegation transcripts (/home/lzrk/.hermes/cache/delegation/live/)
- deleg_8571ae8f/ — wave 3: worked examples, kernel lab setup, current malware families + Shizuku security tools + non-root Frida
- deleg_04084e8e/ — wave 2: consolidated pipeline (MobSF+Frida+YARA), RE workflow
- deleg_355d2891/ — wave 4 (running): advanced Frida/RASP bypass, deobfuscation/unpacking, vulnerability discovery methodology
- deleg_875d6ca5/ — wave 1b: exploit identification (result in android-exploit-identification.md)
- deleg_f6fa80dd/ — wave 1a: ADB/Shizuku workflows, malware detection, RE tooling (results in respective files)

## What's where
- The skill is the synthesized, usable artifact. Start there.
- Structured research files are the detailed backups, organized by topic.
- Web cache is the raw source material — use `read_file` with offset/limit to page through.
- Delegation transcripts are the research traces — useful for understanding how conclusions were reached, or for re-running specific searches.

## Next wave topics (flagged in skill Section 11)
1. Android app packing / commercial protector bypass (DexGuard, Bangcle, Liuling, etc.) — now covered: android-commercial-protector-bypass.md (423 lines) + android-protector-walkthroughs.md (837 lines)
2. Native code RE on Android (Ghidra/IDA, JNI tracing, native anti-debug) — now covered: android-native-re-workflow.md (493 lines)
3. Advanced kernel exploit development (KASLR defeat, heap spraying, ARM64 techniques) — now covered: android-kernel-exploit-dev-methodology.md (777 lines) + android-kernel-exploit-lab-setup.md (365 lines)
4. Hardware-level forensics (JTAG/ISP/chip-off — out of scope for this skill, needs hardware)
5. Android malware attribution & campaign tracking — now covered: android-malware-attribution-campaign-tracking.md (967 lines)
6. Shizuku/Dhizuku exploitation (abusing Shizuku-enabled apps, UserService exploitation) — now covered: shizuku-exploitation-research.md (324 lines)
7. Android 14/15/16-specific security changes and attack surface evolution — now covered: android-14-15-16-security-changes.md (248 lines). Includes Android 16+ Advanced Protection Mode / Intrusion Logging (May 2026).
8. Android rooting methods deep dive — now covered: android-rooting-methods-deep-dive.md (418 lines). Section 10 added 2026-09-04: historical sideload methods (Quip, Priv/Ease), KingRoot NOT CURRENTLY VETTED.
9. Android 17/18 early security direction — now covered: android-17-18-security-direction.md (243 lines)
10. Darknet/underground research for Android malware (surface-web intelligence only — Hermes cannot browse Tor)
11. Exploit chain composition (app-layer → kernel → post-exploit) — partially covered: android-exploit-chains.md (202 lines)
12. C2 infrastructure analysis for Android malware — now covered: android-malware-c2-infra-analysis.md (123 lines) + sources: android-malware-c2-infra-analysis-sources.md (26 lines)
13. RE/Frida tooling pipeline — now covered: android-re-frida-pipeline.md (191 lines)
14. On-device malware detection tooling landscape — now covered: android-malware-detection-research.md (689 lines, 6-phase triage) + android-malware-detection-tools-survey.md (309 lines, tool survey)
15. Safe encrypted storage: path to correctness from vulnerable usage in production traceability — now covered: android-safe-storage-encryption.md (30 lines) + sources: android-safe-storage-encryption-sources.md (39 lines)
16. Exploit identification workflow — now covered: android-exploit-identification.md (635 lines, 5-phase isolate-and-identify + CVE reference card with CVE-2025-48595 + 124-Jun/215-H1 stats)
15. Exploit identification workflow — now covered: android-exploit-identification.md

*End of index.*
