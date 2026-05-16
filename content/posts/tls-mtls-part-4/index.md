---
title: "TLS & mTLS — Part 4: The Secure Enclave"
date: 2026-04-28
draft: true
tags: ["security", "tls", "mtls", "ios", "secure-enclave", "frida"]
series: ["TLS & mTLS"]
series_order: 4
summary: "We fix what the crow broke. The Secure Enclave generates a key that never leaves the chip — and Autolycus finally meets a lock he can't pick."
---

{{% dialog "🦆 Nestor" %}}
So the key lives in a tiny vault inside the chip?
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
Not quite *in* the chip. The Secure Enclave (SE) is a **separate processor** — its own CPU, its own encrypted memory, its own firmware — physically isolated from the main application processor. It was introduced with the A7 chip (iPhone 5S) and has been in every iPhone since. Our iPhone 6S has an A9, so it has a fully capable Secure Enclave.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
A separate processor? Like a whole other computer?
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
A very specialized one. It handles cryptographic operations — key generation, signing, encryption — and stores keys in a way that the main processor can never access directly. When your app asks the SE to sign data, the request goes through a hardware mailbox. The data goes in, the signature comes out. The private key stays inside.
{{% /dialog %}}

{{% dialog "🐦‍⬛ Autolycus" %}}
*Lands on the desk.* I heard there's a new lock to pick.
{{% /dialog %}}

You'll get your turn. First, let's build it.

## The plan

In Part 2, we bundled a `.p12` file and imported it into the Keychain. The private key was software — bytes in memory, extractable, stealable. In Part 3, Autolycus proved it.

Now we're going to do things differently:

1. **Generate a key pair** inside the Secure Enclave — the private key never leaves
2. **Build a Certificate Signing Request (CSR)** using that key
3. **Send the CSR** to our server's new provisioning endpoint
4. **Store the signed certificate** in the Keychain, linked to the SE key
5. **Connect with mTLS** using a hardware-backed identity

{{% dialog "🦆 Nestor" %}}
Wait — we're building a CSR? Like the ones from Part 1?
{{% /dialog %}}

Exactly the same concept. In Part 1, we used `openssl req` to create CSRs on the command line. Now the iPhone is going to do the same thing — but the signing happens inside the Secure Enclave.

{{% dialog "🦉 Menthor" %}}
*Opens a thick book.* This is, incidentally, how the industry protects CA private keys on the server side — using Hardware Security Modules (HSMs). The Secure Enclave is essentially an HSM built into the phone. Same principle: the key is generated inside tamper-resistant hardware and never leaves. The only interface is "sign this data" and "here's your signature."
{{% /dialog %}}

## Step 1: Generating a key in the Secure Enclave

```swift
private func generateSecureEnclaveKey() throws -> SecKey {
    var error: Unmanaged<CFError>?

    let access = SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        .privateKeyUsage,
        &error
    )!

    let attributes: [String: Any] = [
        kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
        kSecAttrKeySizeInBits as String: 256,
        kSecAttrTokenID as String: kSecAttrTokenIDSecureEnclave,
        kSecPrivateKeyAttrs as String: [
            kSecAttrIsPermanent as String: true,
            kSecAttrAccessControl as String: access,
            kSecAttrLabel as String: "com.demo.TLSDemo.se-key"
        ]
    ]

    guard let privateKey = SecKeyCreateRandomKey(attributes as CFDictionary, &error) else {
        throw error!.takeRetainedValue() as Error
    }

    return privateKey
}
```

{{% dialog "🦆 Nestor" %}}
That looks like the Keychain code from Part 2, but with extra flags.
{{% /dialog %}}

The critical differences:

| Flag | What it does |
|------|-------------|
| `kSecAttrTokenIDSecureEnclave` | Generate the key *inside the SE*, not in software |
| `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | Key is never included in backups or synced to other devices |
| `.privateKeyUsage` | Key can only be used for signing, not exported |
| `kSecAttrIsPermanent: true` | Key persists in the SE across app launches |

{{% dialog "🦉 Menthor" %}}
Note the constraint: `kSecAttrKeyTypeECSECPrimeRandom` with 256 bits. The Secure Enclave only supports **EC P-256** keys. No RSA, no P-384, no Ed25519. This is a hardware limitation — the SE's cryptographic engine is built specifically for this curve.
{{% /dialog %}}

{{% dialog "🐦‍⬛ Autolycus" %}}
*Examines the code.* `kSecAttrTokenIDSecureEnclave`. That one little flag changes everything, huh?
{{% /dialog %}}

Everything.

### Simulator fallback

The Secure Enclave doesn't exist in the simulator. We need a runtime check:

```swift
static var isSecureEnclaveAvailable: Bool {
    #if targetEnvironment(simulator)
    return false
    #else
    // Try to create a test SE key
    let attributes: [String: Any] = [
        kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
        kSecAttrKeySizeInBits as String: 256,
        kSecAttrTokenID as String: kSecAttrTokenIDSecureEnclave
    ]
    var error: Unmanaged<CFError>?
    let key = SecKeyCreateRandomKey(attributes as CFDictionary, &error)
    if let key = key {
        // Clean up the test key
        let deleteQuery: [String: Any] = [
            kSecClass as String: kSecClassKey,
            kSecValueRef as String: key
        ]
        SecItemDelete(deleteQuery as CFDictionary)
        return true
    }
    return false
    #endif
}
```

The UI shows a badge: **"Secure Enclave (hardware)"** on the real device, **"Software Key (simulator)"** on the simulator.

## Step 2: Building a CSR

{{% dialog "🦆 Nestor" %}}
Can't we just use a library for this?
{{% /dialog %}}

We could. But the CSR is the most interesting part of this whole series — it's where the Secure Enclave meets the certificate infrastructure we built in Part 1. And it's only about 150 lines of code.

A Certificate Signing Request (CSR) is a PKCS#10 structure. It says: "Here's my public key, here's who I am, and here's a signature proving I own the corresponding private key." The CA reads it, verifies the signature, and issues a certificate.

{{% dialog "🦉 Menthor" %}}
The structure, formally defined in RFC 2986, is:

```
CertificationRequest ::= SEQUENCE {
    certificationRequestInfo  CertificationRequestInfo,
    signatureAlgorithm        AlgorithmIdentifier,
    signature                 BIT STRING
}

CertificationRequestInfo ::= SEQUENCE {
    version       INTEGER (v1),
    subject       Name,
    subjectPKInfo SubjectPublicKeyInfo,
    attributes    [0] Attributes
}
```

We need to build this in ASN.1 DER encoding — the same binary format that every X.509 certificate uses.
{{% /dialog %}}

{{% dialog "🐦‍⬛ Autolycus" %}}
You're building ASN.1 by hand? This is going to be fun to watch.
{{% /dialog %}}

### The ASN.1 builder

[PLACEHOLDER: ASN.1 helper code — functions to encode TLV (tag-length-value), SEQUENCE, OID, PrintableString, INTEGER, BIT STRING]

### Assembling the CSR

The key insight: we extract the **public** key from the SE (that's allowed — only the private key is locked away), wrap it in ASN.1, build the request info, then ask the SE to sign the whole thing.

```swift
func buildCSR(privateKey: SecKey, commonName: String) throws -> Data {
    // 1. Get the public key
    guard let publicKey = SecKeyCopyPublicKey(privateKey) else {
        throw CSRError.noPublicKey
    }
    guard let publicKeyData = SecKeyCopyExternalRepresentation(publicKey, nil) as Data? else {
        throw CSRError.cantExportPublicKey
    }

    // 2. Build CertificationRequestInfo
    let subject = buildSubjectDN(commonName: commonName)
    let spki = buildSubjectPublicKeyInfo(publicKeyData)
    let requestInfo = buildCertificationRequestInfo(
        subject: subject,
        subjectPublicKeyInfo: spki
    )

    // 3. Sign with the SE private key — this is the magic moment
    guard let signature = SecKeyCreateSignature(
        privateKey,
        .ecdsaSignatureMessageX962SHA256,
        requestInfo as CFData,
        nil
    ) as Data? else {
        throw CSRError.signingFailed
    }

    // 4. Wrap in CertificationRequest
    let csr = buildCertificationRequest(
        info: requestInfo,
        signature: signature
    )

    return csr
}
```

{{% dialog "🦆 Nestor" %}}
`SecKeyCreateSignature` — that's the call that goes into the Secure Enclave?
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
Precisely. The `requestInfo` bytes are sent to the SE through the hardware mailbox. Inside the enclave, the private key signs the data using ECDSA with SHA-256. The signature comes back out. At no point does the private key — or any representation of it — enter the app's memory.
{{% /dialog %}}

{{% dialog "🐦‍⬛ Autolycus" %}}
*Watching intently.* So even if I hook `SecKeyCreateSignature` with Frida... I'd see the data going in and the signature coming out. But not the key.
{{% /dialog %}}

Exactly. You can *use* the key (by calling the signing function while the app is running), but you can never *take* it.

[PLACEHOLDER: Detailed ASN.1 construction code with explanation — buildSubjectDN, buildSubjectPublicKeyInfo, buildCertificationRequestInfo, buildCertificationRequest]

### Verifying the CSR

Before we send it anywhere, let's make sure our hand-built CSR is valid:

```bash
# PEM-encode it and verify with openssl
openssl req -in se_csr.pem -text -noout -verify
```

```
[PLACEHOLDER: openssl output showing the CSR details — subject, public key, signature verification]
```

## Step 3: Provisioning — getting a signed certificate

Now we need a server endpoint that accepts CSRs and returns signed certificates. Time to add a provisioning port to our Go server.

```go
// --- Port 8445: Provisioning (signs CSRs) ---
provMux.HandleFunc("/sign-csr", func(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)

    block, _ := pem.Decode(body)
    csr, _ := x509.ParseCertificateRequest(block.Bytes)
    csr.CheckSignature()

    template := &x509.Certificate{
        SerialNumber: big.NewInt(rand.Int63()),
        Subject:      csr.Subject,
        NotBefore:    time.Now(),
        NotAfter:     time.Now().Add(365 * 24 * time.Hour),
        KeyUsage:     x509.KeyUsageDigitalSignature,
        ExtKeyUsage:  []x509.ExtKeyUsage{x509.ExtKeyUsageClientAuth},
    }

    certDER, _ := x509.CreateCertificate(nil, template, caCertParsed, csr.PublicKey, caKey)

    w.Header().Set("Content-Type", "application/x-pem-file")
    pem.Encode(w, &pem.Block{Type: "CERTIFICATE", Bytes: certDER})
})
```

{{% dialog "🦉 Menthor" %}}
Notice: the server signs using the CSR's **public key** — `csr.PublicKey`. It never sees the private key. It doesn't need to. The CSR's own signature proves that whoever sent it possesses the corresponding private key. The server trusts that proof and issues a certificate binding the public key to the subject name.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
So the private key never leaves the phone, the server never sees it, and we still get a valid certificate?
{{% /dialog %}}

That's the entire point.

### Submitting the CSR from iOS

```swift
func submitCSR(_ csrData: Data) async throws -> SecCertificate {
    let pemString = "-----BEGIN CERTIFICATE REQUEST-----\n"
        + csrData.base64EncodedString(options: [.lineLength76Characters, .endLineWithLineFeed])
        + "\n-----END CERTIFICATE REQUEST-----\n"

    var request = URLRequest(url: ServerConfig.provisioningURL)
    request.httpMethod = "POST"
    request.httpBody = pemString.data(using: .utf8)

    // Use a session that trusts our CA (same as TLS tab)
    let session = URLSession(configuration: .ephemeral, delegate: self, delegateQueue: nil)
    defer { session.invalidateAndCancel() }

    let (data, response) = try await session.data(for: request)

    guard let http = response as? HTTPURLResponse, http.statusCode == 200 else {
        throw ProvisioningError.serverRejected
    }

    // Parse PEM certificate from response
    let pemResponse = String(data: data, encoding: .utf8)!
    guard let certData = extractDERFromPEM(pemResponse) else {
        throw ProvisioningError.invalidCertificate
    }

    guard let certificate = SecCertificateCreateWithData(nil, certData as CFData) else {
        throw ProvisioningError.invalidCertificate
    }

    return certificate
}
```

## Step 4: Storing the certificate

We have an SE key (permanent, in the Secure Enclave) and a signed certificate (from the server). We need to store the certificate in the Keychain so that iOS can assemble a `SecIdentity` — which links the certificate to its corresponding private key.

```swift
func storeCertificate(_ certificate: SecCertificate, label: String) throws {
    let query: [String: Any] = [
        kSecClass as String: kSecClassCertificate,
        kSecValueRef as String: certificate,
        kSecAttrLabel as String: label,
        kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
    ]

    let status = SecItemAdd(query as CFDictionary, nil)
    guard status == errSecSuccess || status == errSecDuplicateItem else {
        throw KeychainError.storeFailed(status)
    }
}
```

{{% dialog "🦉 Menthor" %}}
The magic here is in how `SecIdentity` works. When you query the Keychain for a `kSecClassIdentity`, iOS automatically matches certificates with private keys by comparing the **public key hash**. The SE key's public key hash matches the certificate's public key hash — because the certificate was issued for exactly that key. iOS connects the dots.
{{% /dialog %}}

```swift
func loadSEIdentity(label: String) -> SecIdentity? {
    let query: [String: Any] = [
        kSecClass as String: kSecClassIdentity,
        kSecAttrLabel as String: label,
        kSecReturnRef as String: true
    ]
    var result: CFTypeRef?
    let status = SecItemCopyMatching(query as CFDictionary, &result)
    guard status == errSecSuccess else { return nil }
    return (result as! SecIdentity)
}
```

## Step 5: Connecting with mTLS

The delegate code is **identical** to Part 2. The `URLSession` doesn't know or care that the identity is backed by the Secure Enclave. It calls `SecKeyCreateSignature` during the TLS handshake, which routes to the SE transparently. From the server's perspective, it's the same mTLS handshake.

```
[PLACEHOLDER: App log output showing the SE-backed mTLS connection succeeding]
```

{{% dialog "🦆 Nestor" %}}
It works! And the key never left the chip?
{{% /dialog %}}

Never.

## The rematch

{{% dialog "🐦‍⬛ Autolycus" %}}
*Hops down from the shelf.* My turn again.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
Oh no.
{{% /dialog %}}

{{% dialog "🐦‍⬛ Autolycus" %}}
Oh *yes*. Same three attacks. Let's see what happens.
{{% /dialog %}}

### Attack 1 (revisited): The P12 heist

```bash
find /var/containers/Bundle/Application/ -name "*.p12" -path "*TLSDemo*"
```

```
[PLACEHOLDER: only client.p12 from Part 2 tabs — no P12 for the SE tab]
```

{{% dialog "🐦‍⬛ Autolycus" %}}
There's no P12 for the Secure Enclave tab. The key was generated on-device. There was never a file to steal.
{{% /dialog %}}

**Result: nothing to extract.**

### Attack 2 (revisited): Hooking the delegate

```bash
frida -U -f com.demo.TLSDemo -l hook_delegate.js
```

```
[PLACEHOLDER: Same hook output — challenges are visible, credential is presented. But the identity references an SE key.]
```

{{% dialog "🐦‍⬛ Autolycus" %}}
I can still see the handshake. The challenges, the credential, the `SecIdentity`. But let me try to extract the key...
{{% /dialog %}}

```javascript
// attempt_extract_se_key.js
var identity = /* ... obtained from hook ... */;
var privateKey = ObjC.classes.SecIdentityCopyPrivateKey(identity);
var error = Memory.alloc(8);
var data = ObjC.classes.SecKeyCopyExternalRepresentation(privateKey, error);
console.log("Key data: " + data);
console.log("Error: " + new ObjC.Object(Memory.readPointer(error)));
```

```
[PLACEHOLDER:
Key data: null
Error: The operation couldn't be completed. (OSStatus error -25260: CSSMERR_CSP_INVALID_KEY_REFERENCE — key is not extractable)
]
```

{{% dialog "🐦‍⬛ Autolycus" %}}
`SecKeyCopyExternalRepresentation` returns an error. The SE refuses to hand over the key bytes. I can see the *reference* to the key, but I can't read what's inside.
{{% /dialog %}}

**Result: key visible but not extractable.**

### Attack 3 (revisited): Keychain dump

```bash
objection -g com.demo.TLSDemo explore
ios keychain dump
```

```
[PLACEHOLDER:
Created                    Accessible                      ACL               Type                  Data
-------------------------  ------------------------------  ----------------  --------------------  ----------------------------
2026-04-29 ...             WhenUnlockedThisDeviceOnly      PrivateKeyUsage   kSecClassKey          <SecureEnclave>
2026-04-29 ...             WhenUnlockedThisDeviceOnly      None              kSecClassCertificate  (certificate data)
]
```

{{% dialog "🐦‍⬛ Autolycus" %}}
The key is there. I can see it in the Keychain. But look at the data column: `<SecureEnclave>`. Not the key bytes. A *reference* to hardware. And the ACL says `PrivateKeyUsage` — I can ask it to sign, but I can't ask it to export.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
So you can *use* it but not *steal* it?
{{% /dialog %}}

{{% dialog "🐦‍⬛ Autolycus" %}}
Only while the app is running, on *this* device, with *this* chip. I can't copy it to my laptop. I can't extract it with `scp`. I can't brute-force a password because there is no password — there are no bytes to decrypt. The key exists only as transistors inside the Secure Enclave.
{{% /dialog %}}

**Result: key reference visible, raw material inaccessible.**

### Scorecard

| Attack | Part 2 (P12) | Part 4 (Secure Enclave) |
|--------|-------------|------------------------|
| **1. Extract from bundle** | `client.p12` copied, identity stolen | No P12 exists |
| **2. Hook delegate** | Credential intercepted, key readable | Credential visible, key not extractable |
| **3. Keychain dump** | Private key extractable (`true`) | `<SecureEnclave>` — hardware reference only |
| **Impersonate from Mac?** | `curl --cert stolen.p12` succeeds | Impossible — key can't leave the device |

{{% dialog "🐦‍⬛ Autolycus" %}}
*Long pause.* Well played. I can see the lock. I can rattle the handle. But I can't pick it — because the lock isn't software. It's silicon.
{{% /dialog %}}

## The full picture

Let's zoom out and look at what we've built across all four parts:

{{< mermaid >}}
sequenceDiagram
    participant SE as 🔐 Secure Enclave
    participant App as 📱 App
    participant Server as 🖥️ Server
    participant CA as 🏛️ CA (on server)

    Note over SE,CA: Provisioning (once)

    App->>SE: Generate key pair
    SE-->>App: Public key (private key stays inside)
    App->>App: Build CSR with public key
    App->>SE: Sign CSR
    SE-->>App: Signature
    App->>Server: POST /sign-csr (PEM-encoded CSR)
    Server->>CA: Verify CSR signature, sign certificate
    CA-->>Server: Signed certificate
    Server-->>App: PEM certificate
    App->>App: Store certificate in Keychain

    Note over SE,CA: mTLS connection (every time)

    App->>Server: TLS ClientHello
    Server->>App: ServerHello + Server Certificate
    App->>App: Verify server cert against bundled CA
    Server->>App: CertificateRequest
    App->>App: Load SecIdentity (cert + SE key ref)
    App->>SE: Sign handshake data
    SE-->>App: Signature
    App->>Server: Client Certificate + Signature
    Server->>CA: Verify client cert
    CA-->>Server: Valid
    Server-->>App: 🤝 Mutual trust established
{{< /mermaid >}}

{{% dialog "🦉 Menthor" %}}
*Closes the book.* Let us review what we have accomplished across this series.

In Part 1, we built the infrastructure — a CA, certificates, a server. In Part 2, we built a client that used those certificates. In Part 3, we demonstrated that software-only key storage is insufficient against a determined attacker on a compromised device. And in Part 4, we moved the private key into hardware that is architecturally isolated from the rest of the system.

The cryptography did not change. TLS is the same. mTLS is the same. The certificates are the same format, the handshake is the same protocol. What changed is *where the key lives* — and that changes everything.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
So the lesson isn't "use better crypto." It's "protect the key."
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
Precisely. The algorithm is only as strong as the key management. A 256-bit EC key is unbreakable in theory — but if it's stored as extractable bytes in software, it's as vulnerable as a house key under a doormat. The Secure Enclave doesn't make the *math* stronger. It makes the *key* unreachable.
{{% /dialog %}}

{{% dialog "🐦‍⬛ Autolycus" %}}
*Ruffles feathers.* I'll be back. There's always another angle.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
But not this angle.
{{% /dialog %}}

{{% dialog "🐦‍⬛ Autolycus" %}}
*Grins.* No. Not this one.
{{% /dialog %}}

*The crow flies off. For now.*
