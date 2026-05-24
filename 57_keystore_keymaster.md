# Keystore & Keymaster — Android Crypto Key Management

## 1. Tổng quan

Android Keystore lưu cryptographic keys trong phần cứng bảo mật — keys không bao giờ rời TEE/StrongBox ở dạng plaintext.

```
App (KeyPairGenerator, Cipher, Signature)
  │ Java KeyStore API
  ▼
Keystore System Service (keystore2 daemon)
  │ IKeystoreService Binder
  ▼
Keymaster HAL (IKeyMintDevice AIDL)
  │ Trusty IPC (TIPC) hoặc HW-backed
  ▼
Keymaster Trusted Application (trong Trusty TEE)
  │ ARM TrustZone EL1 (secure world)
  ▼
Hardware: TPM / StrongBox / ARM TrustZone
```

---

## 2. Key Protection Levels

```
Security levels (from strongest to weakest):

StrongBox (SECURITY_LEVEL_STRONGBOX):
  Separate secure element (dedicated chip)
  Tamper-resistant hardware
  Certified: CC EAL4+, FIPS 140-2 Level 3
  Slower (separate processor)
  Not on Pi4

TrustZone (SECURITY_LEVEL_TRUSTED_ENVIRONMENT):
  ARM TrustZone (Trusty TEE on Android)
  Keys in secure world, never exposed to normal world
  Fast (same SoC, different execution state)
  Pi4: has TrustZone but Keymaster TA depends on build

Software (SECURITY_LEVEL_SOFTWARE):
  Keys in keystore2 process memory
  Protected by Linux DAC/SELinux only
  Pi4: fallback if TEE not fully configured
```

---

## 3. Generate Keys

```java
// Generate RSA key pair in Android Keystore
KeyPairGenerator kpg = KeyPairGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_RSA, "AndroidKeyStore");

kpg.initialize(new KeyGenParameterSpec.Builder(
    "my_key_alias",
    KeyProperties.PURPOSE_SIGN | KeyProperties.PURPOSE_VERIFY)
    .setKeySize(2048)
    .setDigests(KeyProperties.DIGEST_SHA256)
    .setSignaturePaddings(KeyProperties.SIGNATURE_PADDING_RSA_PKCS1)
    // Require user authentication (fingerprint / PIN)
    .setUserAuthenticationRequired(true)
    .setUserAuthenticationValidityDurationSeconds(30)
    // Require device to be unlocked
    .setUnlockedDeviceRequired(true)
    .build());

KeyPair keyPair = kpg.generateKeyPair();
// Private key stays in TEE — never leaves hardware!
// Public key accessible as X.509 cert
```

```java
// Generate AES key for encryption
KeyGenerator kg = KeyGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore");

kg.init(new KeyGenParameterSpec.Builder(
    "aes_key",
    KeyProperties.PURPOSE_ENCRYPT | KeyProperties.PURPOSE_DECRYPT)
    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
    .setKeySize(256)
    .build());

SecretKey key = kg.generateKey();
```

---

## 4. Use Keys for Crypto Operations

```java
// Sign data with stored private key
KeyStore ks = KeyStore.getInstance("AndroidKeyStore");
ks.load(null);

PrivateKey privateKey = (PrivateKey) ks.getKey("my_key_alias", null);

Signature signer = Signature.getInstance("SHA256withRSA");
signer.initSign(privateKey);
signer.update(dataToSign);
byte[] signature = signer.sign();
// Sign operation happens INSIDE TEE — key never exported

// Verify
PublicKey publicKey = ks.getCertificate("my_key_alias").getPublicKey();
Signature verifier = Signature.getInstance("SHA256withRSA");
verifier.initVerify(publicKey);
verifier.update(dataToSign);
boolean valid = verifier.verify(signature);
```

```java
// AES-GCM encryption
KeyStore ks = KeyStore.getInstance("AndroidKeyStore");
ks.load(null);
SecretKey key = (SecretKey) ks.getKey("aes_key", null);

Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
cipher.init(Cipher.ENCRYPT_MODE, key);
byte[] iv = cipher.getIV();        // save IV for decryption
byte[] ciphertext = cipher.doFinal(plaintext);
```

---

## 5. Key Attestation

```java
// Key Attestation: cryptographic proof that key is in hardware
// Certificate chain: Key cert → Google cert → Root (Google)
// Verifiable by remote party (server)

KeyPairGenerator kpg = KeyPairGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_EC, "AndroidKeyStore");

kpg.initialize(new KeyGenParameterSpec.Builder("attested_key",
    KeyProperties.PURPOSE_SIGN)
    .setAlgorithmParameterSpec(new ECGenParameterSpec("secp256r1"))
    .setDigests(KeyProperties.DIGEST_SHA256)
    .setAttestationChallenge(serverChallenge) // nonce from server
    .build());

KeyPair kp = kpg.generateKeyPair();

// Get attestation certificate chain
Certificate[] chain = ks.getCertificateChain("attested_key");
// chain[0] = leaf cert (contains attestation extension)
// chain[1] = intermediate (Google Attestation CA)
// chain[2] = root (Google Root CA)

// Parse attestation extension (OID 1.3.6.1.4.1.11129.2.1.17)
// Contains: securityLevel, keyPurpose, attestationChallenge, ...
```

---

## 6. keystore2 Daemon

```
keystore2 (new in Android 12, replaces keystore):
  /system/bin/keystore2
  
  Responsibilities:
    Key storage: /data/misc/keystore/ (encrypted blobs)
    Access control: per-app, per-key permissions
    IKeystoreService Binder interface
    Delegates crypto to Keymaster HAL
    
  Key blob format:
    Encrypted with Keymaster-generated key
    metadata (alias, purpose, auth requirements)
    
  Authentication binding:
    Keys can require: FINGERPRINT / PIN / PATTERN
    AuthToken generated by BiometricService after auth
    Passed to Keymaster with operation
```

---

## 7. Pi4 Keymaster

```
Pi4 (without StrongBox):
  Trusty OS runs in ARM TrustZone
  Keymaster TA in Trusty (see 33_trustzone_tee_trusty.md)
  
  If Trusty not running → Software Keymaster fallback:
    SECURITY_LEVEL_SOFTWARE
    Keys protected by keystore2 daemon (Normal World)
    Less secure but functional
    
Check:
adb shell keystore2_client get-attestation-status
# Or via Java:
// keySpec.getSecurityLevel() == SecurityLevel.TRUSTED_ENVIRONMENT
```

---

## 8. BiometricPrompt + KeyStore

```java
// Use key with biometric authentication
// Create key that requires biometric auth
kpg.initialize(new KeyGenParameterSpec.Builder("bio_key",
    KeyProperties.PURPOSE_SIGN)
    .setUserAuthenticationRequired(true)
    .setUserAuthenticationParameters(
        0,  // 0 = require auth for every use
        KeyProperties.AUTH_BIOMETRIC_STRONG)
    .build());

// Show biometric prompt when using key
BiometricPrompt prompt = new BiometricPrompt(activity, executor, callback);
BiometricPrompt.PromptInfo promptInfo = new BiometricPrompt.PromptInfo.Builder()
    .setTitle("Sign document")
    .setNegativeButtonText("Cancel")
    .build();

// Create Cipher pre-auth, finalize in callback
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
cipher.init(Cipher.ENCRYPT_MODE, secretKey);
BiometricPrompt.CryptoObject cryptoObject =
    new BiometricPrompt.CryptoObject(cipher);

prompt.authenticate(promptInfo, cryptoObject);
// callback.onAuthenticationSucceeded() → cipher is ready to use
```

---

## 9. Debug Keystore

```bash
# List keys in keystore
adb shell keystore2_client list --user 0
# alias       uid   securityLevel
# my_key      10123 TrustedEnvironment
# aes_key     10123 TrustedEnvironment

# Delete key
adb shell keystore2_client delete my_key_alias

# Keystore logs
adb logcat -s keystore2 KeyMint Keymaster

# Dump keystore state
adb shell dumpsys keystore

# Check TEE attestation
adb shell keystore2_client get-attestation-chain my_key_alias

# Verify KeyMint HAL is running
adb shell service check android.hardware.security.keymint.IKeyMintDevice/default
# Service android.hardware.security.keymint.IKeyMintDevice/default found
```

---

## 10. FIDO2 / Passkeys (Android 9+)

```
Android Keystore powers FIDO2/WebAuthn:
  Passkeys = phishing-resistant password replacement
  
  Registration:
    Device generates EC key pair (P-256) in Keystore
    Public key sent to server
    
  Authentication:
    Server sends challenge
    Device signs with private key (requires user auth)
    → challenge response verified by server
    
  Backed by:
    FIDO2 Authenticator → CTAP2 → Keystore
    Biometric required for each sign operation
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `system/security/keystore2/` | keystore2 daemon source |
| `system/security/keystore2/src/service.rs` | Keystore2 service (Rust!) |
| `hardware/interfaces/security/keymint/aidl/` | KeyMint AIDL HAL |
| `system/keymint/` | Software KeyMint implementation |
| `trusty/app/keymaster/` | Trusty Keymaster TA |
| `frameworks/base/keystore/java/android/security/keystore2/` | Java Keystore provider |
