---
title: "TLS & mTLS — Part 4: The Secure Enclave"
date: 2026-04-28
draft: true
tags: ["security", "tls", "mtls", "ios", "secure-enclave", "frida"]
series: ["TLS & mTLS"]
series_order: 4
summary: "We fix what the crow broke. The Secure Enclave generates a key that never leaves the chip — and Autolycus finally meets a lock he can't pick."
---

*Code for this part: [github.com/odisei369/tls-mtls-demo @ `part-4`](https://github.com/odisei369/tls-mtls-demo/tree/part-4)*

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

Every ASN.1 value, in DER, is a **tag-length-value** record: one byte for the tag, one-or-more bytes for the length, then the value. Constructed types like `SEQUENCE` are just TLV records whose value is the concatenation of more TLV records. So the whole encoder collapses into a tiny set of primitives that compose:

```swift
enum ASN1 {

    enum Tag {
        static let integer: UInt8    = 0x02
        static let bitString: UInt8  = 0x03
        static let oid: UInt8        = 0x06
        static let utf8String: UInt8 = 0x0C
        static let sequence: UInt8   = 0x30  // constructed
        static let set: UInt8        = 0x31  // constructed
    }

    /// tag || length || value
    static func tlv(tag: UInt8, value: Data) -> Data {
        var out = Data([tag])
        out.append(encodeLength(value.count))
        out.append(value)
        return out
    }

    /// DER length: short form for <128, long form otherwise.
    static func encodeLength(_ length: Int) -> Data {
        if length < 0x80 { return Data([UInt8(length)]) }
        var bytes: [UInt8] = []
        var n = length
        while n > 0 {
            bytes.insert(UInt8(n & 0xFF), at: 0)
            n >>= 8
        }
        return Data([0x80 | UInt8(bytes.count)] + bytes)
    }

    static func sequence(_ children: [Data]) -> Data {
        tlv(tag: Tag.sequence, value: children.reduce(Data(), +))
    }

    static func set(_ children: [Data]) -> Data {
        tlv(tag: Tag.set, value: children.reduce(Data(), +))
    }

    /// Context-specific tag [n], constructed — used for the CSR's `attributes [0]`.
    static func contextSpecificConstructed(_ tagNumber: UInt8, _ value: Data) -> Data {
        tlv(tag: 0xA0 | (tagNumber & 0x1F), value: value)
    }

    /// Non-negative INTEGER. DER requires a leading 0x00 if the top bit is set
    /// (otherwise the value would be interpreted as negative).
    static func integer(_ value: Int) -> Data {
        var bytes: [UInt8] = []
        var n = value
        if n == 0 { bytes = [0x00] } else {
            while n > 0 { bytes.insert(UInt8(n & 0xFF), at: 0); n >>= 8 }
            if bytes[0] & 0x80 != 0 { bytes.insert(0x00, at: 0) }
        }
        return tlv(tag: Tag.integer, value: Data(bytes))
    }

    /// BIT STRING with the leading 0x00 "unused bits" byte
    /// (we never pad partial bytes in a CSR).
    static func bitString(_ data: Data) -> Data {
        var value = Data([0x00])
        value.append(data)
        return tlv(tag: Tag.bitString, value: value)
    }

    static func utf8String(_ s: String) -> Data {
        tlv(tag: Tag.utf8String, value: Data(s.utf8))
    }

    /// OID from a dotted string, e.g. "1.2.840.10045.2.1".
    /// Rules: first two arcs combine as `40 * a + b`; later arcs are base-128
    /// with the high bit set on every byte except the last.
    static func oid(_ dotted: String) -> Data {
        let arcs = dotted.split(separator: ".").compactMap { UInt64($0) }
        var bytes: [UInt8] = [UInt8(arcs[0] * 40 + arcs[1])]
        for arc in arcs.dropFirst(2) { bytes.append(contentsOf: base128(arc)) }
        return tlv(tag: Tag.oid, value: Data(bytes))
    }

    private static func base128(_ value: UInt64) -> [UInt8] {
        if value == 0 { return [0x00] }
        var out: [UInt8] = []
        var n = value
        while n > 0 { out.insert(UInt8(n & 0x7F), at: 0); n >>= 7 }
        for i in 0..<(out.count - 1) { out[i] |= 0x80 }
        return out
    }
}
```

{{% dialog "🦆 Nestor" %}}
That's the whole encoder?
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
That is the whole encoder. ASN.1 has a fearsome reputation, but DER is — at its core — recursion plus a couple of edge cases. The cases worth remembering are the length encoding (short vs. long form), the leading-zero byte on positive INTEGERs, and the "unused bits" prefix on BIT STRINGs. Everything else is bookkeeping.
{{% /dialog %}}

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

Each helper assembles one fragment of the PKCS#10 structure. They all return raw `Data` so the outer functions can compose them like Lego bricks.

```swift
private enum OID {
    static let commonName  = "2.5.4.3"
    static let ecPublicKey = "1.2.840.10045.2.1"
    static let prime256v1  = "1.2.840.10045.3.1.7"
    static let ecdsaSHA256 = "1.2.840.10045.4.3.2"
}

// Name ::= SEQUENCE OF RelativeDistinguishedName
// RDN ::= SET OF AttributeTypeAndValue { type OID, value ANY }
// We use a single RDN containing only a CommonName.
private static func buildSubjectDN(commonName: String) -> Data {
    let attribute = ASN1.sequence([
        ASN1.oid(OID.commonName),
        ASN1.utf8String(commonName)
    ])
    return ASN1.sequence([ ASN1.set([attribute]) ])
}

// SubjectPublicKeyInfo ::= SEQUENCE {
//     algorithm        AlgorithmIdentifier { ecPublicKey, prime256v1 },
//     subjectPublicKey BIT STRING  -- the 65-byte uncompressed EC point
// }
private static func buildSubjectPublicKeyInfo(publicKeyPoint: Data) -> Data {
    let algorithm = ASN1.sequence([
        ASN1.oid(OID.ecPublicKey),
        ASN1.oid(OID.prime256v1)
    ])
    return ASN1.sequence([
        algorithm,
        ASN1.bitString(publicKeyPoint)
    ])
}

// CertificationRequestInfo — these are the bytes that get signed.
private static func buildCertificationRequestInfo(commonName: String, publicKeyPoint: Data) -> Data {
    ASN1.sequence([
        ASN1.integer(0),                                           // version v1
        buildSubjectDN(commonName: commonName),                     // subject
        buildSubjectPublicKeyInfo(publicKeyPoint: publicKeyPoint),  // subjectPKInfo
        ASN1.contextSpecificConstructed(0, Data())                  // attributes [0] — empty
    ])
}

// CertificationRequest ::= SEQUENCE {
//     certificationRequestInfo CertificationRequestInfo,
//     signatureAlgorithm       AlgorithmIdentifier,
//     signature                BIT STRING
// }
// ecdsa-with-SHA256 has ABSENT parameters per RFC 5758 — no NULL after the OID.
// SecKeyCreateSignature already returns a DER-encoded ECDSA-Sig-Value
// (SEQUENCE { r INTEGER, s INTEGER }), so we drop it straight into a BIT STRING.
private static func buildCertificationRequest(info: Data, signature: Data) -> Data {
    let sigAlgorithm = ASN1.sequence([ ASN1.oid(OID.ecdsaSHA256) ])
    return ASN1.sequence([
        info,
        sigAlgorithm,
        ASN1.bitString(signature)
    ])
}
```

{{% dialog "🦆 Nestor" %}}
Two OIDs in the public key algorithm, but only one in the signature algorithm?
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
Different specs, different conventions. For `id-ecPublicKey`, the curve is encoded as a parameter — it tells you *which* EC group the key lives in. For `ecdsa-with-SHA256`, the curve is already implied by the public key, and the spec explicitly forbids parameters (not even an explicit `NULL`). It is the kind of detail you only learn by reading the RFC, building the CSR, watching `openssl` reject it, and reading the RFC again.
{{% /dialog %}}

### Verifying the CSR

Before we send it anywhere, let's make sure our hand-built CSR is valid:

```bash
# PEM-encode it and verify with openssl
openssl req -in se_csr.pem -text -noout -verify
```

```
Certificate request self-signature verify OK
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: CN=TLSDemo-SE-Client
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:f2:b8:fb:55:08:da:19:65:61:d1:94:d5:cc:86:
                    50:62:1b:40:a4:68:99:36:8d:c0:63:60:cb:64:5a:
                    7d:cb:c3:21:b0:7c:0a:41:8b:65:88:2d:de:4b:09:
                    2e:19:d2:4d:11:d1:d8:b3:b2:28:be:76:ea:80:e7:
                    b1:b2:d8:a6:24
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        Attributes:
            (none)
            Requested Extensions:
    Signature Algorithm: ecdsa-with-SHA256
    Signature Value:
        30:45:02:21:00:b3:a8:13:3e:6e:6d:b8:53:08:73:dc:9b:8c:
        03:58:b8:bb:6c:32:3f:b7:9f:34:44:37:d5:2f:f6:9e:55:00:
        98:02:20:6b:c6:76:a8:09:98:b8:43:c0:49:d1:2d:8a:78:bb:
        d5:0a:76:98:a2:3b:7e:21:71:d5:bb:7b:9e:ad:85:db:02
```

That first line — `Certificate request self-signature verify OK` — is what we care about. The CSR was signed by the Secure Enclave, and the signature checks out against the public key embedded in the same CSR. Subject `CN=TLSDemo-SE-Client`, `id-ecPublicKey` on `prime256v1` (P-256), `ecdsa-with-SHA256` — exactly the structure we built by hand.

## Step 3: Provisioning — getting a signed certificate

Now we need a server endpoint that accepts CSRs and returns signed certificates. Time to add a provisioning port to our Go server.

> **Reminder:** this is the first step in Part 4 that talks to the server. If your Mac's LAN IP has changed since Part 2 — or if you're jumping straight into Part 4 — regenerate the server certificate with your current IP using Part 1's `--extra-ip` flag (`./generate_certs.sh --extra-ip <your-mac-ip>`), copy the fresh `ca.der` into the app's bundle, restart the Go server, and clean-build the app so the new CA is picked up. Otherwise the TLS handshake to `:8445` will fail with a trust error before the CSR ever reaches the endpoint.

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
Loaded SE-backed SecIdentity from Keychain
Starting SE-backed mTLS connection to <IP>:8444
Received server trust challenge
Server identity verified
Received client certificate challenge
Presenting SE-backed client identity (key stays in chip)
Response: Hello TLSDemo-SE-Client, mutual trust established
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
Ilyas-iPhone:~ mobile% find /var/containers/Bundle/Application/ -name "*.p12" -path "*TLSDemo*"
/var/containers/Bundle/Application/E7D3000A-DDC2-4865-9B72-CFF10225B1AD/TLSDemo.app/client.p12
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
