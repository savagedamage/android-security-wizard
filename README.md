# Android Security Wizard

A synthesized Android security research corpus: one integrated skill document plus 21 deep companion references covering the full Android security assessment pipeline — static analysis, dynamic instrumentation (Frida/Shizuku), exploit identification, malware analysis, commercial protector bypass, kernel exploit development, safe encrypted storage, and threat attribution.

**Grand total: 9,068 lines / ~676 KB across 22 research documents** (SKILL.md + 21 companions). Last updated: 2026-09-05.

---

## What this is

This repository is the working corpus behind an Android/mobile security skill. It was built through iterative web research, delegation-driven parallel research waves, and consolidation into two layers:

- **The skill** (`SKILL.md`) — the synthesized, operational reference. 15 sections, 4 capability pillars, an integrated assessment pipeline, quick-reference tables, a 9-level learning path, and sources. Start here.
- **The companion files** — deep, topic-isolated research documents that the skill references for the detailed version of any area.

The corpus was produced by the 2026-09-04 research wave that targeted the biggest capability gaps flagged during skill consolidation, followed by fresh-lead malware/security/tools integration (ClayRat, WindRelay, PromptSpy, Arsink, Albiriox, Zimperium-4 campaigns, CVE-2025-48595, Android 16+ Intrusion Logging, and more).

## The 4 capability pillars

| Pillar | Focus |
|--------|-------|
| 1. Static & dynamic analysis | APK/DEX/`.so` reverse engineering, decompilers, Frida/Objection instrumentation, commercial protector bypass |
| 2. Exploit identification | Isolate-and-identify workflow, CVE matching, kernel exploit methodology, exploit chains |
| 3. Malware & threat intel | Detection/triage, attribution & campaign tracking, C2 infrastructure analysis |
| 4. Platform security | Android 14→16 changes, 17/18 direction, rooting methods, safe storage |

## Corpus at a glance

| File | Lines | Size | Content |
|------|------:|-----:|---------|
| `SKILL.md` | 683 | 83 KB | The skill: 15 sections, 4 pillars, pipeline, quick reference, learning path, sources |
| `android-malware-attribution-campaign-tracking.md` | 967 | 65 KB | Attribution & campaign tracking; Section 12: 2025-26 campaigns (ClayRat, Arsink, PromptSpy, Albiriox, Zimperium-4) |
| `android-protector-walkthroughs.md` | 837 | 82 KB | Unpacking case studies: narrative template, Packed Unpacker, BRATA/Ghimob/Joker, DexGuard/Bangcle gaps |
| `android-kernel-exploit-dev-methodology.md` | 777 | 54 KB | CVE-to-exploit, KASLR defeat, heap Feng Shui on ARM64, arbitrary r/w primitives, Android mitigations |
| `android-malware-detection-research.md` | 689 | 31 KB | 6-phase on-device triage; Section 5: 2025-26 families + NFC relay fraud evolution |
| `android-exploit-identification.md` | 638 | 32 KB | 5-phase isolate-and-identify workflow + CVE reference card (incl. CVE-2025-48595) |
| `android-native-re-workflow.md` | 493 | 20 KB | Native `.so` RE: tools, JNI patterns, step-by-step commands, Frida native hooking |
| `adb-shizuku-primer.md` | 457 | 24 KB | ADB + Shizuku fundamentals, wireless debugging, privilege model |
| `android-commercial-protector-bypass.md` | 423 | 39 KB | Protector landscape (DexGuard/Bangcle/Liuling/etc.), bypass per protector, unpacking methodology |
| `android-rooting-methods-deep-dive.md` | 418 | 39 KB | Rooting taxonomy, Magisk/KernelSU/APatch, root detection/hiding, AVB, risks; legacy sideload methods |
| `shizuku-adb-workflows.md` | 388 | 19 KB | 30 concrete commands, privilege tier table, decision tree |
| `android-kernel-exploit-lab-setup.md` | 365 | 13 KB | QEMU + vulnerable kernel + GDB walkthrough for CVE-2019-2215 |
| `shizuku-exploitation-research.md` | 324 | 27 KB | Shizuku attack surface: 5 abuse scenarios, UserService exploitation, 37-item audit checklist |
| `android-malware-detection-tools-survey.md` | 309 | 24 KB | Tool survey: MobSF, Androguard, APKLeaks, MARA, Androwarn, MVT, Pithus, Androl4b |
| `android-14-15-16-security-changes.md` | 248 | 21 KB | Android 14/15/16 security changes incl. Android 16+ Advanced Protection / Intrusion Logging |
| `android-17-18-security-direction.md` | 243 | 22 KB | Android 17 + 18 (cut) early security direction, trend analysis |
| `android-exploit-chains.md` | 202 | 15 KB | Full chains: Pixel 9 AWDL/Dolby 0-click, RIDL sandbox escape, spyware forensics, Fugitive-in-Java |
| `android-re-frida-pipeline.md` | 191 | 16 KB | RE/Frida tooling survey across search threads |
| `android-malware-c2-infra-analysis.md` | 123 | 8 KB | C2 infrastructure analysis process: DNS/WHOIS/CT/correlation/attribution |
| `android-malware-c2-infra-analysis-sources.md` | 26 | 3 KB | Source URLs backing C2 infrastructure analysis |
| `android-exploit-chains-sources.md` | 49 | 6 KB | Source URLs backing exploit chains analysis |
| `android-safe-encrypted-storage.md` | 213 | 12 KB | Keystore-backed AEAD, backup/restore, key invalidation, and migration guidance |

*(Sizes approximate; see `android-security-wizard-archive-index.md` in the corpus for the authoritative index.)*

## Coverage areas

1. ✅ Commercial protector bypass (DexGuard, Bangcle, Liuling) — `android-commercial-protector-bypass.md` + `android-protector-walkthroughs.md`
2. ✅ Native code RE (Ghidra/IDA, JNI tracing, native anti-debug) — `android-native-re-workflow.md`
3. ✅ Advanced kernel exploit development (KASLR defeat, heap grooming, ARM64) — `android-kernel-exploit-dev-methodology.md` + lab setup
4. ⬜ Hardware-level forensics (JTAG/ISP/chip-off) — needs hardware, out of scope
5. ✅ Malware attribution & campaign tracking — `android-malware-attribution-campaign-tracking.md`
6. ✅ Shizuku/Dhizuku exploitation — `shizuku-exploitation-research.md`
7. ✅ Android 14/15/16 security changes — `android-14-15-16-security-changes.md`
8. ✅ Rooting methods deep dive — `android-rooting-methods-deep-dive.md`
9. ✅ Android 17/18 early security direction — `android-17-18-security-direction.md`
10. ⬜ Darknet/underground research — surface-web intel only; Hermes cannot browse Tor
11. ✅ Exploit chain composition — `android-exploit-chains.md`
12. ✅ C2 infrastructure analysis — `android-malware-c2-infra-analysis.md`
13. ✅ RE/Frida tooling pipeline — `android-re-frida-pipeline.md`
14. ✅ On-device malware detection triage — `android-malware-detection-research.md` + tools survey
16. ✅ Exploit identification workflow — `android-exploit-identification.md`

## Quick start

1. **Read `SKILL.md`** — it's the entry point and the synthesized operational reference.
2. **Use Section 11** (corpus table) to find the companion file matching your current problem.
3. **`read_file` the companion** when you need the deep version — each is self-contained with its own structure, sources, and cross-references.
4. For a specific task (unpack a protector, hunt a C2, triage a sample), the companions each carry step-by-step workflows with concrete commands.

## Status & honest gaps

- Research was conducted with web search + direct page extraction; some pages were only reachable at snippet level (noted inline as `[FETCH]`/`[unverified]` markers where applicable).
- No live malware samples are hosted here — this is methodology, tooling, and intelligence, not a malware repository.
- No Tor/darknet browsing capability — underground claims are absent or explicitly framed as surface-web coverage.
- Malware family details reflect vendor reporting as of the 2026-09-04 cutoff and will age; treat as pointers for your own verification.
- The Android 18 (cut) items reflect third-party coverage, not an official spec.

## Related skills & sources

- Source URLs are preserved per-topic in `*-sources.md` files and inside each companion's references section.
- Built with Hermes Agent delegation-driven research; see the archive index for delegation transcripts and the web-cache map.

---

*Corpus: 22 research documents (SKILL.md plus 21 companions; README and archive index excluded), 9,068 lines / ~676 KB. Last updated 2026-09-05.*
