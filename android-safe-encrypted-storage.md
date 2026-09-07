# Android Safe Encrypted Storage

## Purpose and threat model

Use this guide when an Android app must protect tokens, refresh credentials,
PII, recovery material, or other secrets at rest. It assumes the app is
defending against backup extraction, a copied sandbox, accidental logs, and
untrusted files. It does not claim to protect secrets after the app process is
fully compromised, after an unlocked device has been instrumented, or from a
user who can authenticate to the app.

Keep three questions separate:

1. **Confidentiality:** can another app or a copied file read the plaintext?
2. **Integrity:** can an attacker modify the value without detection?
3. **Availability:** can key loss or corruption prevent legitimate recovery?

Authenticated encryption addresses confidentiality and tamper detection. It
does not solve key recovery, rollback, screen capture, memory scraping, or
server-side token revocation.

## Current API position

`androidx.security:security-crypto` is no longer the preferred long-term API.
The Android reference marks `EncryptedFile`, `EncryptedSharedPreferences`, and
`MasterKey` deprecated and directs new code toward platform APIs, direct
Android Keystore use, and ordinary `File`/`SharedPreferences` as storage
primitives:

- [AndroidX security crypto package reference](https://developer.android.com/reference/androidx/security/crypto/package-summary)
- [Android Keystore system](https://developer.android.com/privacy-and-security/keystore)
- [AndroidX security release notes](https://developer.android.com/jetpack/androidx/releases/security)

The deprecation does **not** mean that a plaintext preference or file is safe.
It means the application should own the encryption envelope and lifecycle
explicitly instead of adding new dependencies on the deprecated wrappers.
Existing deployments need a compatibility and migration plan; do not delete an
old key or encrypted file before a successful read and verified rewrite.

## Recommended architecture

For small values, use a Keystore-backed AES-GCM key directly. For larger or
structured data, use envelope encryption:

```
random data-encryption key (DEK)
        │ AES-GCM encrypts payload + authenticated metadata
        ▼
payload file / database blob
        ▲
        │ Keystore-backed key-encryption key (KEK) wraps the DEK
        │
Android Keystore (non-exportable KEK)
```

The file contains a version, algorithm identifier, key identifier, nonce,
ciphertext, and authentication tag. It must not contain the plaintext DEK.
Use a fresh, unpredictable 12-byte nonce for every AES-GCM encryption under a
given key. Bind stable context (for example package, record type, and schema
version) as associated data so a valid ciphertext cannot be transplanted into
another record type.

### Small secret example (Kotlin)

This is a minimal pattern for a single secret. Production code should add
atomic replacement, explicit error handling, a format version, and tests for
process death and key invalidation.

```kotlin
private const val ALIAS = "app_secret_v1"
private const val TRANSFORMATION = "AES/GCM/NoPadding"

private fun key(): SecretKey {
    val ks = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
    (ks.getKey(ALIAS, null) as? SecretKey)?.let { return it }

    val generator = KeyGenerator.getInstance(
        KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore"
    )
    generator.init(
        KeyGenParameterSpec.Builder(
            ALIAS,
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        )
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .setKeySize(256)
            .build()
    )
    return generator.generateKey()
}

fun seal(plaintext: ByteArray, aad: ByteArray): ByteArray {
    val cipher = Cipher.getInstance(TRANSFORMATION)
    cipher.init(Cipher.ENCRYPT_MODE, key())
    cipher.updateAAD(aad)
    return cipher.iv + cipher.doFinal(plaintext)
}

fun open(blob: ByteArray, aad: ByteArray): ByteArray {
    require(blob.size > 12) { "truncated ciphertext" }
    val cipher = Cipher.getInstance(TRANSFORMATION)
    cipher.init(
        Cipher.DECRYPT_MODE,
        key(),
        GCMParameterSpec(128, blob.copyOfRange(0, 12))
    )
    cipher.updateAAD(aad)
    return cipher.doFinal(blob.copyOfRange(12, blob.size))
}
```

Never silently fall back to a hard-coded key, a password-derived key with a
fixed salt, ECB, CBC without authentication, or Base64-only “encryption”. A
Keystore alias is an identifier, not secret material; protect access to the
alias with app authentication when the threat model requires it.

## Files, preferences, and databases

### Preferences

Use ordinary `SharedPreferences` only for non-sensitive configuration, or store
one authenticated encrypted blob rather than scattering secrets across many
keys. Keep keys, values, and migration markers separate from logs and analytics.
For new designs, prefer a small repository abstraction so the storage backend
can be replaced without changing callers.

### Files

Encrypt bytes before writing them to `filesDir` or an app-private database.
Write to a temporary file, flush and close it, then atomically replace the
destination. Set a restrictive app-private location and never place secret
material in external/shared storage, cache directories, screenshots, URI
exports, or crash reports.

The deprecated `EncryptedFile` documentation warns that encrypted files and
their keyset preferences should be excluded from Auto Backup because a restored
file may no longer have its Keystore key. The same rule applies to a custom
envelope unless the restore flow deliberately re-establishes the key.

### Database records

Use per-record or per-table associated data and avoid deterministic encryption
of values unless equality leakage is explicitly accepted. Do not encrypt only
one column while leaving identifiers, search indexes, timestamps, or debug
exports able to reconstruct the secret. If searchable encrypted fields are
required, document the leakage profile and prefer server-side tokenization
where appropriate.

## Backup, restore, and uninstall behavior

Choose one policy per secret:

- **Non-restorable:** exclude ciphertext and metadata from Auto Backup; require
  re-authentication or server re-enrollment after restore.
- **Restorable:** restore encrypted data only through a designed key migration
  or recovery mechanism; never assume an Android Keystore key survives device
  transfer, uninstall, factory reset, or a different profile.
- **Server-recoverable:** store only a revocable, short-lived token locally and
  reissue it after attestation or user authentication.

Test `adb backup`/Auto Backup behavior on supported API levels, cloud restore,
device-to-device transfer, uninstall/reinstall, work-profile removal, locked
boot, and a Keystore key invalidated by biometric enrollment changes. A
`KeyPermanentlyInvalidatedException` is a state transition requiring a clear
product decision; deleting the alias and silently losing the account is not a
safe default.

## Migration from EncryptedSharedPreferences / EncryptedFile

1. Freeze the old format and record its exact alias, filename, keyset location,
   algorithm, and backup exclusions.
2. On first run after upgrade, read the old value through the old API in a
   controlled migration path.
3. Validate type, length, schema, and any server-side binding before accepting
   it.
4. Re-encrypt into the new versioned envelope with a new alias or DEK.
5. `fsync`/atomically replace the destination and verify a read-back.
6. Only then remove the old ciphertext and old keyset; retain rollback rules
   until the migration is proven in the field.

Do not use a blanket “catch `Exception`, return empty string” migration. Treat
authentication failure, malformed data, missing keys, and corruption as
different telemetry-safe outcomes. Never log ciphertext, key aliases together
with secret names, exception dumps containing plaintext, or migration payloads.

## Review checklist

- [ ] Keystore key uses AES-GCM or another reviewed AEAD construction.
- [ ] Every encryption gets a unique nonce; nonce and tag are stored with the
      ciphertext, not reused as a password or key.
- [ ] Associated data binds the record type, app/profile context, and format
      version.
- [ ] No key, plaintext, or decrypted byte array reaches logs, analytics,
      clipboard, backups, screenshots, or exported intents.
- [ ] Files are private, writes are atomic, and corruption is detected.
- [ ] Backup/restore, uninstall, profile transfer, and key invalidation have
      explicit product behavior.
- [ ] Deprecated AndroidX crypto wrappers are not added to new code; existing
      use has a tested migration path.
- [ ] Tests cover wrong AAD, modified nonce/ciphertext/tag, truncation,
      concurrent initialization, process death, and rotation.
- [ ] Server sessions remain revocable and do not depend solely on local
      encryption for authorization.

## References

- [Android Keystore system](https://developer.android.com/privacy-and-security/keystore)
- [AndroidX `EncryptedSharedPreferences` reference](https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences)
- [AndroidX `EncryptedFile` reference](https://developer.android.com/reference/androidx/security/crypto/EncryptedFile)
- [AndroidX security-crypto release notes](https://developer.android.com/jetpack/androidx/releases/security)
- [Android data backup documentation](https://developer.android.com/identity/data/autobackup)

