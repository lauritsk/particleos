# YubiKey Secure Boot Signing for ParticleOS (macOS)

This guide documents a practical, repo-aligned setup for using a YubiKey PIV key
with `mkosi` Secure Boot signing in ParticleOS.

It covers:
- macOS tooling setup
- ECC P-384 key generation
- encrypted backup and restore
- importing the same signing key into two YubiKeys
- PKCS#11 checks
- `mkosi.local.conf` settings used in this repository
- hardened operational and recovery practices

## Trust model

Use separate key roles:
- `PK` (Platform Key): offline only
- `KEK` (Key Exchange Key): offline only
- `db` signing key (mkosi signing cert): YubiKey PIV slot `9c`

The YubiKey in this guide is only for the image-signing key (db-level signing).
Do not store PK or KEK private keys on daily-use tokens.

## Requirements (macOS)

Install required tools:

```sh
brew install yubikey-manager opensc openssl
```

Verify tools:

```sh
ykman --version
pkcs11-tool --version
openssl version
```

Resolve OpenSC PKCS#11 module path:

```sh
OPENSC_P11="$(brew --prefix opensc)/lib/opensc-pkcs11.so"
test -f "$OPENSC_P11" && echo "$OPENSC_P11"
```

On Apple Silicon this is typically:

```text
/opt/homebrew/lib/opensc-pkcs11.so
```

## Standard workflow

### 1) Generate an ECC P-384 signing key

```sh
openssl genpkey \
  -algorithm EC \
  -pkeyopt ec_paramgen_curve:secp384r1 \
  -pkeyopt ec_param_enc:named_curve \
  -out mkosi.key

chmod 600 mkosi.key
```

Verify curve:

```sh
openssl pkey -in mkosi.key -text -noout | grep 'ASN1 OID: secp384r1'
```

### 2) Create signing certificate

```sh
openssl req \
  -new -x509 \
  -key mkosi.key \
  -subj "/CN=mkosi" \
  -days 3650 \
  -out mkosi.crt
```

### 3) Create encrypted offline backup

```sh
openssl enc -aes-256-cbc -pbkdf2 -salt \
  -in mkosi.key \
  -out mkosi.key.enc
```

Store these offline:
- `mkosi.key.enc`
- `mkosi.crt`

Then remove plaintext `mkosi.key`:

```sh
rm -f mkosi.key
```

Note: secure deletion is not guaranteed on APFS/SSD. Use the hardened RAM-disk
workflow below if you want to avoid plaintext key material on persistent storage.

### 4) Prepare each YubiKey

Run this for YubiKey #1, then repeat for YubiKey #2:

```sh
ykman piv access change-pin
ykman piv access change-puk
ykman piv access change-management-key --generate --protect
```

`--protect` ties the management key to the PIN.

### 5) Import key and certificate to slot 9c

Temporarily decrypt:

```sh
openssl enc -d -aes-256-cbc -pbkdf2 \
  -in mkosi.key.enc \
  -out mkosi.key
```

Import to YubiKey #1:

```sh
ykman piv keys import 9c mkosi.key
ykman piv certificates import 9c mkosi.crt
```

Remove plaintext key:

```sh
rm -f mkosi.key
```

Repeat decrypt/import/remove for YubiKey #2.

### 6) Verify certificate and key identity on both YubiKeys

Export cert from YubiKey #1:

```sh
ykman piv certificates export 9c cert.pem
```

Check algorithm:

```sh
openssl x509 -in cert.pem -text -noout | grep -A2 'Public Key Algorithm'
```

Expected:
- `Public Key Algorithm: id-ecPublicKey`
- `ASN1 OID: secp384r1`

Record fingerprint from YubiKey #1:

```sh
openssl x509 -in cert.pem -noout -fingerprint -sha256
```

Insert YubiKey #2, export certificate again, and compare fingerprints. They must
match exactly.

### 7) Verify PKCS#11 visibility

```sh
pkcs11-tool \
  --module "$OPENSC_P11" \
  --list-objects --type cert
```

Confirm output includes:
- token label (for example `token=mkosi`)
- key ID `02`
- URI with `id=%02`

If your URI has a different token label, use that value in `mkosi.local.conf`
instead of `mkosi`.

### 8) Optional signing smoke test

Create test data and digest:

```sh
echo 'particleos test' > test.txt
openssl dgst -sha384 -binary test.txt > test.sha384
```

Sign digest with YubiKey private key (`id 02`):

```sh
pkcs11-tool \
  --module "$OPENSC_P11" \
  --login \
  --sign \
  --id 02 \
  --mechanism ECDSA \
  --input-file test.sha384 \
  --output-file sig.bin
```

For P-384, raw ECDSA signature output is typically 96 bytes:

```sh
wc -c sig.bin
```

Clean test artifacts:

```sh
rm -f test.txt test.sha384 sig.bin cert.pem
```

## `mkosi.local.conf` settings for this repo

This repository uses full PKCS#11 settings for Secure Boot, PCR signing, and
verity signing:

```conf
[Validation]
SecureBootKey=pkcs11:token=mkosi;id=%%02;type=private
SecureBootKeySource=provider:pkcs11
SecureBootCertificate=pkcs11:token=mkosi;id=%%02;type=cert
SecureBootCertificateSource=provider:pkcs11
SignExpectedPcrKey=pkcs11:token=mkosi;id=%%02;type=private
SignExpectedPcrKeySource=provider:pkcs11
SignExpectedPcrCertificate=pkcs11:token=mkosi;id=%%02;type=cert
SignExpectedPcrCertificateSource=provider:pkcs11
VerityKey=pkcs11:token=mkosi;id=%%02;type=private
VerityKeySource=provider:pkcs11
VerityCertificate=pkcs11:token=mkosi;id=%%02;type=cert
VerityCertificateSource=provider:pkcs11
```

Important: `%%02` (double `%`) is required in `mkosi.local.conf`.

## Restore from backup

If a token is lost or damaged:

```sh
openssl enc -d -aes-256-cbc -pbkdf2 \
  -in mkosi.key.enc \
  -out mkosi.key

ykman piv keys import 9c mkosi.key
ykman piv certificates import 9c mkosi.crt

rm -f mkosi.key
```

This restores the same signing identity on the replacement token.

## Hardened workflow: no plaintext key on persistent disk

Use a RAM disk so plaintext private key material does not persist on disk.

### 1) Create RAM disk

```sh
RAMDISK=$(hdiutil attach -nomount ram://32768)
diskutil erasevolume APFS RAMDisk "$RAMDISK"
```

### 2) Generate key and cert on RAM disk

```sh
cd /Volumes/RAMDisk

openssl genpkey \
  -algorithm EC \
  -pkeyopt ec_paramgen_curve:secp384r1 \
  -pkeyopt ec_param_enc:named_curve \
  -out mkosi.key

openssl req \
  -new -x509 \
  -key mkosi.key \
  -subj "/CN=mkosi" \
  -days 3650 \
  -out mkosi.crt
```

### 3) Save encrypted backup and certificate to secure storage

```sh
mkdir -p ~/secure-storage

openssl enc -aes-256-cbc -pbkdf2 -salt \
  -in mkosi.key \
  -out ~/secure-storage/mkosi.key.enc

cp mkosi.crt ~/secure-storage/
```

### 4) Import to YubiKey #1 and #2

```sh
ykman piv keys import 9c mkosi.key
ykman piv certificates import 9c mkosi.crt
```

Swap token and repeat.

### 5) Destroy RAM disk

```sh
cd ~
hdiutil detach /Volumes/RAMDisk
```

## Disaster recovery model

### Scenario 1: one YubiKey lost

No Secure Boot enrollment change is required.
- continue signing with remaining token
- provision replacement from `mkosi.key.enc`

### Scenario 2: both YubiKeys lost, backup exists

No Secure Boot enrollment change is required.
- restore from `mkosi.key.enc`
- import into replacement token(s)

### Scenario 3: both YubiKeys lost, backup lost

Signing identity is permanently lost.
- generate new signing key
- use offline KEK to update `db`
- enroll new signing certificate

## Key rotation policy (recommended)

Rotate signing keys every 2-3 years:
- generate new db signing key
- temporarily trust old + new certs in `db`
- migrate builds to new key
- remove old cert from `db`
- archive and revoke old material as appropriate

## Recommended secure boot key repository layout

Store this on encrypted removable media or encrypted vault storage, not in your
normal home directory.

```text
secureboot/
├── README.md
├── fingerprints/
├── pk/
├── kek/
├── db/
├── backups/
├── revoked/
└── rotation/
```

Suggested structure:

```text
secureboot/
├── pk/
│   ├── private/pk.key.enc
│   ├── certs/pk.crt
│   ├── esl/pk.esl
│   └── auth/pk.auth
├── kek/
│   ├── private/kek.key.enc
│   ├── certs/kek.crt
│   ├── esl/kek.esl
│   └── auth/kek.auth
├── db/
│   ├── active/mkosi_signing.crt
│   ├── private-backup/mkosi.key.enc
│   ├── esl/db.esl
│   └── auth/db.auth
├── fingerprints/
├── revoked/
├── rotation/
└── backups/
```

Recommended fingerprint records:
- `fingerprints/pk.sha256`
- `fingerprints/kek.sha256`
- `fingerprints/db.sha256`
- `fingerprints/mkosi_signing.sha256`

Example:

```sh
openssl x509 -in db/active/mkosi_signing.crt -noout -fingerprint -sha256 \
  > fingerprints/mkosi_signing.sha256
```

## Minimal safe baseline

- two signing YubiKeys with identical key material
- encrypted backup of signing key (`mkosi.key.enc`)
- offline KEK private key backup
- offline PK private key backup
- printed SHA-256 fingerprints stored separately
- backups stored in at least two physical locations
