# Course content encryption

This repository can store **course content encrypted at rest** on GitHub while leaving the **navigation structure** readable. Clients unlock content with a password; helper scripts for authors live locally under `.helper/` (gitignored).

## Goals

- Protect lesson bodies and flashcard text from casual browsing of the repo.
- Keep TOC / IDs / paths / XP in `library.xml` plaintext so the app can still show the outline before unlock.
- Allow the client to **reject a wrong password quickly** via a fingerprint in `library.xml`, before decrypting hundreds of files.

This is **not** DRM against a determined attacker with the ciphertext and unlimited offline guessing. Use a strong password.

## What is encrypted vs plaintext

| Path | Encrypted? |
|------|------------|
| `library.xml` | **No** (structure + encryption metadata only) |
| `flashcard_definition.xml` | **No** |
| `icons.xml` / icon assets | **No** |
| `vocabulary_list_map.xml` | **No** |
| `lessons/**/*.xml` | **Yes** |
| `vocabulary.xml` | **Yes** |

Encrypted files keep their original relative paths (as referenced from `library.xml`). On disk they are binary envelopes, not plaintext XML.

## `library.xml` markers

An encrypted course sets attributes on the `<course>` element:

| Attribute | Meaning |
|-----------|---------|
| `encrypted="true"` | Client must unlock before loading content files |
| `encryptionVersion` | Envelope / scheme version (`1`) |
| `kdf` | Key derivation function (`argon2id`) |
| `kdfSalt` | Base64 salt for Argon2id |
| `kdfMemory` | Argon2 memory cost (KiB) |
| `kdfIterations` | Argon2 time parameter |
| `kdfParallelism` | Argon2 lanes / parallelism |
| `contentFingerprint` | Base64 password verifier (see below) |

Courses without these attributes (or with `encrypted="false"`) are open plaintext, e.g. public language courses.

## Cryptography (version 1)

1. **Key derivation**  
   `key = Argon2id(password, salt, memory, iterations, parallelism)` → 32 bytes.

2. **Per-file envelope** (`LAN1`):

   ```
   magic "LAN1" (4 bytes)
   version (1 byte) = 1
   nonce (12 bytes, random per file)
   AES-256-GCM ciphertext || 16-byte tag
   ```

   Associated data (AAD): `"{courseUuid}|{relativePath}"` with forward-slash paths  
   (example: `5da2df3b-…|lessons/01_…/01_einfuehrung.xml`).

3. **Password fingerprint** (`contentFingerprint`)  
   AES-256-GCM encrypt of the UTF-8 string:

   ```
   LAN-COURSE-FP|{courseUuid}
   ```

   with AAD `fingerprint`, random 12-byte nonce. Stored as Base64(`nonce || ciphertext || tag`).

   Unlock flow: derive key → decrypt fingerprint → must equal `LAN-COURSE-FP|{uuid}`. Wrong password fails here without touching lesson files.

## Client unlock sketch

1. Parse `library.xml`. If `encrypted` is not true, load content as today.
2. Prompt for password; read salt and KDF params; derive `key`.
3. Verify `contentFingerprint`.
4. On success, keep `key` in memory for the session.
5. When opening a lesson or vocabulary: read file bytes → if `LAN1` envelope, decrypt with AAD `{uuid}|{relPath}` → parse XML.

Tampered or swapped files fail the GCM tag check.

## Author workflow (local helpers)

Scripts under `.helper/` (not published; install deps with `pip install -r .helper/requirements.txt`):

```text
python .helper/encrypt_course.py course_ms_sc200
python .helper/decrypt_course.py course_ms_sc200
python .helper/verify_password.py course_ms_sc200
```

Password via prompt or `LAN_COURSE_PASSWORD`. Encrypt before committing locked courses; decrypt locally only for editing. Do **not** commit decrypted content for courses that are meant to stay locked.

### Password rotation

Decrypt all content → encrypt again with `--force-new-salt` (new salt + fingerprint) → commit. Old ciphertext becomes obsolete.

## Threat model (summary)

| Protected against | Not protected against |
|-------------------|------------------------|
| Casual GitHub viewers | Offline brute-force of a weak password |
| Accidental plaintext leaks in PRs for content files | Disclosure of titles/structure in `library.xml` |
| Bit-flip / file-swap without the key (GCM) | Insiders who know the password |

Passwords must never be committed to this repository.
