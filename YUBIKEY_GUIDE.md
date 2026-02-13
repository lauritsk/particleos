# os
Personal ParticleOS System Setup

ParticleOS Secure Boot Signing with YubiKey (macOS)

This guide covers:

Installing required macOS tools

Generating an ECC P-384 signing key

Creating encrypted offline backup

Importing the same key into two YubiKeys

Verifying correctness

Restoring from backup

Confirming PKCS#11 works for mkosi

Requirements (macOS)

Install required tools via Homebrew:

brew install yubikey-manager opensc openssl


Verify installation:

ykman --version
pkcs11-tool --version
openssl version


OpenSC PKCS#11 module path (Apple Silicon):

/opt/homebrew/lib/opensc-pkcs11.so

1. Generate ECC P-384 Signing Key

We use secp384r1 (ECC P-384).

openssl genpkey \
  -algorithm EC \
  -pkeyopt ec_paramgen_curve:secp384r1 \
  -pkeyopt ec_param_enc:named_curve \
  -out mkosi.key

chmod 600 mkosi.key


Verify:

openssl pkey -in mkosi.key -text -noout | grep secp384r1


You should see:

ASN1 OID: secp384r1

2. Create Signing Certificate
openssl req \
  -new -x509 \
  -key mkosi.key \
  -subj "/CN=mkosi" \
  -days 3650 \
  -out mkosi.crt

3. Create Encrypted Offline Backup (CRITICAL)

Encrypt private key:

openssl enc -aes-256-cbc -pbkdf2 \
  -in mkosi.key \
  -out mkosi.key.enc


Then securely remove plaintext key:

shred -u mkosi.key


Store offline:

mkosi.key.enc

mkosi.crt

Recommended storage:

Encrypted USB stored physically secure

Password manager attachment

Offline encrypted archive

4. Prepare Each YubiKey

Insert YubiKey #1.

Change Default PIN

(Default is 123456)

ykman piv access change-pin

Change Default PUK (Recommended)
ykman piv access change-puk

Change Management Key (Very Important)
ykman piv access change-management-key --generate --protect


--protect ties the management key to your PIN.

Repeat this entire section for YubiKey #2.

5. Import Key into Slot 9c (Digital Signature)

Temporarily decrypt backup:

openssl enc -d -aes-256-cbc -pbkdf2 \
  -in mkosi.key.enc \
  -out mkosi.key


Import to slot 9c:

ykman piv keys import 9c mkosi.key
ykman piv certificates import 9c mkosi.crt


Securely remove plaintext key again:

shred -u mkosi.key


Repeat for YubiKey #2.

Now both tokens contain the exact same private key.

6. Verify Key Algorithm (ECC P-384)

Export certificate:

ykman piv certificates export 9c cert.pem


Verify algorithm:

openssl x509 -in cert.pem -text -noout | grep -A2 "Public Key Algorithm"


Expected output:

Public Key Algorithm: id-ecPublicKey
ASN1 OID: secp384r1

7. Verify Both YubiKeys Contain Identical Key

With YubiKey #1 inserted:

openssl x509 -in cert.pem -noout -fingerprint -sha256


Copy fingerprint.

Insert YubiKey #2 and repeat.

Fingerprints must match exactly.

If they match → both tokens contain the same key.

8. Verify PKCS#11 Access (Required for mkosi)

List certificate objects:

pkcs11-tool \
  --module /opt/homebrew/lib/opensc-pkcs11.so \
  --list-objects --type cert


Expected output includes:

token=mkosi

ID: 02

URI line

Example:

token=mkosi;id=%02;type=cert

9. Test Private Key Signing (Important)

Create test file:

echo "particleos test" > test.txt


Sign:

pkcs11-tool \
  --module /opt/homebrew/lib/opensc-pkcs11.so \
  --sign \
  --id 02 \
  --mechanism ECDSA \
  --input-file test.txt \
  --output-file sig.bin


Verify:

openssl dgst -sha384 \
  -verify <(openssl x509 -in cert.pem -pubkey -noout) \
  -signature sig.bin \
  test.txt


Expected result:

Verified OK


This confirms:

Private key works

ECC P-384 signing works

PKCS#11 works

mkosi will work

10. mkosi.local.conf Configuration

Based on:

token=mkosi
id=02


Use:

[Validation]
SecureBootKey=pkcs11:token=mkosi;id=%%02;type=private
SecureBootKeySource=provider:pkcs11
SecureBootCertificate=pkcs11:token=mkosi;id=%%02;type=cert
SecureBootCertificateSource=provider:pkcs11


Note: %%02 (double %) is required.

Restoring From Backup

If a YubiKey is lost or damaged:

Decrypt backup:

openssl enc -d -aes-256-cbc -pbkdf2 \
  -in mkosi.key.enc \
  -out mkosi.key


Import to replacement YubiKey:

ykman piv keys import 9c mkosi.key
ykman piv certificates import 9c mkosi.crt


Securely remove plaintext:

shred -u mkosi.key


The new token will now behave identically.

Security Checklist

✔ ECC P-384 used
✔ Same key on both YubiKeys
✔ Encrypted offline backup exists
✔ Plaintext key shredded
✔ PIN changed
✔ Management key replaced
✔ PKCS#11 verified
✔ Signing tested

Optional Hardening (Advanced)

Generate key inside a RAM disk

Store backup in encrypted archive (e.g. age or GPG)

Maintain third sealed backup token

Enroll certificate into UEFI Secure Boot chain


----------


Excellent — now we’re moving into serious territory.

Below are two sections you can append to your README:

Hardened “no-plaintext-on-disk” workflow (macOS)

Secure Boot disaster recovery model (real-world operational plan)

This assumes:

ECC P-384

Two YubiKeys

You control Secure Boot keys (custom PK/KEK/db)

🔐 Hardened Key Generation (No Plaintext on Disk)

Goal:
The private key never touches persistent storage unencrypted.

We’ll use a RAM disk on macOS.

1️⃣ Create Temporary RAM Disk

Create 16MB RAM disk:

RAMDISK=$(hdiutil attach -nomount ram://32768)
diskutil erasevolume APFS RAMDisk $RAMDISK


Mount point will be:

/Volumes/RAMDisk


All key material will exist only here.

2️⃣ Generate ECC P-384 Key in RAM
cd /Volumes/RAMDisk

openssl genpkey \
  -algorithm EC \
  -pkeyopt ec_paramgen_curve:secp384r1 \
  -pkeyopt ec_param_enc:named_curve \
  -out mkosi.key

3️⃣ Create Certificate
openssl req \
  -new -x509 \
  -key mkosi.key \
  -subj "/CN=mkosi" \
  -days 3650 \
  -out mkosi.crt

4️⃣ Immediately Create Encrypted Backup (to persistent disk)

Choose a secure destination (external encrypted drive recommended):

openssl enc -aes-256-cbc -pbkdf2 \
  -in mkosi.key \
  -out ~/secure-storage/mkosi.key.enc


Copy certificate:

cp mkosi.crt ~/secure-storage/


Now the only plaintext key is in RAM.

5️⃣ Import into YubiKey #1
ykman piv keys import 9c mkosi.key
ykman piv certificates import 9c mkosi.crt

6️⃣ Import into YubiKey #2

Insert second key and repeat:

ykman piv keys import 9c mkosi.key
ykman piv certificates import 9c mkosi.crt

7️⃣ Destroy RAM Disk (Key Gone Forever)
cd ~
hdiutil detach /Volumes/RAMDisk


At this point:

Private key never existed unencrypted on disk

Only encrypted backup remains

Both YubiKeys contain identical key

This is extremely clean operationally.

🧨 Secure Boot Disaster Recovery Model

Now the important part: what happens if:

You lose one YubiKey?

You lose both?

A token fails?

You rotate keys?

Firmware update bricks db?

You need a trust hierarchy model.

🔐 Recommended Secure Boot Architecture

Think in 3 tiers:

PK   (Platform Key – Root of Trust)
 └── KEK (Key Exchange Key)
      └── db (Allowed Signing Keys)
            └── mkosi signing cert


Your YubiKey holds the mkosi signing key (db-level signing key).

It should NOT hold:

PK

KEK

Those should be offline-only.

🛡 Recommended Key Roles
Key	Where Stored	Usage
PK	Offline encrypted storage only	Own platform
KEK	Offline encrypted storage only	Authorize db updates
mkosi signing key	YubiKey (x2)	Sign OS images

Never mix these roles.

🔄 Disaster Scenarios
🟢 Scenario 1: One YubiKey Lost

Nothing breaks.

Use second YubiKey

Immediately provision replacement

Restore from mkosi.key.enc

Procedure:

Decrypt backup

Import to new YubiKey

Destroy plaintext

Zero Secure Boot changes required.

🟡 Scenario 2: Both YubiKeys Lost BUT Backup Exists

Still safe.

Restore key from encrypted backup

Provision new YubiKeys

Continue signing

Secure Boot chain unaffected.

🔴 Scenario 3: Backup Lost + Both YubiKeys Lost

Now signing key is gone permanently.

Systems already enrolled with that db certificate will:

Continue booting old images

Refuse new images signed with new key

Recovery requires:

Generate new signing key

Use KEK to update db

Enroll new signing certificate

This is why KEK must be preserved offline.

🧠 Recommended Recovery Strategy

Keep:

1 encrypted backup of mkosi.key

1 offline copy of KEK private key

1 offline copy of PK private key

Store in:

Separate physical locations

Encrypted storage

Printed fingerprint sheet in safe

🔁 Key Rotation Model (Professional Setup)

Every 2–3 years:

Generate new mkosi signing key

Sign new images with both old + new keys temporarily

Update db to include new key

Remove old key from db

Destroy old key material

This avoids hard cutovers.

🔐 Maximum Paranoia Mode (Optional)

For high assurance:

Third YubiKey stored offsite

KEK stored in separate HSM

PK stored offline air-gapped

Shamir Secret Sharing for PK backup

Maintain printed SHA256 fingerprints

🧱 Practical Minimal Safe Setup

For most secure homelab / production setups:

✔ 2 signing YubiKeys
✔ 1 encrypted backup
✔ Offline KEK private key
✔ Offline PK private key
✔ Printed fingerprints
✔ Secure storage in separate locations

That gives you:

Hardware compromise resistance

Disaster survivability

Recoverability

No single point of failure



------------



🔐 Recommended Secure Boot Key Repository Layout

This repository should live:

On an encrypted external drive

Or inside an encrypted vault (VeraCrypt / LUKS / FileVault disk image)

Never in your normal home directory

Example root:

secureboot/

📂 Top-Level Structure
secureboot/
├── README.md
├── fingerprints/
├── pk/
├── kek/
├── db/
├── backups/
├── revoked/
└── rotation/

📄 README.md

Human-readable document describing:

Key purpose

Creation dates

Expiry dates

Where physical backups are stored

Recovery procedure summary

This becomes critical in 3 years.

🔎 fingerprints/

Contains printable, immutable references.

fingerprints/
├── pk.sha256
├── kek.sha256
├── db.sha256
├── mkosi_signing.sha256


Generate like:

openssl x509 -in db/db.crt -noout -fingerprint -sha256 > fingerprints/db.sha256


Print these and store physically.

👑 pk/ (Platform Key – Root of Trust)

Highest authority. Rarely used.

pk/
├── private/
│   └── pk.key.enc
├── certs/
│   └── pk.crt
├── esl/
│   └── pk.esl
└── auth/
    └── pk.auth


Rules:

pk.key.enc = encrypted private key only

Never keep plaintext pk.key

This key should almost never be decrypted

🔑 kek/ (Key Exchange Key)

Used to authorize db updates.

kek/
├── private/
│   └── kek.key.enc
├── certs/
│   └── kek.crt
├── esl/
│   └── kek.esl
└── auth/
    └── kek.auth


Used only when:

Rotating db keys

Adding new signing keys

Revoking compromised ones

🖊 db/ (Allowed Signing Keys)

Contains OS signing certificates.

db/
├── active/
│   └── mkosi_signing.crt
├── private-backup/
│   └── mkosi.key.enc
├── esl/
│   └── db.esl
└── auth/
    └── db.auth


Important:

mkosi_signing.crt = certificate enrolled in firmware

mkosi.key.enc = encrypted backup of YubiKey key

No plaintext private keys here

🚫 revoked/

For compromised keys.

revoked/
├── old_mkosi_signing.crt
├── dbx.esl
└── dbx.auth


If you rotate keys, archive old certs here.

Never delete revoked material.

🔁 rotation/

For future key rollovers.

rotation/
├── 2026-rotation/
│   ├── new_db.crt
│   ├── transition_notes.md
│   └── timeline.md


Keep rotation isolated and documented.

💾 backups/

Metadata about physical storage locations.

backups/
├── storage-locations.md
├── recovery-playbook.md


Example storage-locations.md:

Encrypted USB #1 – Safe at home
Encrypted USB #2 – Bank safe deposit box
YubiKey #1 – Daily carry
YubiKey #2 – Fireproof safe

🔒 Encryption Standard Recommendation

Always encrypt private keys like this:

openssl enc -aes-256-cbc -pbkdf2 \
  -in pk.key \
  -out pk.key.enc


Then:

shred -u pk.key


Never store plaintext.

🧠 Separation of Duties Model

Best practice:

Key	Storage	Usage Frequency
PK	Offline only	Almost never
KEK	Offline only	Rare
db signing	YubiKey	Regular
db backup	Encrypted file	Emergency only

Never keep PK, KEK, and db private keys on same device.

📌 Practical Minimal Setup

If you're running ParticleOS for serious systems:

2 signing YubiKeys

1 encrypted signing key backup

Offline KEK key

Offline PK key

Printed fingerprints

Encrypted storage in two locations

This eliminates single-point-of-failure risk.

🧱 Example Final Layout Snapshot
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


Clean. Auditable. Recoverable.
