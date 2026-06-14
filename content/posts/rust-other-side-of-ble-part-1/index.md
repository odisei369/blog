---
title: "Rust on the Other Side of BLE — Part 1: The Board Is Not a Blank Slate"
date: 2026-06-14
draft: false
tags: ["rust", "embedded", "nrf52840", "ble", "embassy"]
series: ["Rust on the Other Side of BLE"]
series_order: 1
aliases:
  - /rust-ble
summary: "A duck buys a $6 microcontroller. The cuckoo inside it points out that 200 KB of the flash is already occupied by code the duck never wrote."
---

*A small cardboard envelope lands on the table. Nestor pokes at it with a webbed foot.*

{{% dialog "🦆 Nestor" %}}
Six bucks. *Six dollars.* For a whole BLE microcontroller! With an antenna and a USB port and everything!
{{% /dialog %}}

It is a small black PCB, the size of a thumb, labelled **nRF52840 SuperMini**. The same form factor as the Nice!Nano, which is what hand-wired keyboard people use to make their wireless keyboards. ARM Cortex-M4F at 64 MHz, 1 MB of flash, 256 KB of RAM, BLE 5.0, about six dollars from any of the usual import sites. By any reasonable standard it is an absurd amount of computer for the price.

{{% dialog "🦆 Nestor" %}}
I'm going to write Rust on it. Async Rust. On bare metal. No operating system. Just me, the silicon, and the future.
{{% /dialog %}}

*A small grey bird pops up from behind the USB connector, hops onto a decoupling capacitor, then onto the SWD pads, then back. She does not appear capable of standing still.*

{{% dialog "🐦 Penelope" %}}
Oh, hello! Finally! You have *no idea* how long I've been waiting for someone to actually pay attention to this thing.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
...who are you?
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
Penelope. Cuckoo. I live in the clock — the high-frequency one mostly, the 64 MHz one, it just *hums* — and I hop around when there's nothing happening, which up until you plugged this in was basically constantly. I've been watching this board boot for twenty-six thousand cycles. Give or take.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
Twenty-six thousand cycles? You've been here that long?
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
Hex! Hexadecimal. Don't worry about it. Listen — much more importantly — half your flash is already full. *Half!* By stuff you didn't write. It's wild, you have to see it.
{{% /dialog %}}

This is, it turns out, the first thing nobody tells you about this board.

## What "blank" actually means

When you buy a microcontroller, you imagine a blank slate. Empty flash. A pristine canvas for your code. You picture pressing the reset button and a void looking back, asking *"what should I do?"*

The reality on the nRF52840 SuperMini is closer to moving into a furnished apartment. The previous tenant left:

1. A **Master Boot Record** at `0x00000000` — 4 KB of code that decides what to run when the chip powers up.
2. A **SoftDevice S140** at `0x00001000` — 152 KB of Nordic's proprietary, closed-source BLE stack. You did not ask for this. It is here anyway.
3. An **Adafruit UF2 Bootloader** at `0x000F4000` — 48 KB at the top of flash, the thing that turns the board into a USB drive when you double-tap reset.

The space in between — from `0x00026000` to `0x000F4000`, about 824 KB — is yours. *That's* the canvas. The rest is furniture you can't easily move.

```text
0x00000000  ┌──────────────────────────┐
            │  MBR                4 KB │  ← protected
0x00001000  ├──────────────────────────┤
            │                          │
            │  SoftDevice S140  152 KB │  ← protected, unused by us
            │                          │
0x00026000  ├──────────────────────────┤
            │                          │
            │                          │
            │  Application      824 KB │  ← your code lives here
            │                          │
            │                          │
0x000F4000  ├──────────────────────────┤
            │  UF2 Bootloader    48 KB │  ← protected, self-preservation
0x00100000  └──────────────────────────┘
                  1 MB total flash
```

{{% dialog "🦆 Nestor" %}}
Wait. Why is there a BLE stack already on it? I'm writing my own BLE code.
{{% /dialog %}}

*An owl descends, opens a thick book labelled "Nordic Semiconductor Architecture Decisions 2012–2024", and clears her throat.*

{{% dialog "🦉 Menthor" %}}
Ah. The SoftDevice. *Ahem.* For more than a decade, Nordic's prescribed model for Bluetooth on their chips was the following: Nordic compiles the BLE radio, the link layer, the host stack, GATT, GAP, encryption, and the scheduler into a single closed-source binary called a "SoftDevice." You flash it to the lower portion of your chip. Your application sits above it. When your application wants to do anything BLE-related, it does not call functions — it issues a `SVC` interrupt, which traps into the SoftDevice. The SoftDevice owns the radio, owns certain timers, and owns the scheduler. You do not.
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
The S140 is the BLE 5.0 variant of this design for the nRF52840. Adafruit's bootloader, designed originally for boards using Zephyr and ZMK keyboard firmware, ships with the S140 pre-flashed. It is, in essence, infrastructure for a BLE architecture you may or may not opt into.
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
Oh, we're *not* opting in. No SVC traps into a blob, no calling into Nordic's C code — we're doing all of this in Rust. Controller, host, GATT, everything compiled straight into our binary. The S140 can sit there and watch. It's going to be *gorgeous*.
{{% /dialog %}}

We're going to use a pure-Rust BLE stack — `trouble-host` for GATT, `nrf-sdc` for the controller — both compiled directly into our firmware binary. The S140 will just sit there, 152 KB of dead weight. We will not call it. We will not delete it. We will walk around it.

Why not just delete it? That's its own rabbit hole, and we'll get to it. First we have to talk about what happens when you don't.

## The silent-write trap

Every embedded Rust tutorial I'd ever read starts the same way. Add an `embassy-nrf` dependency. Set up your `memory.x`:

```c
MEMORY
{
  FLASH : ORIGIN = 0x00000000, LENGTH = 1024K
  RAM   : ORIGIN = 0x20000000, LENGTH = 256K
}
```

A nice round zero. The beginning of flash. The first byte the CPU reads after reset. This is what every guide tells you to write. So I wrote it.

```bash
$ cargo build --release
   Compiling ble-quiz-display v0.1.0
    Finished release [optimized + debuginfo] target(s) in 12.4s

$ cargo objcopy --release -- -O binary firmware.bin
$ uf2conv firmware.bin --base 0x00000000 --family 0xADA52840 -o firmware.uf2
$ cp firmware.uf2 /Volumes/NICENANO/
```

The board accepted the file. The USB drive disappeared, the way it does after a successful flash. The board rebooted. The LED came on briefly.

And then it crashed.

No error. No warning. No diagnostic. Just a board that rebooted, blinked once, and rebooted again.

{{% dialog "🦆 Nestor" %}}
Did it not write? It said it wrote. The drive ejected itself. That's the "I'm done" signal, right?
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
Oh no, it wrote *part* of it! That's the trap, that's the *whole* trap. Your UF2 had blocks for every address from zero on up, but everything between `0x00000` and `0x26000` hit a wall — the bootloader has protected regions and it just drops those blocks on the floor. No message, no log, nothing. The "flash complete" you got was the USB transfer finishing — the bootloader never actually promised to write the bytes where you asked. It is *so polite* about lying to you.
{{% /dialog %}}

This is the trap.

The Adafruit UF2 bootloader **protects** three regions of flash: the MBR, the SoftDevice area, and itself. Any UF2 block targeting an address inside those regions is silently ignored. No error message, no warning, no log. The "Write complete" feedback you get from the USB transfer is purely about the transfer — the bootloader did not promise to write anywhere you told it to.

So a firmware built with `FLASH ORIGIN = 0x00000000` lands like this:

- Bytes `0x00000` – `0x26000` (MBR + S140 region): silently discarded.
- Bytes `0x26000` – wherever your firmware ends: written.
- Result: the first ~150 KB of your firmware — including, you know, the *reset vector* — is gone.

The CPU boots. The MBR jumps to the S140 (because that's what the MBR does). The S140 expects to find a calling application above it but finds your `0x26000`-and-up scraps, which were compiled assuming the rest of them were at `0x00000`. Nothing matches. Stack pointers are wrong. Branch targets are wrong. Reset vector is wrong.

Crash. Reset. Crash. Reset.

**A gotcha if your board ships with a SoftDevice:** every "Hello World" embedded Rust example you'll find online assumes a fresh chip with `FLASH ORIGIN = 0x00000000`. On any Nice!Nano-compatible board, that origin is *wrong by 152 KB*, and the UF2 bootloader will give you zero diagnostics about it. You will spend hours debugging code that was never actually flashed to the address it thinks it lives at.

## How to find out what's actually on your board

There is, miraculously, a way to discover this in 30 seconds. The UF2 bootloader doubles as a USB mass storage device. When you double-tap the reset button, the board mounts as a drive called `NICENANO` (or `NRF52BOOT`, depending on the variant). On that drive there is a file:

```bash
$ cat /Volumes/NICENANO/INFO_UF2.TXT
UF2 Bootloader 0.6.0 lib/nrfx (v2.0.0) lib/tinyusb (0.10.1-41-gdf0cda2d)
Model: nice!nano
Board-ID: nRF52840-nicenano
SoftDevice: S140 version 6.1.1
Date: Jun 19 2021
```

That single line — `SoftDevice: S140 version 6.1.1` — tells you everything. The board ships with the S140. You cannot flash `0x00000000` through UF2. You must either set `FLASH ORIGIN = 0x00026000` and live above the SoftDevice, or remove the SoftDevice entirely (which, spoiler, requires a debug probe and burns the bootloader along with it).

{{% dialog "🦆 Nestor" %}}
So before you flash anything... you read a text file. On a USB drive. That the chip pretends to be.
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
Right?! It's the best part! The chip *pretends to be a USB drive* so it can hand you a single text file that tells you the truth about being a chip. There's no filesystem on flash — the bootloader makes one up on the fly just for that one file! And nobody tells you to read it, you just have to know. Secret handshake. I love secret handshakes.
{{% /dialog %}}

I had not read this file before flashing. I had not read it because the tutorial I followed did not say *"before you do anything, read `INFO_UF2.TXT`."* Every tutorial assumes you have a development board you bought from a reputable distributor and that the manufacturer's datasheet describes the silicon, not the previous tenant's apartment.

This was lesson zero. There would be more.

## Two ways out

There are two viable approaches to the SoftDevice problem, and the choice has consequences that ripple through the rest of the firmware design.

### Option A — Keep the S140 and live above it

This is the path of least resistance. Set the linker to skip the S140 region:

```c
/* memory.x — adjusted for Nice!Nano with UF2 bootloader + S140 */
MEMORY
{
  FLASH : ORIGIN = 0x00026000, LENGTH = 824K   /* after the S140 */
  RAM   : ORIGIN = 0x20000000, LENGTH = 256K   /* full RAM available */
}
```

You also have to tell `uf2conv` about the offset, otherwise the UF2 file will encode the wrong target address:

```bash
$ cargo objcopy --release -- -O binary firmware.bin
$ uf2conv firmware.bin --base 0x26000 --family 0xADA52840 -o firmware.uf2
$ cp firmware.uf2 /Volumes/NICENANO/
```

**Pros:**
- Keeps the UF2 bootloader, so you can keep flashing by drag-and-drop. Very convenient for demos: no probe, no wires, no host-side tooling, just plug the board in and copy a file.
- Forgiving — if your firmware ever bricks, you can recover by double-tapping reset and flashing a known-good UF2.

**Cons:**
- You lose 152 KB of flash to a stack you'll never call.
- The S140 owns certain interrupt vectors via the MBR. If your pure-Rust BLE stack also wants those vectors (and `nrf-sdc` *does*), there's potential for conflict. In our case, this turned out to be OK — the controller library binds the radio handlers directly and the MBR's forwarding logic stays out of the way — but it's a thing to be aware of.

### Option B — Wipe everything, reclaim the chip

If you have an SWD debug probe, you can erase the entire flash. Not just your application area: the MBR, the S140, the bootloader, all of it. `probe-rs` will happily oblige:

```bash
$ probe-rs erase --chip nRF52840_xxAA
   Erased 1024 KiB in 4.21s
```

Now flash is genuinely empty. Set `FLASH ORIGIN = 0x00000000` like every tutorial says, and the firmware lands exactly where it expects to.

**Pros:**
- Full 1 MB of flash for your application.
- No dead 152 KB of unused C code in flash.
- Clean architecture: the MBR doesn't forward interrupts to anyone but you, because there is nobody else.

**Cons:**
- The UF2 bootloader is gone forever (well, until you flash it back from Adafruit's GitHub). All future updates must go through the SWD probe.
- The board is not recoverable without the probe. If you brick it from across the room, you cannot fix it from across the room.

### What we picked

For this project we chose **Option A**. Our firmware is 132 KB. We have 824 KB of usable flash. We can afford the dead weight, and keeping the UF2 bootloader means a brick is a recoverable mistake — double-tap reset, drag-and-drop a known-good UF2, and you're back. That recoverability is worth more than 152 KB to me.

**A gotcha for option A in particular:** the binary you flash via UF2 will be larger than you expect. We'll see in Part 3 that converting an ELF directly to UF2 produces a 6.2 MB file for a 132 KB firmware (because of address-space gaps between flash and RAM sections), and you have to go through a raw binary as an intermediate step. We're not done with the build pipeline yet.

## What happens when you finally flash it correctly

So I fixed `memory.x`, set `--base 0x26000`, rebuilt, reflashed. The board accepted the file. It rebooted. The on-board LED started doing this:

```
flash, flash, [long pause], flash, flash, [long pause], flash, flash, [long pause]...
```

Two short flickers. A pause of about five seconds. Two more flickers. Five more seconds. On and on.

{{% dialog "🦆 Nestor" %}}
It's blinking! It's *working!* That's... wait, that's not the pattern I programmed. I didn't program any pattern yet. I haven't even initialised the LED.
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
Oh — oh no. Oh wait. That's not a heartbeat.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
What is it?
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
It's a *Penelope-loop*. Look at the rhythm — two flashes, five-second pause, two flashes, five-second pause. That's the Adafruit bootloader's *double-tap detector* firing! It's there for humans: press reset twice fast, drop into DFU mode for five seconds, drag-and-drop a fresh UF2. But your firmware is crashing so fast on boot that two reset cycles happen back-to-back, and the bootloader can't tell the difference between a quick double-tap and a quick double-crash. So it opens the DFU window, waits politely for an upload you're not sending, gives up, runs your firmware, which crashes again — and we go around. Weave, unweave, weave, unweave. Honestly kind of beautiful! But we have to break it.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
...how am I supposed to debug something I can't see?
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
You're not. Not with an LED — you've got one bit of bandwidth and like five things you're trying to say with it. You need a *probe*. Tiny SWD probe, three dollars, an ST-Link clone off any import site. Solder four wires to the four little pads on the back of this board, install `probe-rs`, and suddenly you can see *everything* — panic messages with line numbers, live RTT logs at full speed, register dumps in real time. It's the difference between guessing in the dark and turning on a light. Get one. Today.
{{% /dialog %}}

That's Part 2. We'll buy a $3 ST-Link clone, attach four wires to four pads the size of dust mites, learn what `defmt-rtt` is, and discover that the crash loop is hiding not one but **three** bugs stacked on top of each other — and that fixing the visible one only reveals the next one underneath.

See you in Part 2.

---

*The full source is at [github.com/odisei369/ble-quiz-display](https://github.com/odisei369/ble-quiz-display).*
