---
title: "Rust on the Other Side of BLE — Part 3: Four Bytes, Four Bars, a Live Demo"
date: 2026-05-16
draft: false
tags: ["rust", "embedded", "nrf52840", "ble", "embassy", "trouble-host"]
series: ["Rust on the Other Side of BLE"]
series_order: 3
summary: "A GATT service in eight lines, a single-task event loop, an LED matrix driver with an XOR cutout trick, and the small army of glue that turns audience taps on phones into bars rising across the room."
---

In [Part 1](/posts/rust-other-side-of-ble-part-1/) we discovered that the nRF52840 SuperMini was nothing like a blank slate. In [Part 2](/posts/rust-other-side-of-ble-part-2/) we got a debug probe, peeled three bugs off the stack, and watched the board finally show up in nRF Connect as `RustQuiz`.

This part is about the firmware that's actually running now that it's running. The interesting code, the things the proc macros do for you, the SPI driver that paints bars onto LEDs, and — at the end — a short tour of the glue around the firmware: a Mac BLE bridge in Rust and a Bun + TypeScript quiz server. The firmware is the protagonist; the rest is supporting cast.

There is also one outstanding mystery from Part 2: the 6.2 MB UF2 file for a 132 KB firmware. We resolve that here too.

## The GATT service in eight lines

A BLE peripheral advertises one or more *services*, each containing *characteristics* — addressable little values that a connected central can read, write, or subscribe to. The whole point of GATT (Generic Attribute Profile) is to put structure on top of "two devices want to send each other bytes."

For the quiz display we need exactly two characteristics:

- **Votes** — four bytes, one per answer option (A, B, C, D), each a percentage from 0 to 100.
- **Control** — two bytes: a command code and a parameter. Used to clear the display, or to mark the correct answer for the blink animation.

The entire GATT definition, in `trouble-host`, looks like this:

```rust
#[gatt_service(uuid = "00001000-b0cd-11ec-871f-d45ddf138840")]
struct QuizService {
    /// Vote percentages: [A%, B%, C%, D%], each 0–100.
    #[characteristic(uuid = "00001001-b0cd-11ec-871f-d45ddf138840", write)]
    votes: [u8; 4],

    /// Control commands: [cmd, param].
    /// cmd=0: clear display. cmd=1: reveal correct answer (param = 0–3).
    #[characteristic(uuid = "00001002-b0cd-11ec-871f-d45ddf138840", write)]
    control: [u8; 2],
}

#[gatt_server]
struct Server {
    quiz: QuizService,
}
```

That's it. Two attributes. Two UUIDs. Two writable byte arrays. Eight or so lines of meaningful Rust.

*An owl, slightly out of breath, lands carrying the BLE 5.3 spec.*

{{% dialog "🦉 Menthor" %}}
What the proc macros generate behind that declaration, for the curious, is a great deal. A GATT attribute table, with handle assignments. A small parser for each characteristic's value type. A `Server` struct with typed accessors — `server.quiz.votes.handle`, `server.quiz.control.handle`, both `u16` attribute handles assigned at startup. And, crucially, an event delivery surface that produces strongly-typed `GattEvent` values, not byte buffers and switch statements. *Ahem.*
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
Look at the UUIDs! They're 128-bit because they're application-specific — anyone can mint one. But the *first* 32 bits, `00001000`, `00001001`, `00001002`, are sequential within our project's UUID base. So you can read the table at a glance: thousand-block is the service, plus-one is votes, plus-two is control. Tiny detail. Doesn't matter to BLE. Matters a lot when you're debugging at 11pm with three terminals open.
{{% /dialog %}}

{{% dialog "🦆 Nestor" %}}
And the `write` attribute on each characteristic — that means the central can change the value but not subscribe to changes from the peripheral, right? We're not pushing anything out.
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
Correct. Indeed, this peripheral is pure sink: it receives, it never notifies. Which simplifies a great deal — no subscription state, no notification queue, no MTU negotiation about how big a notification can be. The central writes, the peripheral acts. That's the entire protocol surface.
{{% /dialog %}}

There is a small philosophical point hiding here that's worth pulling out. With `nrf-softdevice` — the older, traditional Rust BLE library on this chip family — characteristic writes arrive as **synchronous callbacks**: you register a closure, and when a write happens, your closure runs on the BLE event thread, and you have a few milliseconds to handle it before the next event arrives. `trouble-host` does it differently. Events are delivered as **async values**: you `.await` the next event, and when it arrives, you handle it on your own task at your own pace. The difference looks small in code and is enormous in practice. Async events let you write a normal-looking event loop. Callbacks force you into a state-machine-with-shared-state design that is harder to reason about in `no_std`.

We'll see that loop next.

## The single-task event loop

Here is the entire main loop of the firmware, slightly simplified for narrative. It runs forever, handles every connection, and is one function.

```rust
loop {
    // Advertise.
    let adv = Advertisement::ConnectableScannableUndirected {
        adv_data: ADV_DATA,
        scan_data: SCAN_DATA,
    };
    let advertiser = peripheral.advertise(&Default::default(), adv).await?;

    // Wait for a central to connect.
    let acceptor = advertiser.accept().await?;
    let conn = acceptor.with_attribute_server(&server)?;
    info!("Central connected");

    let mut current_votes = [0u8; 4];
    let mut blink_target: Option<u8> = None;
    let mut blink_phase: u8 = 0;

    'session: loop {
        // If we're mid-blink, race the next GATT event against a 500 ms timer.
        let event = if blink_target.is_some() {
            match select(conn.next(), Timer::after_millis(500)).await {
                Either::First(e) => e,
                Either::Second(_) => {
                    blink_phase ^= 1;
                    display.show_with_blink(current_votes,
                                            blink_target.unwrap() as usize,
                                            blink_phase).await;
                    continue 'session;
                }
            }
        } else {
            conn.next().await
        };

        match event {
            GattConnectionEvent::Disconnected { .. } => break 'session,
            GattConnectionEvent::Gatt { event } => {
                if let GattEvent::Write(evt) = &event {
                    if evt.handle() == server.quiz.votes.handle {
                        current_votes.copy_from_slice(evt.data());
                        display.show_bars(current_votes).await;
                    } else if evt.handle() == server.quiz.control.handle {
                        // [cmd, param]: 0 = clear, 1 = blink option `param`, 2 = stop blink
                        // (full match arms in the real code)
                    }
                }
                event.accept()?.send().await;
            }
            _ => {}
        }
    }
}
```

What's interesting about this loop is what's *not* there.

There is no channel between the BLE event source and the display. There is no separate task for display updates. There is no shared state, no mutex, no signal, no notification. The GATT event arrives, we update the display directly in the same `await` chain, we acknowledge the event, we loop. The display calls themselves are `async fn`s — they're SPI writes that yield to the executor while the bytes shift out — but they're called inline.

The reason you can write it this way is that `trouble-host` delivers events as async values, not callbacks. The act of awaiting `conn.next()` is the act of running the connection: the BLE state machine advances, the GATT parser parses the next inbound PDU, and you receive the result as a value you can `match` on. While you're processing the event, the next inbound PDU queues up in the BLE controller's buffers. As long as you process events at the rate they arrive — and writing four bytes to an LED matrix takes about 200 microseconds — there's no back-pressure problem.

{{% dialog "🐦 Penelope" %}}
The `select(conn.next(), Timer::after_millis(500))` is the bit I want to point at. Two futures, racing. *Whichever fires first wins.* If a GATT event arrives, we handle it. If 500 ms passes without an event, we flip the blink phase, redraw, and continue. No timer task, no shared "should I blink?" flag — the entire blink animation lives inside the same event loop, gated on a `select`. That's the *single-task design.* Once you see it you can't unsee it.
{{% /dialog %}}

The whole thing fits in one screen of code. It's the part of the project I'm most happy with, and the part that took the least time to write.

## The MAX7219 driver

The display side, by contrast, is *physical*. Four 8×8 red LED matrices, chained together over SPI, with a MAX7219 driver chip on each. The chain looks like this:

```
nRF52840            MAX7219 #0          MAX7219 #1          MAX7219 #2          MAX7219 #3
─────────           ──────────          ──────────          ──────────          ──────────
MOSI (P0.17) ─────► DIN  ───── DOUT ──► DIN  ───── DOUT ──► DIN  ───── DOUT ──► DIN
SCK  (P0.20) ─┬───► CLK             ├─► CLK             ├─► CLK             ├─► CLK
              │                     │                   │                   │
CS   (P0.22) ─┴─────────────────────┴───────────────────┴───────────────────┘
```

CLK is shared. CS is shared. Data shifts: bytes flow into module #0, and as new bytes arrive, the old bytes shift out the back of module #0 into module #1, and so on. Each module latches whatever happens to be in its 16-bit shift register at the moment CS goes high. So:

- To send a register write to module #0 only, you must send a *no-op* to every later module, because they're all in the chain, all clocking, all latching at the same CS edge.
- To send a register write to module #3 (the last one), you must send module #3's data **first**, so that by the time CS rises, module #3's data has been shifted all the way to the end of the chain.

That second point is the one that confused me for an evening. The naive code is:

```rust
for i in 0..NUM_DEVICES {
    buf[i * 2] = register;
    buf[i * 2 + 1] = data[i];
}
```

That sends module #0's data first, which arrives at module #0 first, which seems right. It is wrong. Module #0's bytes get shifted past module #0 by subsequent bytes, and end up at module #3 by the time CS latches. The correct code reverses the iteration:

```rust
async fn send_raw(&mut self, data: &[u8; NUM_DEVICES], register: u8) {
    let mut buf = [0u8; NUM_DEVICES * 2];
    for i in 0..NUM_DEVICES {
        let dev = NUM_DEVICES - 1 - i;   // last device first
        buf[i * 2] = register;
        buf[i * 2 + 1] = data[dev];
    }
    self.cs.set_low();
    let _ = self.spi.write(&buf).await;
    self.cs.set_high();
}
```

{{% dialog "🐦 Penelope" %}}
*Last device first.* You're sending data destined for the *back of the chain* at the *front of the chain*, so by the time everything has shifted into place, the back-of-chain data is at the back of the chain. Shift registers! Sixteen bits per chip! The whole topology is one long conveyor belt and you load it from one end. I love these things. I love them.
{{% /dialog %}}

### The XOR letter trick

Each module needs to show a letter (A, B, C, or D) with a vote bar drawn over it. The "bar" is a vertical column of lit pixels, height proportional to the percentage — at 0% you see only the letter, at 100% the whole 8×8 is lit, in between you see the bar with the letter cut out of the lit portion.

The naive way is to draw the letter, then erase the pixels the bar covers, then draw the bar around the erased letter. That's three operations per row, and they have to compose in the right order.

The actual code is one operation:

```rust
let bar = if bar_lit[i] & row_bit != 0 { 0b1111_1111 } else { 0x00 };
let letter = LETTERS[OPTION_FOR_MODULE[i]][(row - 1) as usize];
data[i] = bar ^ letter;
```

XOR. For any row where the bar is lit, every pixel of that row turns on — *except* the pixels where the letter glyph has a `1`, which get flipped back to `0`. The letter becomes negative space inside the bar. For any row where the bar isn't lit, the letter renders normally on a dark background.

```text
At 0% — letter only:                  At 100% — letter as cutout:
  ░ ░ ░ ░ ░ ░ ░ ░                       ■ ■ ■ ■ ■ ■ ■ ■
  ░ ░ ░ ■ ■ ░ ░ ░                       ■ ■ ■ ░ ░ ■ ■ ■
  ░ ░ ■ ░ ░ ■ ░ ░                       ■ ■ ░ ■ ■ ░ ■ ■
  ░ ░ ■ ░ ░ ■ ░ ░                       ■ ■ ░ ■ ■ ░ ■ ■
  ░ ░ ■ ░ ░ ■ ░ ░                       ■ ■ ░ ■ ■ ░ ■ ■
  ░ ░ ■ ■ ■ ■ ░ ░                       ■ ■ ░ ░ ░ ░ ■ ■
  ░ ░ ■ ░ ░ ■ ░ ░                       ■ ■ ░ ■ ■ ░ ■ ■
  ░ ░ ■ ░ ░ ■ ░ ░                       ■ ■ ░ ■ ■ ░ ■ ■
```

One line of arithmetic. No state machines, no conditional rendering. XOR is the right primitive for "draw A over B with B as a cutout" and embedded code can use it directly without a graphics library involved.

There's also a chain-order subtlety: the four modules display answers `D C B A` from left to right (because that's how the wiring works out — module #0 is physically rightmost in the room layout, and we wanted A on the right). The driver has a constant:

```rust
const OPTION_FOR_MODULE: [usize; NUM_DEVICES] = [3, 2, 1, 0];
```

Flip it to `[0, 1, 2, 3]` if you wire your modules the other way. Everything else in the driver indexes through this constant, so the wiring layout is one constant change.

### The reveal animation

When the presenter clicks *Reveal*, the firmware receives a `[0x01, N]` write on the Control characteristic — start blinking option N as the correct answer. The blink is implemented by alternating the chosen module between its actual bar and a vertically-inverted bar:

```rust
pub async fn show_with_blink(&mut self, pct: [u8; 4], blink_option: usize, phase: u8) {
    let mut bar_lit = bars_from_pct(pct);
    if let Some(module) = module_for_option(blink_option) {
        if phase % 2 == 1 {
            bar_lit[module] = !bar_lit[module];   // bitwise NOT — flip every row
        }
    }
    self.render(bar_lit).await;
}
```

At 0% this becomes a clean letter-vs-fully-lit flash. At 100% it becomes the inverse: fully-lit-vs-letter-only. At intermediate percentages the bar appears to oscillate top-to-bottom. Three lines of arithmetic; one of the better visual effects in the project.

The phase advance, recall, lives in the main loop's `select` — every 500 ms with no incoming GATT event, the phase flips and `show_with_blink` is called again with the new phase.

## The 6.2 MB UF2 file (the teaser from Part 2)

If you build the firmware and try to convert it straight from ELF to UF2 — the obvious thing to try, the thing every quick tutorial implies you can do — you get this:

```bash
$ uf2conv target/thumbv7em-none-eabihf/release/ble-quiz-display \
          --base 0x00000000 -o firmware.uf2

$ ls -lh firmware.uf2
-rw-r--r--  1 user  staff   6.2M firmware.uf2
```

A 6.2 MB file. For a 132 KB firmware. On a chip with 1 MB of total flash.

```bash
$ cargo size --release
   text    data     bss     dec     hex
 126684    4872    9804  141360   22830
```

132 KB. *Where did the other 6 MB come from?*

The answer is: an ELF file is a description of how segments should be loaded into memory, and the nRF52840's memory map has a 512 MB hole in it.

```
0x00026000 ──┬────────────────  start of flash segment (.text + .data init)
             │  ~132 KB of code & data
0x00046000 ──┴────────────────  end of flash segment
   ...
   (512 MB of nothing)
   ...
0x20000000 ──┬────────────────  start of RAM segment (.bss)
             │  ~10 KB of zero-initialized RAM
0x20001300 ──┴────────────────  end of RAM segment
```

An ELF cheerfully describes both: flash at `0x00026000`, RAM at `0x20000000`. When `uf2conv` reads the ELF and sees loadable segments at addresses 512 MB apart, it produces UF2 blocks spanning the entire range. UF2 is a block-based format — 512 bytes per block — so a sparse 512 MB address range fills out to a lot of blocks, almost all of them empty.

The fix is to extract the flash content as a raw binary first, *then* convert:

```bash
# Step 1: ELF → raw binary (strips debug info, keeps only loadable flash content)
$ cargo objcopy --release -- -O binary firmware.bin
$ ls -lh firmware.bin
-rw-r--r--  1 user  staff   132K firmware.bin

# Step 2: raw binary → UF2 (with correct base address and chip family)
$ uf2conv firmware.bin --base 0x26000 --family 0xADA52840 -o firmware.uf2
$ ls -lh firmware.uf2
-rw-r--r--  1 user  staff   263K firmware.uf2
```

263 KB. That's the firmware plus UF2's wrapping overhead (each 256-byte payload chunk is wrapped in a 512-byte UF2 block; roughly 2× the raw binary). Reasonable.

**A gotcha for the UF2 path:** `--base 0x26000` must match `FLASH ORIGIN` in `memory.x`. If you change one and not the other, the UF2 will be written to the wrong address and the firmware will crash on boot — silently, because, you'll recall from Part 1, the UF2 bootloader does not warn about that.

`--family 0xADA52840` is the UF2 family ID for the nRF52840-with-Adafruit-bootloader variant. The bootloader checks it and rejects UF2 files for other chips. A wrong family ID is one of the few things that will produce a visible error: the file copies to the drive, the drive remounts immediately without flashing, and `INFO_UF2.TXT` shows up unchanged. That's the "I refused" signal.

## The rest of the system, briefly

The firmware is one of three programs in this project. The other two exist so that real human beings can vote on real questions from real phones, and have the bars rise on the LED display in real time, without anybody having to open nRF Connect.

```
Audience phones        Quiz server          BLE bridge          nRF52840
(browser)              (Bun + TS)           (Rust + btleplug)   (this firmware)
    |                       |                     |                       |
    |─── POST /vote ───────►|                     |                       |
    |                       |◄─── GET /results ───|                       |
    |                       |─── JSON ───────────►|                       |
    |                       |                     |─── BLE Write ───────►|
    |                       |                     |   [70, 20, 8, 2]      |
    |                       |                     |                       |─► SPI ─► LEDs
```

### The Mac BLE bridge (Rust + `btleplug`)

The bridge is a small Rust binary that runs on the presenter's laptop. It scans for `RustQuiz` over BLE, connects, and then polls the quiz server every 2 seconds:

```rust
let results = fetch_results(http, results_url).await?;

peripheral
    .write(&votes_char, &results.votes, WriteType::WithResponse)
    .await?;

let desired = if results.revealed && results.correct < 4 {
    Some(results.correct)
} else {
    None
};

if desired != last_blink_target {
    let payload = match desired {
        Some(target) => [CONTROL_BLINK_START, target],
        None         => [CONTROL_BLINK_STOP, 0],
    };
    peripheral
        .write(&control_char, &payload, WriteType::WithResponse)
        .await?;
    last_blink_target = desired;
}
```

Two things are worth pointing out:

- The bridge does **rising-edge detection** on the `revealed` flag from the server. It only writes to the Control characteristic when the flag *changes*. Otherwise it'd be sending duplicate "start blinking" commands twice a second, which would work but would be obnoxious on the air.
- It uses [`btleplug`](https://github.com/deviceplug/btleplug), the cross-platform Rust BLE library. On macOS this becomes a thin wrapper around CoreBluetooth; on Linux, BlueZ; on Windows, the WinRT Bluetooth APIs. It is the only sane way to write a desktop BLE central in Rust right now.

The bridge also handles disconnects gracefully: any BLE error drops it back to the outer scan-and-connect loop. The peripheral can be power-cycled mid-demo and the bridge will reconnect within a few seconds.

### The quiz server (Bun + TypeScript)

The server is the simplest part of the system. It's a single-file Bun app that serves three pages and a handful of JSON endpoints:

- `GET /` — the audience voting page. Each browser gets a session cookie, picks a name, taps a vote.
- `GET /admin` — the presenter's control panel. Locks voting, reveals the answer, advances to the next question.
- `GET /results` — the JSON endpoint that the bridge polls. Returns `{ votes: [a,b,c,d], correct, revealed, ... }`.

All state is in-memory. There is no database. If the server restarts mid-demo, the question stack and the player list are gone. That is fine for a 20-minute demo on a presenter's laptop and would be wrong for anything bigger. We are pricing in the smallness.

The question set is hard-coded in `server.ts`, four questions about embedded Rust:

```typescript
const QUESTIONS: Question[] = [
  {
    text: "Which crate provides the async runtime on bare-metal Rust?",
    options: ["tokio", "Embassy", "async-std", "smol"],
    correct: 1,
  },
  // ...
];
```

This is the entire interaction model. The audience taps. The server tallies. The bridge polls. The firmware writes bytes. The LEDs light up. Each layer is small enough that you can read its source in one sitting.

## What embedded Rust actually feels like

I want to close on something honest about the experience of building this.

A pitch you hear often is *"Rust is memory-safe, so embedded Rust is safer than embedded C, so embedded Rust is easier than embedded C."* The first half of that is true. The second half is not, or at least not the way the pitch implies.

Embedded Rust is not easier than embedded C. It is **safer** than embedded C — the borrow checker catches a real class of bugs that take hours to find with `gdb` and a logic analyzer. But the world it operates in is identical: the same linker scripts, the same chip errata, the same vendor SDKs translated to Rust, the same datasheets full of register tables, the same five hours of "why doesn't this peripheral respond" ending in *"oh, the clock to that peripheral was off the whole time."*

What changes is *the texture of the work*. In embedded C, you usually find out your code is wrong when the chip crashes and you spend two hours figuring out which pointer was bad. In embedded Rust, you usually find out your code is wrong when it doesn't compile, and you spend two hours figuring out which lifetime annotation will make the borrow checker stop complaining. The bugs move from runtime to compile time. The frequency of bugs goes down. The frustration of each individual bug, in my experience, goes up — because the bugs that remain are the ones the compiler can't help with, and those are the gnarly ones.

{{% dialog "🦆 Nestor" %}}
Three weeks ago I bought a six-dollar chip and thought I'd write some Rust on it. I have now written nine pages of debugging notes, ordered a $3 probe, destroyed a `VDD` pad, learned the BT spec's address-type encoding, and reasoned about ELF segment layouts.
{{% /dialog %}}

{{% dialog "🐦 Penelope" %}}
And the BLE works. The bars rise. The phones vote and the LEDs respond and *everyone in the room can see it* and *every byte in that firmware is a byte you understand.* You wrote it, you placed it, you flashed it, you watched it run. Try buying that experience as a service. You can't! It's not for sale!
{{% /dialog %}}

{{% dialog "🦉 Menthor" %}}
The deeper reward, in my view, is that the abstraction stack collapses. There is no operating system between you and the radio. No driver layer between you and the SPI peripheral. No allocator between you and the bytes. The thing that happens when you write `display.show_bars([70, 20, 8, 2])` is *exactly* what you think happens: bytes get clocked out on a wire, latches latch, LEDs light up. Few activities in modern software have that property anymore. *Ahem.*
{{% /dialog %}}

The trade-off, of course, is that you can no longer pretend the abstractions don't exist. The MBR is real. The SoftDevice region is real. The HCI status codes are real. The 1.5 mm pitch of the SWD pads is real. You spend a lot of time learning things that, in a higher-level system, somebody had already learned for you. Some of it is genuinely interesting and some of it is the kind of detail you wish you could buy your way out of.

But when it works, you *own it*. There's nothing in the binary you didn't put there. The 132 KB on flash is 132 KB you wrote (or linked, or knowingly imported). You can read every line. You can explain every byte. That ownership, more than the safety, is what keeps me coming back.

## See it run

The full project is at [github.com/odisei369/ble-quiz-display](https://github.com/odisei369/ble-quiz-display). Firmware in [`src/`](https://github.com/odisei369/ble-quiz-display/tree/main/src), Mac bridge in [`bridge/`](https://github.com/odisei369/ble-quiz-display/tree/main/bridge), and quiz server in [`quiz-server/`](https://github.com/odisei369/ble-quiz-display/tree/main/quiz-server).

If you want to build something similar, the parts list is:

- nRF52840 SuperMini (~$6)
- 8×32 MAX7219 LED matrix module (~$5)
- ST-Link V2 clone (~$3)
- Four thin wires, a soldering iron, and a willingness to destroy at least one component on the way to a working prototype

About $14 of hardware, plus a weekend or two.

That's the series. Thanks for reading. If there's a Part 4 it will be about porting this to ESP32-C6 — which has its own particular flavor of *"the board is not a blank slate,"* and which would let me compare what's portable in the `embassy` ecosystem from one chip family to the next. We'll see.

Until then — go buy a chip, get a probe before the chip arrives, and read `INFO_UF2.TXT` *first*.

---

*Source: [github.com/odisei369/ble-quiz-display](https://github.com/odisei369/ble-quiz-display). Catch up on the series: [Part 1: The Board Is Not a Blank Slate](/posts/rust-other-side-of-ble-part-1/) · [Part 2: The Reset Loop That Lied](/posts/rust-other-side-of-ble-part-2/).*
