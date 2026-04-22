---
title: "TLS & mTLS — Part 2: The iOS Client"
date: 2026-03-21
draft: false
tags: ["security", "tls", "mtls", "ios", "swift"]
series: ["TLS & mTLS"]
series_order: 2
summary: "Nestor and the developer teach an iPhone to speak TLS. A crow shows up uninvited."
---

{{% dialog "🦆 Nestor" %}}
Last time we built a server and poked it with sticks. When do we build the phone thing?
{{% /dialog %}}

Right now. We have a Go server running three ports — TLS on 8443, mTLS on 8444, provisioning on 8445. Time to write an iOS app that connects to it.

But first, a small detour.

## Why iOS 15?

We're targeting iOS 15. Not iOS 18. Not even iOS 17 with all its shiny new `@Observable` macros and —

*A loud flutter. Something dark lands on the monitor.*

{{% dialog "🐦‍⬛ Autolycus" %}}
Shiny? Someone said shiny? Where?
{{% /dialog %}}

...Who are you?

{{% dialog "🐦‍⬛ Autolycus" %}}
Autolycus. Crow. Connoisseur of interesting things. Especially things that people try to keep locked away. *Tilts head.* Did you say shiny new APIs?
{{% /dialog %}}

We said we're *not* using them. We're targeting iOS 15 because we have a jailbroken iPhone 6S, and we want to test how secure our app actually is on a real compromised device.

{{% dialog "🐦‍⬛ Autolycus" %}}
Jailbroken? *Eyes widen.* Oh, I like where this is going.
{{% /dialog %}}

You'll get your turn later. For now, sit on the shelf and watch.

{{% dialog "🐦‍⬛ Autolycus" %}}
*Hops to a higher shelf. Watches intently.*
{{% /dialog %}}

So — iOS 15 means `ObservableObject` instead of `@Observable`, `NavigationView` instead of `NavigationStack`, and no `#Preview` macro. Old school. Everything still works, the code is just a bit more verbose. Worth the tradeoff for what comes in Part 3.

{{% dialog "🦆 Nestor" %}}
What comes in Part 3?
{{% /dialog %}}

We'll get there. Let's build first.

## The app structure

Two tabs for now:

| Tab | What it does | Server port |
|-----|-------------|-------------|
| **TLS** | Connects to the server, verifies its certificate | `:8443/hello` |
| **mTLS** | Same as above, plus presents a client certificate | `:8444/secure` |

Both tabs share the same Go server from Part 1, the same certificates, the same CA. The only difference is what happens during the handshake.

## The gatekeeper: URLSessionDelegate

Here's the thing about iOS networking that most tutorials gloss over: when you call `URLSession.data(from: url)`, the system doesn't just fetch data. Before any HTTP request is sent, TLS kicks in. The server presents its certificate. And iOS asks your app: *"Do you trust this server?"*

That question arrives as an **authentication challenge**, delivered to your `URLSessionDelegate`.

{{% dialog "🦆 Nestor" %}}
So the delegate is like... a bouncer?
{{% /dialog %}}

More like a customs officer. The server shows its passport (certificate), and the delegate decides whether to let the connection through.

For regular HTTPS to `api.google.com`, you don't need a delegate — iOS checks the certificate against the system trust store automatically. But we're using a self-signed CA, so we need to handle the challenge ourselves.

Here's what the flow looks like:

{{< mermaid >}}
sequenceDiagram
    participant App as 📱 App
    participant Session as URLSession
    participant Delegate as URLSessionDelegate
    participant Server as 🖥️ Server

    App->>Session: data(from: url)
    Session->>Server: TLS ClientHello
    Server->>Session: ServerHello + Certificate
    Session->>Delegate: didReceive challenge:<br/>serverTrust
    Note over Delegate: Is this cert signed<br/>by our CA?
    Delegate->>Session: useCredential ✓
    Session->>Server: Finished
    Server->>Session: HTTP 200 + JSON
    Session->>App: (data, response)
{{< /mermaid >}}

For mTLS, there's a second challenge:

{{< mermaid >}}
sequenceDiagram
    participant App as 📱 App
    participant Session as URLSession
    participant Delegate as URLSessionDelegate
    participant Server as 🖥️ Server

    App->>Session: data(from: url)
    Session->>Server: TLS ClientHello
    Server->>Session: ServerHello + Certificate<br/>+ CertificateRequest
    Session->>Delegate: didReceive challenge:<br/>serverTrust
    Delegate->>Session: useCredential ✓
    Session->>Delegate: didReceive challenge:<br/>clientCertificate
    Note over Delegate: Load P12, extract<br/>SecIdentity
    Delegate->>Session: useCredential + identity ✓
    Session->>Server: Client Certificate + Finished
    Server->>Session: HTTP 200 + JSON
    Session->>App: (data, response)
{{< /mermaid >}}

Two challenges instead of one. That's the entire difference on the client side.

## Trusting our CA

Before we can handle challenges, we need a way to verify the server certificate against our custom CA. Remember from Part 1 — we generated `ca.der` for exactly this purpose.

```swift
enum CertificateHelper {

    static func loadCACertificate() -> SecCertificate? {
        guard let url = Bundle.main.url(forResource: "ca", withExtension: "der"),
              let data = try? Data(contentsOf: url) else {
            return nil
        }
        return SecCertificateCreateWithData(nil, data as CFData)
    }

    static func evaluateServerTrust(
        _ serverTrust: SecTrust,
        against caCert: SecCertificate
    ) -> (success: Bool, error: String?) {
        // "Trust ONLY this CA, nothing else"
        SecTrustSetAnchorCertificates(serverTrust, [caCert] as CFArray)
        SecTrustSetAnchorCertificatesOnly(serverTrust, true)

        var error: CFError?
        let result = SecTrustEvaluateWithError(serverTrust, &error)
        if let error = error {
            let desc = CFErrorCopyDescription(error) as String? ?? "unknown"
            return (false, desc)
        }
        return (result, nil)
    }
}
```

`SecTrustSetAnchorCertificates` tells iOS: "Here's my list of trusted CAs." And `SecTrustSetAnchorCertificatesOnly(serverTrust, true)` adds: "And *only* these. Don't fall back to the system trust store." Without that second call, iOS would also accept certificates signed by any public CA — not what we want.

{{% dialog "🦉 Menthor" %}}
*Opens a book.* A note on `SecTrustEvaluateWithError`. It performs the full chain validation: signature verification, expiry check, hostname matching against the Subject Alternative Name, Key Usage constraints, and Extended Key Usage. If any of these fail, it returns `false` with an error describing which check failed. It is, shall we say, thorough.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
Learned that the hard way, didn't we?
{{% /dialog %}}

We did. If your CA certificate doesn't have `keyUsage=keyCertSign`, iOS will reject it with "certificate is not trusted" — even though the signature is mathematically valid. And if your server certificate doesn't have `extendedKeyUsage=serverAuth`, you get "certificate is not permitted for this usage." Both pass just fine with `curl`. iOS is stricter.

## Tab 1: TLS — verifying the server

The TLS service only handles one challenge type: `NSURLAuthenticationMethodServerTrust`.

```swift
final class TLSService: NSObject, URLSessionDelegate {

    private let caCert: SecCertificate
    private let log: (LogEntry) -> Void

    init(caCert: SecCertificate, log: @escaping (LogEntry) -> Void) {
        self.caCert = caCert
        self.log = log
    }

    func connect(to url: URL) async throws -> ServerResponse {
        log(.info("Starting TLS connection..."))

        let session = URLSession(
            configuration: .ephemeral,
            delegate: self,
            delegateQueue: nil
        )
        defer { session.invalidateAndCancel() }

        let (data, response) = try await session.data(from: url)
        guard let http = response as? HTTPURLResponse,
              http.statusCode == 200 else {
            throw TLSError.badResponse
        }
        return try JSONDecoder().decode(ServerResponse.self, from: data)
    }

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard challenge.protectionSpace.authenticationMethod
                == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        log(.info("Received server trust challenge"))

        let evaluation = CertificateHelper.evaluateServerTrust(serverTrust, against: caCert)
        if evaluation.success {
            log(.success("Server identity verified"))
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            log(.error("Verification failed: \(evaluation.error ?? "unknown")"))
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

That's it. The delegate receives the challenge, evaluates the server certificate against our CA, and either accepts or rejects. Tap "Connect" in the app and the log shows:

```
→ Starting TLS connection...
→ Received server trust challenge
→ Evaluating server certificate against custom CA...
✓ Server identity verified
✓ Response: Hello from TLS server
```

{{% dialog "🦆 Nestor" %}}
That's... not that much code.
{{% /dialog %}}

Most of the work is in understanding *what* each piece does. The code itself is short.

## Tab 2: mTLS — proving who we are

Now the interesting part. The mTLS service handles **two** challenge types.

But first, we need to load the client identity. Remember the `.p12` file from Part 1? It bundles the client's private key and certificate into one password-protected container.

```swift
static func loadIdentity(from p12Name: String, password: String) -> SecIdentity? {
    guard let url = Bundle.main.url(forResource: p12Name, withExtension: "p12"),
          let data = try? Data(contentsOf: url) else {
        return nil
    }

    let options: [String: Any] = [
        kSecImportExportPassphrase as String: password
    ]
    var items: CFArray?
    let status = SecPKCS12Import(data as CFData, options as CFDictionary, &items)

    guard status == errSecSuccess,
          let itemArray = items as? [[String: Any]],
          let firstItem = itemArray.first,
          let identity = firstItem[kSecImportItemIdentity as String] else {
        return nil
    }
    return (identity as! SecIdentity)
}
```

`SecPKCS12Import` takes the raw `.p12` data and the password, and gives back a `SecIdentity` — an opaque object that holds both the private key and the certificate. We'll hand this to the URL session when the server asks "who are you?"

{{% dialog "🐦‍⬛ Autolycus" %}}
*From the shelf.* Did he just hardcode the password as `"demo"`?
{{% /dialog %}}

It's a demo. Hence the name.

{{% dialog "🐦‍⬛ Autolycus" %}}
*Takes notes.*
{{% /dialog %}}

Now the delegate. It's the same as the TLS version, but with one extra case:

```swift
func urlSession(
    _ session: URLSession,
    didReceive challenge: URLAuthenticationChallenge,
    completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
) {
    let method = challenge.protectionSpace.authenticationMethod

    switch method {
    case NSURLAuthenticationMethodServerTrust:
        // Same as TLS — verify the server
        guard let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        let evaluation = CertificateHelper.evaluateServerTrust(serverTrust, against: caCert)
        if evaluation.success {
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }

    case NSURLAuthenticationMethodClientCertificate:
        // NEW — the server is asking us to prove our identity
        guard let identity = clientIdentity else {
            log(.error("No client identity — server will reject us"))
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        log(.success("Presenting client identity"))
        let credential = URLCredential(
            identity: identity,
            certificates: nil,
            persistence: .forSession
        )
        completionHandler(.useCredential, credential)

    default:
        completionHandler(.cancelAuthenticationChallenge, nil)
    }
}
```

The `serverTrust` case is identical. The `clientCertificate` case is the new one — it wraps our `SecIdentity` in a `URLCredential` and hands it to the session.

That's the entire difference between a TLS client and an mTLS client: **one extra case in a switch statement**.

Tap "Connect" on the mTLS tab:

```
→ Loaded client identity from bundled P12
→ Starting mTLS connection...
→ Received server trust challenge
✓ Server identity verified
→ Received client certificate challenge
✓ Presenting client identity
✓ Response: Hello demo-iphone, mutual trust established
```

The server recognized us as `demo-iphone` — the Common Name from our client certificate. Mutual trust.

## What happens without a client cert?

The app has a "Without Cert" button that attempts the same connection but skips loading the `.p12`. What does the server do?

```
→ Skipping client certificate (will fail)
→ Starting TLS-only (no client cert) connection...
→ Received server trust challenge
✓ Server identity verified
✗ Connection failed: ...certificate required
```

The server trust check passes — we still trust the server. But then the server asks for *our* certificate, we have nothing to show, and the handshake dies. The server log shows `TLS handshake error` — it never even gets to the HTTP layer.

{{% dialog "🦆 Nestor" %}}
So mTLS is all-or-nothing? Either both sides authenticate, or nobody talks?
{{% /dialog %}}

Exactly. There's no "let me in without a badge" option. That's the point.

## Side by side

Let's look at the two delegates next to each other. Everything is the same except what's highlighted:

{{< mermaid >}}
graph TD
    subgraph TLS["TLS Delegate"]
        A1[Challenge arrives] --> B1{authenticationMethod?}
        B1 -->|serverTrust| C1[Evaluate against CA]
        C1 -->|pass| D1[useCredential ✓]
        C1 -->|fail| E1[cancel ✗]
    end

    subgraph mTLS["mTLS Delegate"]
        A2[Challenge arrives] --> B2{authenticationMethod?}
        B2 -->|serverTrust| C2[Evaluate against CA]
        C2 -->|pass| D2[useCredential ✓]
        C2 -->|fail| E2[cancel ✗]
        B2 -->|clientCertificate| F2[Load P12 identity]
        F2 -->|found| G2[useCredential + identity ✓]
        F2 -->|missing| H2[cancel ✗]
    end

    style F2 fill:#2d5a27,stroke:#4a8,color:#fff
    style G2 fill:#2d5a27,stroke:#4a8,color:#fff
    style H2 fill:#5a2727,stroke:#a44,color:#fff
{{< /mermaid >}}

One extra branch. That's mTLS on iOS.

## It works!

We have an iOS app that does both TLS and mTLS. The server verifies the client. The client verifies the server. Mutual trust, transport-level authentication, no tokens or passwords involved.

{{% dialog "🦆 Nestor" %}}
So... we're done? Ship it?
{{% /dialog %}}

{{% dialog "🐦‍⬛ Autolycus" %}}
*Perks up from the shelf.* Yes. Definitely ship it. Looks very secure. Nothing to worry about. *Preens feathers innocently.*
{{% /dialog %}}

...

{{% dialog "🦆 Nestor" %}}
Why is the crow smiling?
{{% /dialog %}}

Good question. See you in Part 3.
