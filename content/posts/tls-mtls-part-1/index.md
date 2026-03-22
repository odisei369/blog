---
title: "TLS & mTLS — Part 1: Setting Up the Playground"
date: 2026-03-20
draft: true
tags: ["security", "tls", "mtls", "go"]
series: ["TLS & mTLS"]
series_order: 1
summary: "A duck and a developer set up a TLS/mTLS test server to finally understand how certificates work."
---

*A duck enters a dark room.*

{{% dialog "🦆 Nestor" %}}
Hello? Who's there?
{{% /dialog %}}

...Who's asking?

{{% dialog "🦆 Nestor" %}}
It's me, duck Nestor!
{{% /dialog %}}

I don't know that. You *say* you're Nestor, but how can I be sure? Anyone could waddle in here and claim to be a duck. Do you have... proof?

{{% dialog "🦆 Nestor" %}}
Proof? Like... my library card?
{{% /dialog %}}

More like a certificate. Signed by someone we both trust.

And that, right there, is the entire problem that TLS solves.

{{% dialog "🦆 Nestor" %}}
Wait, TLS? I thought they use SSL for this. The library is literally called OpenSSL. I'm confused.
{{% /dialog %}}

*An owl descends from a high shelf, adjusts spectacles, and opens a very thick book.*

{{% dialog "🦉 Menthor" %}}
Ah, a common misconception. Allow me to clarify. *Ahem.* SSL — Secure Sockets Layer — was the original protocol, first published by Netscape in 1995. Versions 2.0 and 3.0 served the early web faithfully. However, in 1999, the IETF standardized a successor under a new name: TLS — Transport Layer Security. TLS 1.0 was, in essence, SSL 3.1 wearing a different hat.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
So... they're the same thing?
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
*Closes book. Opens a thicker book.* They are *not* the same thing, but they are the same *lineage*. SSL is the ancestor. TLS is the living descendant. All versions of SSL have been deprecated — SSL 3.0 was officially retired in 2015. What the world runs today is TLS 1.2 and TLS 1.3. When people say "SSL," they almost always mean TLS. It is, if you will, a ghost name — haunting libraries, documentation, and casual conversation long after the protocol itself has passed.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
And OpenSSL?
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
Named in an era when SSL was the word on everyone's beak. The library supports modern TLS perfectly well. It simply never updated its name. Much like how we still say "dial" a phone number, or "rewind" a video. Language, dear Nestor, has a longer memory than technology.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
Okay but for real, what even is TLS then?
{{% /dialog %}}

The short version: when your app connects to `https://api.example.com`, TLS is the protocol that makes the "S" in HTTPS work. It does two things:

1. **Encryption** — nobody between you and the server can read the traffic
2. **Authentication** — you can verify that you're really talking to `api.example.com` and not some impersonator

The server proves its identity by showing a **certificate** — a document signed by a trusted authority (a Certificate Authority, or CA). Your device checks: "Was this certificate signed by a CA I trust? Is it for the right domain? Is it still valid?" If yes, the connection proceeds.

That's regular TLS. One-way trust: the client verifies the server.

**Mutual TLS (mTLS)** flips the script: the server *also* demands a certificate from the client. Both sides prove who they are. This is used in banking apps (device binding), enterprise MDM, and microservice-to-microservice communication.

{{% dialog "🦆 Nestor" %}}
So how are we going to learn this?
{{% /dialog %}}

We'll build a tiny Go server with two modes: one that does regular TLS, and one that requires mTLS. Then we'll poke at it with `openssl` and `curl` to see exactly what's different. Later, in Part 2, we'll connect from an iOS app.

{{% dialog "🦆 Nestor" %}}
"Poke at it." Finally something I understand. Back on the pond we poke things with sticks all the time. Logs, frogs, suspicious breadcrumbs... You learn a lot about something by poking it.
{{% /dialog %}}

Khm... yeah, kind of. Except our stick is `curl` and the thing we're poking will poke back with a certificate. Anyway — here's the setup:

| Port | Mode | Endpoint | What it proves |
|------|------|----------|----------------|
| 8443 | TLS | `GET /hello` | Server proves its identity |
| 8444 | mTLS | `GET /secure` | Both sides prove identity |
| 8445 | TLS | `POST /sign-csr` | Provisioning — signs certificate requests |

Port 8445 exists for later — when our iOS app generates a key in the Secure Enclave and needs the server to sign a certificate for it. We'll get to that in a future part.

## Generating certificates

{{% dialog "🦆 Nestor" %}}
Wait, so we need certificates to connect, but we need a server to test them, but the server needs certificates to run... Is this a chicken-and-egg thing?
{{% /dialog %}}

More of an egg-and-egg thing. Everything needs a certificate, and every certificate needs a CA. So we start with the CA — the chicken that lays all the eggs. We need three things:

- **A Certificate Authority (CA)** — our pretend trusted authority that signs everything
- **A server certificate** — so the server can prove it's `localhost`
- **A client certificate** — so the client can prove it's `demo-iphone`

All using EC P-256 keys. Why P-256 specifically? Because the iPhone's Secure Enclave only supports P-256, and we want everything to use the same algorithm for consistency when we get to the iOS part.

Here's what the flow looks like:

{{< mermaid >}}
sequenceDiagram
    actor Nestor as 🦆 Nestor
    participant CA as 🏛️ Certificate Authority
    participant Srv as 🖥️ Server
    participant Cli as 📱 iPhone

    Note over Nestor,CA: 1. Create the root of trust
    Nestor->>CA: Give me a key! (ecparam -genkey)
    CA-->>CA: Signs own certificate (req -new -x509)
    Note right of CA: ca.key + ca.pem

    Note over Nestor,Srv: 2. Server certificate
    Nestor->>Srv: You too, here's a key (ecparam -genkey)
    Srv->>CA: Hey CA, I'm localhost. Sign me? (req -new)
    CA->>Srv: Alright, you check out. Here. (x509 -req)
    Note right of Srv: server.key + server.pem

    Note over Nestor,Cli: 3. Client certificate
    Nestor->>Cli: And one for you, little phone (ecparam -genkey)
    Cli->>CA: I'm demo-iphone, can I get a cert? (req -new)
    CA->>Cli: Signed. Don't lose it. (x509 -req)
    Note right of Cli: client.key + client.pem

    Note over Nestor,Cli: 4. Bundle for iOS
    Nestor->>Cli: Let me wrap this up nicely (pkcs12 -export)
    Note right of Cli: client.p12 🎁
{{< /mermaid >}}

And here's the script:

```bash
#!/bin/bash
set -euo pipefail
OUT=./out
mkdir -p "$OUT"

# 1. Certificate Authority — our root of trust
openssl ecparam -genkey -name prime256v1 -noout -out "$OUT/ca.key"
openssl req -new -x509 -key "$OUT/ca.key" -out "$OUT/ca.pem" -days 365 \
    -subj "/CN=TLSDemo CA/O=Demo"

# 2. Server certificate — proves "I am localhost"
openssl ecparam -genkey -name prime256v1 -noout -out "$OUT/server.key"
openssl req -new -key "$OUT/server.key" -out "$OUT/server.csr" \
    -subj "/CN=localhost/O=Demo"
openssl x509 -req -in "$OUT/server.csr" \
    -CA "$OUT/ca.pem" -CAkey "$OUT/ca.key" -CAcreateserial \
    -out "$OUT/server.pem" -days 365 \
    -extfile <(printf "subjectAltName=DNS:localhost,IP:127.0.0.1")

# 3. Client certificate — proves "I am demo-iphone"
openssl ecparam -genkey -name prime256v1 -noout -out "$OUT/client.key"
openssl req -new -key "$OUT/client.key" -out "$OUT/client.csr" \
    -subj "/CN=demo-iphone/O=Demo"
openssl x509 -req -in "$OUT/client.csr" \
    -CA "$OUT/ca.pem" -CAkey "$OUT/ca.key" -CAcreateserial \
    -out "$OUT/client.pem" -days 365

# 4. Bundle client cert + key into P12 for iOS
openssl pkcs12 -export -out "$OUT/client.p12" \
    -inkey "$OUT/client.key" -in "$OUT/client.pem" -certfile "$OUT/ca.pem" \
    -passout pass:demo
```

{{% dialog "🦆 Nestor" %}}
That's a lot of flags. I don't like flags. Last time I saw that many flags was at a golf course, and I didn't like that place either.
{{% /dialog %}}

Each certificate follows the same pattern: **generate a key → create a CSR (Certificate Signing Request) → have the CA sign it**.

Here's the same pattern shown for each certificate — generate a key, create a CSR, have the CA sign it:

{{< mermaid >}}
sequenceDiagram
    actor Nestor as 🦆 Nestor
    participant Key as 🔑 Private Key
    participant CSR as 📋 CSR
    participant CA as 🏛️ CA
    participant Cert as 📜 Certificate

    Note over Nestor,Cert: Nestor repeats this dance 3 times

    Nestor->>Key: Make me a key!
    Key->>CSR: Here's my public key + who I am
    CSR->>CA: Please sign this, I'm legit I swear
    CA->>CA: Hmm let me check...
    CA->>Cert: Fine. You're legit for 365 days.
    Cert-->>Nestor: 🎉
{{< /mermaid >}}

The server certificate has a Subject Alternative Name (SAN) — `DNS:localhost,IP:127.0.0.1`. This is important: modern TLS stacks (including iOS) ignore the old Common Name field for hostname matching and require SAN instead.

After running the script, we get:

```
out/
├── ca.key          # CA private key (guards the kingdom)
├── ca.pem          # CA certificate (everyone trusts this)
├── server.key      # Server private key
├── server.pem      # Server certificate (signed by CA)
├── client.key      # Client private key
├── client.pem      # Client certificate (signed by CA)
└── client.p12      # Client cert + key bundled for iOS (password: "demo")
```

## The Go server

{{% dialog "🦆 Nestor" %}}
That's a lot of crypto setup for a "simple" server.
{{% /dialog %}}

Fair. But the server itself is actually short. Go's standard library has everything we need — no external dependencies. Here's the full thing:

```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"encoding/json"
	"encoding/pem"
	"fmt"
	"io"
	"log"
	"math/big"
	"math/rand"
	"net/http"
	"os"
	"time"
)

func main() {
	certsDir := "../certs/out"

	// Load CA — we need it to verify client certs and sign CSRs
	caCert, _ := os.ReadFile(certsDir + "/ca.pem")
	caPool := x509.NewCertPool()
	caPool.AppendCertsFromPEM(caCert)

	caKeyPEM, _ := os.ReadFile(certsDir + "/ca.key")
	caKeyBlock, _ := pem.Decode(caKeyPEM)
	caKey, _ := x509.ParseECPrivateKey(caKeyBlock.Bytes)

	caCertBlock, _ := pem.Decode(caCert)
	caCertParsed, _ := x509.ParseCertificate(caCertBlock.Bytes)

	serverCert, _ := tls.LoadX509KeyPair(
		certsDir+"/server.pem", certsDir+"/server.key",
	)

	// --- Port 8443: TLS only ---
	tlsMux := http.NewServeMux()
	tlsMux.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
		json.NewEncoder(w).Encode(map[string]string{
			"message": "Hello from TLS server",
			"auth":    "server-only (one-way TLS)",
		})
	})

	tlsServer := &http.Server{
		Addr:    ":8443",
		Handler: tlsMux,
		TLSConfig: &tls.Config{
			Certificates: []tls.Certificate{serverCert},
			ClientAuth:   tls.NoClientCert, // <-- the server doesn't ask "who are you?"
		},
	}

	// --- Port 8444: mTLS ---
	mtlsMux := http.NewServeMux()
	mtlsMux.HandleFunc("/secure", func(w http.ResponseWriter, r *http.Request) {
		cn := "unknown"
		if r.TLS != nil && len(r.TLS.PeerCertificates) > 0 {
			cn = r.TLS.PeerCertificates[0].Subject.CommonName
		}
		json.NewEncoder(w).Encode(map[string]string{
			"message": fmt.Sprintf("Hello %s, mutual trust established", cn),
			"auth":    "mutual TLS (both sides verified)",
			"client":  cn,
		})
	})

	mtlsServer := &http.Server{
		Addr:    ":8444",
		Handler: mtlsMux,
		TLSConfig: &tls.Config{
			Certificates: []tls.Certificate{serverCert},
			ClientAuth:   tls.RequireAndVerifyClientCert, // <-- "show me YOUR certificate"
			ClientCAs:    caPool,
		},
	}

	// --- Port 8445: Provisioning (signs CSRs) ---
	provMux := http.NewServeMux()
	provMux.HandleFunc("/sign-csr", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			http.Error(w, "POST only", http.StatusMethodNotAllowed)
			return
		}
		body, _ := io.ReadAll(r.Body)

		block, _ := pem.Decode(body)
		if block == nil || block.Type != "CERTIFICATE REQUEST" {
			http.Error(w, "Invalid PEM", http.StatusBadRequest)
			return
		}

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
		log.Printf("[provisioning] Signed certificate for CN=%s", csr.Subject.CommonName)
	})

	provServer := &http.Server{
		Addr:    ":8445",
		Handler: provMux,
		TLSConfig: &tls.Config{
			Certificates: []tls.Certificate{serverCert},
			ClientAuth:   tls.NoClientCert,
		},
	}

	log.Println("TLS  server on :8443 (server auth only)")
	log.Println("mTLS server on :8444 (mutual auth required)")
	log.Println("Prov server on :8445 (CSR signing)")

	go tlsServer.ListenAndServeTLS("", "")
	go mtlsServer.ListenAndServeTLS("", "")
	provServer.ListenAndServeTLS("", "")
}
```

The entire difference between TLS and mTLS in this server is **one field**:

```go
// TLS — server doesn't care who the client is
ClientAuth: tls.NoClientCert

// mTLS — server demands the client prove its identity too
ClientAuth: tls.RequireAndVerifyClientCert
```

That's it. One line. We're not Go developers — this server is just our test harness. The real action will be on the iOS side in Part 2.

Run it with `go run main.go` from the `backend/` directory.

## Testing with curl

{{% dialog "🦆 Nestor" %}}
Okay it's running. Now what?
{{% /dialog %}}

Now we poke at it. Let's start with `curl`.

### Test 1: TLS — happy path

```bash
$ curl --cacert out/ca.pem https://localhost:8443/hello
```
```json
{"auth":"server-only (one-way TLS)","message":"Hello from TLS server"}
```

We tell curl to trust our CA (`--cacert out/ca.pem`). curl verifies the server certificate was signed by this CA, the handshake succeeds, and we get our response.

### Test 2: TLS — without trusting the CA

```bash
$ curl https://localhost:8443/hello
```
```
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

Without `--cacert`, curl doesn't trust our self-signed CA. This is exactly what happens when an iOS app connects to a server with an unknown certificate — the system rejects it because it can't verify the chain of trust.

{{% dialog "🦆 Nestor" %}}
So `--cacert` is like telling curl "I personally vouch for this CA"?
{{% /dialog %}}

Exactly. In a real app, the system trust store already contains the well-known CAs (Let's Encrypt, DigiCert, etc.). But for our self-signed setup, we need to explicitly say "trust this CA."

### Test 3: mTLS — without a client certificate

```bash
$ curl --cacert out/ca.pem https://localhost:8444/secure
```
```
curl: (56) LibreSSL/3.3.6: error:1404076D:SSL routines:ST_OK:
  certificate required
```

The server requires a client certificate (`RequireAndVerifyClientCert`) and we didn't provide one. Handshake fails.

### Test 4: mTLS — with a client certificate

```bash
$ curl --cacert out/ca.pem \
       --cert out/client.pem \
       --key out/client.key \
       https://localhost:8444/secure
```
```json
{"auth":"mutual TLS (both sides verified)","client":"demo-iphone","message":"Hello demo-iphone, mutual trust established"}
```

Now *both* sides authenticated. The server verified our client certificate was signed by the trusted CA, extracted the Common Name (`demo-iphone`), and included it in the response.

{{% dialog "🦆 Nestor" %}}
So the server knows exactly which client is connecting, not just that "someone with a valid token" is connecting?
{{% /dialog %}}

Right. And the identity is proven at the transport level — before any HTTP request is even sent. No tokens to steal, no passwords to phish. The private key never leaves the client.

## Inspecting the handshake with openssl

curl is great for testing, but `openssl s_client` lets us see the actual TLS handshake.

### TLS handshake

```bash
$ openssl s_client -connect localhost:8443 -CAfile out/ca.pem
```

```
CONNECTED(00000005)
---
Certificate chain
 0 s:O = Demo, CN = localhost
   i:O = Demo, CN = TLSDemo CA
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIBkTCB+wIUH...
-----END CERTIFICATE-----
subject=O = Demo, CN = localhost
issuer=O = Demo, CN = TLSDemo CA
---
...
Verify return code: 0 (ok)
```

Key things to notice:
- **Certificate chain** — our server cert (`CN=localhost`), issued by our CA (`CN=TLSDemo CA`)
- **Verify return code: 0 (ok)** — the chain is valid because we passed `-CAfile out/ca.pem`
- No client certificate was requested or sent

### mTLS handshake

```bash
$ openssl s_client -connect localhost:8444 \
    -CAfile out/ca.pem \
    -cert out/client.pem \
    -key out/client.key
```

```
CONNECTED(00000005)
---
Acceptable client certificate CA names
O = Demo, CN = TLSDemo CA
---
Server certificate
...
subject=O = Demo, CN = localhost
issuer=O = Demo, CN = TLSDemo CA
---
...
Verify return code: 0 (ok)
```

Notice the extra section: **Acceptable client certificate CA names**. The server is telling the client: "I'll accept certificates signed by these CAs." This is the mTLS handshake in action — the server explicitly requests a client certificate.

If you try this without `-cert` and `-key`:

```bash
$ openssl s_client -connect localhost:8444 -CAfile out/ca.pem
```

The connection will fail because the server requires a client certificate and we didn't present one.

## What's next

{{% dialog "🦆 Nestor" %}}
So this is what the server side looks like. What about the phone?
{{% /dialog %}}

Next time — we connect from an iOS app. That's where it gets interesting:

- **`URLSessionDelegate`** and how it handles TLS challenges
- **`SecTrust`** evaluation — verifying our custom CA programmatically
- **`SecPKCS12Import`** — loading the client identity from a `.p12` bundle
- And eventually, the **Secure Enclave** — generating a private key that can never be extracted, even on a jailbroken device

The Go server stays running. The certificates stay the same. We just need to teach the iPhone to speak the same language.

See you in Part 2.
