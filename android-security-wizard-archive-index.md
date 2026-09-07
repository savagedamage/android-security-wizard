# Android Security Wizard — Research Archive Index

*Created: 2026-09-04 | Purpose: long-term reference index for the Android security research corpus*

## Skill (master document)
- `/home/lzrk/.hermes/skills/android-security-wizard/SKILL.md` (83KB) — the working skill. 15 sections, 4 pillars, integrated pipeline, quick reference, learning path, sources, operational notes. This is the entry point for future sessions.

## Structured research files (on disk, in /home/lzrk/)
| File | Lines | KB | Purpose |
|------|-------|-----|---------|
- adb-shizuku-primer.md | 457 | 23.3 | ADB + Shizuku fundamentals, wireless debugging, privilege model
- shizuku-adb-workflows.md | 388 | 19.0 | 30 concrete commands, privilege tier table, decision tree
- android-malware-detection-research.md | 689 | 30.4 | 6-phase on-device triage, 9 tools, YARA sources, stalkerware IOCs
- android-malware-detection-tools-survey.md | 309 | 23.0 | Tool survey (MobSF, Androguard, APKLeaks, MARA, Androwarn, MVT, Pithus, Androl4b)
- android-exploit-identification.md | 638 | 31.1 | 5-phase isolate-and-identify workflow, CVE reference card, drozer+Ghost fit
- android-kernel-exploit-lab-setup.md | 365 | 12.9 | QEMU + vulnerable kernel + GDB walkthrough for CVE-2019-2215
- android-commercial-protector-bypass.md | 423 | 38.2 | Commercial protector landscape (DexGuard/Bangcle/Liuling/etc.), bypass per protector, manual unpacking methodology
- android-14-15-16-security-changes.md | 248 | 20.7 | Android 14/15/16 security changes by feature area, trend analysis, Red Team impact. Includes Android 16+ Intrusion Logging (May 2026)
- android-native-re-workflow.md | 493 | 19.5 | Native .so RE workflow: tools, JNI patterns, step-by-step commands, Frida native hooking
- shizuku-exploitation-research.md | 324 | 26.5 | Shizuku attack surface: 5 abuse scenarios, UserService exploitation, 37-item audit checklist
- android-re-frida-pipeline.md | 191 | 15.2 | RE/Frida tooling survey across multiple search threads
- android-exploit-chains.md | 202 | 14.4 | Full exploit chain analysis: Pixel 9 0-click, post-exploitation campaigns, RIDL escape, spyware forensics
- android-malware-c2-infra-analysis.md | 123 | 7.3 | C2 infrastructure analysis process: DNS/WHOIS/CT/correlation/attribution
- android-kernel-exploit-dev-methodology.md | 777 | 53.1 | Advanced kernel exploit dev: KASLR defeat, heap grooming, r/w primitives, mitigations, ARM64, case studies
- android-malware-attribution-campaign-tracking.md | 967 | 65.4 | Attribution & campaign tracking; Section 12: 2025-26 campaigns (ClayRat, Arsink, PromptSpy, Albiriox, Zimperium-4)
- android-protector-walkthroughs.md | 837 | 79.9 | End-to-end unpacking case studies: narrative template, Packed Unpacker, BRATA/Ghimob/Joker, DexGuard gaps
- android-rooting-methods-deep-dive.md | 418 | 37.7 | Rooting taxonomy, Magisk/KernelSU/APatch, root detection/hiding, AVB, risks; legacy sideload methods
- android-17-18-security-direction.md | 243 | 21.4 | Android 17 + 18 (cut) early security direction, trend analysis, adaptation recommendations
- android-malware-c2-infra-analysis-sources.md | 26 | 3.1 | Source URLs backing C2 infrastructure analysis
- android-exploit-chains-sources.md | 49 | 6.0 | Source URLs backing exploit chains analysis
- android-safe-encrypted-storage.md | 213 | 12.0 | Android Keystore, AEAD envelope encryption, backup/restore, key invalidation, and migration from deprecated AndroidX crypto wrappers

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
15. Safe encrypted storage (Android Keystore, AEAD envelope patterns, backup/restore, and migration from deprecated AndroidX crypto wrappers) — now covered: android-safe-encrypted-storage.md
16. Exploit identification workflow — now covered: android-exploit-identification.md (638 lines, 5-phase isolate-and-identify + CVE reference card with CVE-2025-48595 + 124-Jun/215-H1 stats)

*End of index.*
