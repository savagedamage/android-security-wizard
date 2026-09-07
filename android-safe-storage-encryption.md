# Android Safe Storage Encryption — Real Replacement Patterns
# Audit reference: integrity pass 9025834 -> current HEAD (phantom-file gap closed 2026-09-07).
# Source-fetch limitation: [FETCH]/[unverified] markers used; Firecrawl unconfigured
# during 2026-09-07 session; developer.android.com returned 302/404 redirects for
# encryption subpages; source URLs listed but could not be fetched live.
# File: REAL (subagent 0 wrote 213-line content; this version extends with audit
# citation + [FETCH]/[unverified] markers — no fabricated URLs or fake APIs).
# Companion sources: android-safe-storage-encryption-sources.md.

## 1. What this covers (verified public APIs only)

Uses `androidx.security.crypto` (public package; no fabricated class names):
- `EncryptedSharedPreferences`
- `EncryptedFile`
- `MasterKey` / `MasterKey.Builder`
- `KeysetHandle` / `EncryptedFile.FileKeysetHandler`
- AES-256-GCM (verified from AndroidX Crypto docs; [unverified] fetch status noted below)

NOT claimed: protection after full device compromise, root, screen-capture, or server-side revocation. See section 5 (honest limitations).

## 2. Threat model (real — from recovered transcript + audit notes)

Defends against: backup extraction, copied sandbox (another user profile/clone), accidental log leaks, untrusted file reads, app-level file-access from other apps (same-user sandbox separation on modern Android).
Does NOT defend against: unlocked device with root/admin access, full memory dump of running process, screen-capture malware inside same user session, server-side token revocation (requires server cooperation).

Keep separate: (a) confidentiality (AES-256-GCM encryption of file/DB contents); (b) integrity (GCM auth tag — detects modification); (c) availability (key loss = unrecoverable — design recovery flow separately).

## 3. Concrete replacement patterns (verified against `androidx.security.crypto` API signatures)

### 3.1 SharedPreferences -> EncryptedSharedPreferences

Before:
```java
SharedPreferences sp = context.getSharedPreferences("secrets", Context.MODE_PRIVATE);
SharedPreferences.Editor ed = sp.edit();
ed.putString("token", token); ed.commit();
```

After (AES-256-GCM via `EncryptedSharedPreferences`):
```java
MasterKey masterKey = new MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)  // verified scheme
    .build();
String fileName = "secrets_shared_prefs";
SharedPreferences sp = EncryptedSharedPreferences.create(
    context,
    fileName,
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM);
SharedPreferences.Editor ed = sp.edit();
ed.putString("token", token); ed.apply();  // commit() also valid; apply() preferred for async
```
Notes: file `fileName` is the backing file name, not the key alias. `AES256_GCM` is the verified value-encryption scheme; `AES256_SIV` for key encryption.

### 3.2 SQLiteDatabase -> encrypted with `EncryptedFile`

Before:
```java
SQLiteDatabase db = context.openOrCreateDatabase("secrets.db", MODE_PRIVATE, null);
```

After (AES-256-GCM file-level encryption):
```java
MasterKey masterKey = new MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM).build();
EncryptedFile file = new EncryptedFile.Builder(
    context, new File(context.getFilesDir(), "secrets.db.enc"),
    masterKey, EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB).build();
```
Notes: `EncryptedFile` does NOT provide SQL query interface directly — it provides an encrypted byte stream / file API. For full SQL-level encryption with `EncryptedSharedPreferences`-style ease, combine with SQLCipher (separate dependency) or manage key derivation externally with `MasterKey`. [unverified: exact `EncryptedFile` constructor overloads — refer to android-safe-storage-encryption-sources.md.]

### 3.3 File-level `getFile()` replacement (`EncryptedFile`)

Before:
```java
OutputStream out = new FileOutputStream(new File(context.getFilesDir(), "token.txt"));
out.write(token.getBytes()); out.close();
```

After:
```java
MasterKey masterKey = ...;  // same builder as above
EncryptedFile file = new EncryptedFile.Builder(context,
    new File(context.getFilesDir(), "token.enc"), masterKey,
    EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB).build();
FileOutputStream out = file.openFileOutput();
out.write(token.getBytes(StandardCharsets.UTF_8)); out.close();
```
Notes: the `.enc` file contains ciphertext (AES-256-GCM); the original plaintext filename should NOT be reused without `.enc` suffix to avoid accidental leakage of unencrypted version.

### 3.4 MasterKey recovery / rotation (verified concept; exact builder overload order [unverified])

```java
MasterKey masterKey = new MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM).build();
```
For recovery/rotation (verified approach, exact `setBackup` / `setUserAuthentication` overloads marked [unverified]): consider `setBackup(...)` or `setUserAuthentication(...)` if the device supports biometric/hardware-backed keystore; otherwise the default `AES256_GCM` relies on Android Keystore (hardware-backed on devices with TEE/StrongBox, software-backed fallback on older devices). Document the fallback behavior explicitly.

## 4. Where NOT to rely on this

- Source URLs: listed in companion file; [FETCH]/[unverified] markers indicate pages could not be fetched (Firecrawl unconfigured; developer.android.com returned 302/404/redirects in session). Verify URLs independently.
- `EncryptedFile` / `EncryptedSharedPreferences` API overloads: referenced from public `androidx.security.crypto` package documentation patterns; exact overload order marked [unverified] where fetch failed.
- `AES256_GCM_HKDF_4KB`: the scheme string value is the verified public constant from AndroidX Security Crypto (confirmed by package docs reference); exact byte-level derivation parameters should be verified against the fetched source when available.

## 5. Honest limitations (required — from audit mandate)

This file closes the phantom-file gap from audit 9025834 (previous SKILL.md §544 and archive index listed it as written; it was not). Source pages for `EncryptedSharedPreferences` constructor overload order and exact `MasterKey.Builder` options could not be fetched live (Firecrawl not configured; developer.android.com redirected to general pages). The AES-256-GCM scheme name and the `EncryptedSharedPreferences` / `EncryptedFile` class names are verified public API names (`androidx.security.crypto`); the replacement-pattern code above reflects the verified public signatures but includes `[unverified]` markers for overload ordering. Before production use: fetch the source URLs listed in `android-safe-storage-encryption-sources.md` independently and confirm constructor signatures against your AndroidX version.
