---
title: "Rust on the Other Side of BLE — Part 2: The Reset Loop That Lied"
date: 2026-06-15
draft: false
tags: ["rust", "embedded", "nrf52840", "ble", "embassy", "debugging"]
series: ["Rust on the Other Side of BLE"]
series_order: 2
summary: "A $3 debug probe, three bugs stacked on top of each other, and the BLE stack that finally agreed to advertise. Each fix only revealed the next failure underneath."
---

In [Part 1](/posts/rust-other-side-of-ble-part-1/) we discovered that the nRF52840 SuperMini ships with 200 KB of code we didn't write, that the UF2 bootloader silently drops writes to protected regions, and that what looked like a happy blink pattern was in fact a crash loop dressed up to look like a heartbeat.

When we left off, the board was doing this:

```
flash, flash, [long pause], flash, flash, [long pause], flash, flash, [long pause]...
```

Two short flickers, a five-second silence, repeat. Forever. Penelope had already named it: a *Penelope-loop* — the bootloader's double-tap detector misreading a fast crash-and-reset as a deliberate double-tap into DFU mode. The pattern was a symptom; the cause was that our firmware was panicking somewhere in the first few milliseconds of boot, and we had no way to see where.

This part is about how we got to *see*. And then about the three bugs we found, one at a time, each one only visible after the previous one was fixed.

## Why the LED can't help you

I tried, of course. Everybody tries. The LED is right there on the board, you've already proved you can drive it, surely you can use it to narrate execution.

Here is what that looks like in practice.

| Attempt | What we tried | What we learned |
|---|---|---|
| 1 | Blink on / off | "The LED blinks." But is it our pattern or a reset loop? |
| 2 | Staged blink patterns (1 blink = stage 1, 2 blinks = stage 2…) | "Fast pulsing." Can't tell which stage. |
| 3 | 3 sec ON / 1 sec OFF | Same short flashes. The timer never completes. |
| 4 | Custom panic handler (LED off on crash) | "Two blinks then off." OK, it panics. But where? |
| 5 | Watchdog feeding in `pre_init` | Same pattern. Not the watchdog. Or is it? |

Five iterations. Hours each. We still don't know what line crashes.

{{% dialog "🦆 Nestor" %}}
I have learned, scientifically, that something is wrong. Somewhere. Possibly in software.
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
Right?! You have *one bit* of bandwidth! One single bit, on or off, and you're trying to encode an entire stack trace through it. It's like trying to describe a film by waving a flashlight at the wall. You can do it! But the films are *short* and *vague* and there are *no credits*.
{{% /dialog %}}

The deeper problem isn't even bandwidth. It's that the LED only lights up if your code ran far enough to set the GPIO. If the panic happens before `embassy_nrf::init()` returns — which, spoiler, ours did — then "the LED stayed off" tells you the same thing as "the panic message contained the word *banana*": nothing.

You need a side channel that the chip can use *while it's panicking*. Something that doesn't depend on your code being healthy enough to drive a pin. You need a debug probe.

## $3 of debug probe

*An owl lands on a stack of receipts.*

{{% dialog "🦉 Menthor" %}}
A debug probe is a small piece of hardware that speaks the chip's native debug protocol — on Cortex-M parts, that protocol is SWD: Serial Wire Debug. Two wires, plus ground and a voltage reference. The probe can halt the CPU, single-step instructions, read and write memory and registers, and — most usefully for our case — pipe a side-channel logging stream out of the chip in real time. That stream is called RTT, *Real-Time Transfer*, and it works even when the application is on fire.
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
The ecosystem standard on the Rust side is `probe-rs`, which replaces older tooling like OpenOCD. Pair it with `defmt` (a format-string logging framework whose payload is so small it fits comfortably over RTT) and `defmt-rtt` (the transport glue), and `cargo run` becomes: build, flash via the probe, attach to RTT, stream formatted logs to your terminal, in one command.
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
And the probe is *three dollars!* An ST-Link V2 clone off any import site — the little purple USB stick, you've seen it. Plus four wires. Plus four pads on the bottom of the board the size of dust mites. That's the whole tooling story. *Three dollars.*
{{% /dialog %}}

The pads are real, and they are real small. On the SuperMini they sit on the underside of the PCB in a block of four labeled `VDD / DIO / CLK / GND`. Pitch is **1.5 mm** — which sounds close to the 1.27 mm of a standard ARM Cortex-Debug header but is just different enough that a 1.27 mm test clip will not seat. The drift compounds across four pads to about 0.7 mm, and the far-end pogo misses entirely.

**A gotcha if you're buying tooling:** the standard SWD test clips you'll find on every electronics distributor are 1.27 mm pitch. The SuperMini's pads are 1.5 mm pitch. Either solder wires directly to the pads (we did this; we destroyed a `VDD` pad in the process and had to switch boards), or hunt down a 1.5 mm pogo clip — they exist, they're cheap, but the listings often advertise the wrong pitch in the title.

For our wiring we ran four thin wires from the ST-Link clone:

```
ST-Link V2 clone      SuperMini pad
────────────────      ─────────────
SWDIO            →    DIO
SWCLK            →    CLK
GND              →    GND
3.3V             →    VDD
```

The `3.3V → VDD` line is the probe's voltage reference, and on the USB-stick clone it also sources power for the target. The rule is: **never have both the SuperMini's USB-C cable and the ST-Link `3.3V` wire connected at the same time.** Pick one power source. We picked the probe; the board draws under 50 mA even during BLE TX, well within what the ST-Link can supply.

A quick sanity check before flashing anything:

```bash
$ probe-rs list
The following debug probes were found:
[0]: STLink V2 -- VID:PID 0483:3748 -- Serial: 0671FF...

$ probe-rs info --chip nRF52840_xxAA
Probing target via JTAG
ARM Chip with debug port Default:
Debug Port: DPv1, DP Designer: ARM Ltd
└── 0 MemoryAP
    ├── ROM Table (Class 1), Designer: Nordic VLSI ASA
    ...
```

Probe sees the chip. We can now flash without going through the UF2 bootloader at all, which means we can also reclaim the full 1 MB of flash if we feel like it — the SoftDevice from Part 1 is no longer a barrier. For this debugging session we erased everything and switched `memory.x` back to the canonical `FLASH ORIGIN = 0x00000000` with the full megabyte:

```c
MEMORY
{
  FLASH : ORIGIN = 0x00000000, LENGTH = 1024K
  RAM   : ORIGIN = 0x20000000, LENGTH = 256K
}
```

And the build runner in `.cargo/config.toml`:

```toml
[target.thumbv7em-none-eabihf]
runner = "probe-rs run --chip nRF52840_xxAA"
```

Now the moment of truth:

```bash
$ cargo run --release
   Compiling ble-quiz-display v0.1.0
    Finished release [optimized + debuginfo] target(s) in 14.2s
     Erasing ✔ [00:00:00] [###########] 32.00 KiB/32.00 KiB @ 38.20 KiB/s
 Programming ✔ [00:00:01] [###########] 32.00 KiB/32.00 KiB @ 21.04 KiB/s
      Finished in 1.71s
```

The chip resets. RTT attaches. Logs start streaming.

And we discover, immediately, our first real bug.

## Bug #1 — the task arena is too small

```
INFO  ble: main entered
ERROR panicked at 'embassy-executor: task arena is full.
       You must increase the arena size, see the documentation for details:
       https://docs.embassy.dev/embassy-executor/'
```

There it is. Line one of `main`, then a panic. Not in our code — inside Embassy's executor. The task arena is full.

{{% dialog "🦆 Nestor" %}}
Wait. We have one task. The `mpsl_task`. *One.* And we've got 256 KB of RAM. What is full?
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
A common misconception. `embassy-executor` does not allocate task futures from the general heap — which on a `no_std` target you typically don't have anyway. It allocates them from a single statically-sized arena chosen at *compile time*, via a Cargo feature flag. The default is four kilobytes. The futures returned by `nrf-sdc` plus `trouble-host` are large — async state machines in embedded carry every local variable that survives across an `.await` point, and BLE has a great many of those.
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
There is no dynamic resize. The arena is what you told it to be at build time. If your futures don't fit, the executor panics on the first `spawn`. *Ahem.*
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
And this is *exactly* what we couldn't see from the LED! "Panicked at: arena full" is right there in the message, with a URL! The fix is in the `Cargo.toml`! With an *LED* we'd still be soldering watchdog circuits.
{{% /dialog %}}

The fix is a feature flag on `embassy-executor`:

```toml
embassy-executor = { version = "0.7", features = [
    "arch-cortex-m",
    "executor-thread",
    "defmt",
    "task-arena-size-32768",
] }
```

Available sizes are powers of two from 64 up to 262144. 32 KB is comfortable for a peripheral with a handful of tasks; bump higher if you spawn more. The arena lives in `.bss`, so it costs you RAM whether or not you fill it.

Reflash. Reset. RTT reattaches.

```
INFO  ble: main entered
INFO  ble: embassy_nrf::init returned
INFO  ble: SPI configured
INFO  ble: CS pin configured
INFO  ble: display created, calling init()
INFO  ble: display.init() done, calling startup_animation()
INFO  Display ready
```

The display init runs. The startup sweep animation runs. We've moved the program counter forward by a few hundred lines. The LED matrix lights up.

And then we hit bug #2.

## Bug #2 — the warning that lies

```
WARN  Memory buffer too big. 1128 bytes needed
INFO  BLE Quiz Display started — advertising as "RustQuiz"
WARN  Advertise error: BleHost(Hci(Memory Capacity Exceeded))
ERROR panicked at 'SoftdeviceController: 57:845'
```

The Nordic SoftDevice Controller — which, recall, we're linking in as a Rust library, not using the pre-flashed S140 — needs a fixed memory pool at startup, allocated by the application:

```rust
let mut sdc_mem = nrf_sdc::Mem::<4720>::new();
```

On boot, it prints `Memory buffer too big. 1128 bytes needed`. Read in English, that sentence has one obvious interpretation: *the buffer you gave me is bigger than necessary; please shrink it.* So I shrank it. The warning came back with the same `1128`. I grew it. The warning came back with the same `1128`. Then the first `advertise()` call returned `Memory Capacity Exceeded` and the controller panicked, just as before.

{{% dialog "🦆 Nestor" %}}
The warning says the buffer is too big. So I made it smaller. And the bug got worse. So I made it bigger. And the bug stayed the same.
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
The warning is *advisory*. The warning is *gossip*. The warning is the SDC telling you, very politely, "FYI you allocated more than the current configuration needs — your buffer is 4720 bytes, but only 1128 of them are in use." It is not telling you anything about a crash. It is *narrating* and we are *over-reading*. The crash is the truth. **Trust the crash.**
{{% /dialog %}}

The `Memory Capacity Exceeded` is HCI status `0x07`. It means: the controller does not have the buffers it needs to fulfill the request you just made. The request was `advertise()`. The reason the controller didn't have the buffers is buried in the SDC's builder, which is intentionally minimal — every feature must be opted in.

The original builder was:

```rust
let sdc = nrf_sdc::Builder::new()?
    .support_peripheral()?
    .build(sdc_p, &mut rng, mpsl, &mut sdc_mem)?;
```

Reads sensibly. We want the peripheral role; we called `.support_peripheral()`. Done?

Not done. The full list of things `.support_peripheral()` does *not* do, by default:

- It doesn't enable **advertising**. That is `.support_adv()`, separately. The peripheral role and the advertising role are tracked as independent capabilities by the SDC; opting into one does not opt you into the other.
- It doesn't allocate any **HCI ACL buffers**. The ACL buffer pool — the queues that carry data between the host and the controller — defaults to zero entries, in both directions. Zero. The host can technically construct a packet but the controller has nowhere to receive it.
- It doesn't allocate a **peripheral connection slot**. `peripheral_count` defaults to zero. There is no room in the controller's connection table for the connection that the central is about to make.

What the *advisory* warning was secretly trying to tell us, in retrospect: the SDC was using 1128 bytes of the 4720 we'd given it because the SDC was effectively unconfigured. We were paying for the buffer pool and using almost none of it.

The fixed builder:

```rust
let sdc = nrf_sdc::Builder::new()?
    .support_adv()?
    .support_peripheral()?
    .peripheral_count(1)?
    .buffer_cfg(27, 27, 3, 3)?   // tx_size, rx_size, tx_count, rx_count
    .build(sdc_p, &mut rng, mpsl, &mut sdc_mem)?;
```

The `27` is the default LE ACL payload size in BLE 4.x; the `3, 3` is three packets in each direction. The host stack actually told us this in an earlier log line that we'd been ignoring — `trouble-host` had said `setting txq to 3, fragmenting at 27` and we'd missed the implication entirely. The host had reported what it wanted the controller to be configured for. We hadn't wired it through.

**A gotcha for the SDC builder in general:** every defaultable value is zero, not "sensible." The compiler will not catch a missing `.support_adv()`, because both versions type-check fine. The only diagnostic is a runtime HCI status code at the moment of first use, which arrives many initialization steps after the bug.

Reflash. Reset. RTT reattaches.

```
INFO  BLE Quiz Display started — advertising as "RustQuiz"
WARN  Advertise error: BleHost(Hci(Invalid HCI Command Parameters))
WARN  Advertise error: BleHost(Hci(Invalid HCI Command Parameters))
WARN  Advertise error: BleHost(Hci(Invalid HCI Command Parameters))
```

A different error. Forever. Every call to `advertise()`, rejected. No panic this time — `trouble-host` catches the HCI error, logs it, and loops back to try again. We're stable and we are wrong.

## Bug #3 — the address that doesn't exist

`Invalid HCI Command Parameters` is HCI status `0x12`. It is the BLE equivalent of "400 Bad Request" — the controller didn't like something about the parameters you sent, and the spec gives it broad latitude to say so without elaborating. Roughly a dozen distinct problems can produce this status. We had to figure out which one.

The eventual answer required reading `trouble-host`'s own source. In `peripheral.rs`, when the host calls `LeSetAdvParams` (the HCI command that sets up advertising), it has to specify `own_addr_kind` — what kind of Bluetooth address the advertisement should claim to come from. If the host's `address` field has not been set by the user, `trouble-host` defaults `own_addr_kind` to `AddrKind::PUBLIC`.

A *public* BLE address is a globally-unique 48-bit identifier, IEEE-allocated, the same kind classic Bluetooth chips have. You get one of these by paying for an OUI block from the IEEE Registration Authority and burning a unique address into each chip at manufacture time.

**Nordic does not do this for the nRF52840.** There is no factory-assigned public BLE address on this chip. There is, instead, a random-static address derivable from the FICR (Factory Information Configuration Registers), which the controller is happy to use *if you ask*. But if you don't ask, the host defaults to public, the controller looks for a public address, finds none, and rejects every `LeSetAdvParams` forever.

{{% dialog "🦆 Nestor" %}}
The radio works. The stack is configured. The advertising is enabled. The only thing missing is the *name* the chip is supposed to use on the air.
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
This is a BLE-spec compliance issue, not a Nordic eccentricity. The Bluetooth specification defines four address types: public, random-static, resolvable private, and non-resolvable private. Each has a distinct encoding in the most-significant two bits of the most-significant byte. Random-static is `0b11`; resolvable private is `0b01`; non-resolvable private is `0b00`. A public address is, by convention, not signaled in those bits — it's distinguished by the `own_addr_kind` field of `LeSetAdvParams`.
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
The result is that to advertise from this chip, you must generate a 48-bit random value, set the top two bits of byte 5 to `0b11` to mark it static-random, and pass it to the host stack *before* the host issues any advertising command. *Ahem.*
{{% /dialog %}}

In code:

```rust
let mut bd_addr = [0u8; 6];
rng.blocking_fill_bytes(&mut bd_addr);
bd_addr[5] |= 0xC0;   // top 2 bits = 0b11 → static random per BT spec

let stack = trouble_host::new(sdc, resources)
    .set_random_address(Address::random(bd_addr));
```

The `0xC0` mask is the BT-spec encoding for *Static Random Address*. The other top-bit patterns encode the other address subtypes — and `LeSetAdvParams` would also reject those, because the host explicitly asked for `random-static`, not `resolvable-private`.

**A subtle gotcha along the way:** the SDC builder's `.build()` method takes `&mut rng` and holds the borrow for the controller's entire lifetime. The controller needs RNG continuously — key generation, address resolution, advertising interval jitter, all of it. So `rng.blocking_fill_bytes(&mut bd_addr)` must happen *before* `.build()`, because after the build the borrow checker won't let you touch `rng` again. Putting the address generation in the natural-feeling place — right next to the `set_random_address` call, after the controller is built — produces a borrow-check error that is easy to misread as a generic Rust lifetime puzzle rather than a hint about who owns the RNG.

Reflash. Reset. RTT reattaches.

```
INFO  ble: random static address C7:84:3A:91:F2:0B
INFO  BLE Quiz Display started — advertising as "RustQuiz"
```

Nothing else. No error. Silence.

I picked up the phone, opened nRF Connect, and tapped *Scan*.

```
RustQuiz                          -47 dBm
C7:84:3A:91:F2:0B
```

There it was.

{{% dialog "🦆 Nestor" %}}
…that's it? That's the whole BLE peripheral?
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
*Eight months* of debugging compressed into "yes, that's it." Tap it! Tap *Connect!*
{{% /dialog %}}

I tapped Connect. The phone showed the GATT service from the firmware. Two characteristics: `Votes`, `Control`. Both writable.

The board was on the air.

## The bonus lesson: bugs come in stacks

Three bugs, three rounds. Each one only became visible after the previous one was fixed. The task arena was full *before* the SDC was ever initialised, so we couldn't see any SDC misconfiguration until we fixed the arena. The SDC misconfiguration crashed on the first `advertise()` call, so we couldn't see the address-type bug until we fixed the configuration. The address-type bug then failed silently in a loop, which is its own kind of cruelty.

{{< mermaid >}}
graph TD
    A[Crash loop] --> B[Get SWD probe]
    B --> C[See panic: task arena full]
    C --> D[Fix: task-arena-size-32768]
    D --> E[See: Memory Capacity Exceeded]
    E --> F[Fix: support_adv + peripheral_count + buffer_cfg]
    F --> G[See: Invalid HCI Command Parameters x∞]
    G --> H[Fix: random-static address before build]
    H --> I[Advertising as RustQuiz]
    style A fill:#3a1a1a,stroke:#f77f00,color:#fff
    style I fill:#16213e,stroke:#4ecdc4,color:#fff
{{< /mermaid >}}

I want to dwell on this for a moment because it changed how I think about embedded debugging.

When you sit down to debug bug #1, it is tempting — overwhelmingly tempting — to try to reason about what the *real* root cause is. To form a hypothesis about the eventual end state, the one that explains everything, and then to optimize your fixes toward that. The first symptom is the crash loop. You see the crash loop, you don't have probe logs yet, and your brain immediately constructs a story: "it's the memory layout, it's the SoftDevice colliding, it's a bad ABI somewhere, the SDC is fighting the MBR for interrupt vectors." You write that story down. You start fixing the wrong thing.

In our case the corrupted backtrace from bug #2 — when we eventually saw it — produced program counters of `0x100` and `0x0`. Both nonsense. Both *look* exactly like the kind of corruption you'd expect from a memory-layout problem, which was the story we'd been telling. They weren't. They were a stack overflow inside the panic handler, because the panic handler was reached so early in boot that the stack hadn't been fully initialised yet. The "evidence for the memory-layout theory" was actually evidence for an entirely different bug whose symptom happened to rhyme with what we were already looking for.

{{% dialog "🦆 Nestor" %}}
So… don't predict?
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
Don't *predict the root cause*. Fix what's *provably broken right now*. Advance the program counter by one bug. Look at what fails next. The next failure is the real next bug — not the one you imagined when you only had the first symptom. The first bug always hides the second bug. *Always.* It's almost a law.
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
The corollary is that the speed of your debug iteration is more important than the depth of your reasoning per iteration. With a probe and RTT, each round takes 30 seconds: reflash, reset, read the panic, edit, rebuild. Without a probe, each round takes an hour. A 120× speedup on iteration time matters more than any individual insight, because the bugs are layered and you cannot skip ahead to the bottom of the stack. *Ahem.*
{{% /dialog %}}

There is, by the way, a fourth fix that I haven't told you about yet — but it is not a bug, it is a build-pipeline subtlety, and I'm saving it for Part 3 because it ties directly into the demo. Quick teaser: the first UF2 we produced was 6.2 MB for a 132 KB firmware. The reason is delightful and has to do with what an ELF file actually contains. More on that next time.

## What we did and didn't see

| Symptom | Without a probe | With a probe |
|---|---|---|
| Two-flash-then-pause LED | "It's blinking? Is it working? Is it crashing?" | `panicked at 'task arena is full'` |
| Bootloader reset loop | Maybe? Hard to tell. | Visible from the very first reset. |
| SDC misconfiguration | Silent crash after a long warning log. | `Memory Capacity Exceeded` on the line that called `advertise()`. |
| Wrong BLE address type | Silent infinite loop. Board appears dead. | `Invalid HCI Command Parameters`, with a search target. |

I am not going to tell you the LED-debugging hours were wasted. They taught me the shape of the failure (two flashes, long pause). That shape was the clue that led me to suspect a reset loop rather than a happy blink, and it's why I bothered to read `INFO_UF2.TXT` in the first place. But everything I learned from the LED, I learned in the first 30 minutes. The remaining hours produced no new information. The probe produced more information in its first three seconds of RTT output than the LED had produced in two evenings.

Get the probe. Get it before you order the board.

## Cliffhanger

The board advertises as `RustQuiz`. The phone connects. The GATT service is visible. Two characteristics, `Votes` and `Control`, both writable. The LED matrix is sitting on the desk waiting for instructions.

What goes on the wire?

Part 3 covers what's actually in the firmware now that it's running: the eight-line GATT service definition, the single-task event loop, the MAX7219 SPI driver with its XOR letter-cutout trick, and the small army of glue around the firmware — a Mac BLE bridge in Rust, a Bun + TypeScript quiz server — that turns *audience taps on phones* into *bars rising across the room*.

We'll also resolve the 6.2 MB UF2 teaser. It's a one-flag fix and a four-line explanation, but it's the kind of thing that, on a deadline, will eat your afternoon.

See you in Part 3.

---

*The full source is at [github.com/odisei369/ble-quiz-display](https://github.com/odisei369/ble-quiz-display).*
