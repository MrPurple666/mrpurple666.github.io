---
title: "Building a Game Boy Emulator from Scratch"
permalink: /projects/purplegb/
layout: single
lang: en
locale: en
project: true
translation_key: project-purplegb
alternate_url: /pt/projects/purplegb/
header:
  image: /assets/images/purpleGB.png
---

# PurpleGB: Building a Game Boy Emulator in C

**A single-day deep dive into 8-bit architecture, scanline rendering, and the bugs that only show up when you build it yourself.**

---

If you have spent any time in the emulation scene, you probably know me as MrPurple666. I have contributed to Eden, a Nintendo Switch emulator. I modded Turnip drivers for Mesa3D on Android, pushing mobile GPU compatibility forward. I have debugged shaders, traced syscalls, and stared at more hex dumps than I care to admit.

But there is one thing I had never done: build an emulator from scratch.

I have patched code, profiled bottlenecks, and fixed critical bugs in other people's projects. But the full vertical slice — CPU, memory bus, PPU, APU, cartridge mappers, from power-on to the first pixel — was always someone else's architecture. I knew how emulators worked because I had lived inside them for years. I just had never built one.

That gap bothered me. You can only read so many PPU timing diagrams and Pan Docs revisions before you start wondering whether you truly *understand* or are just very good at pattern-matching other people's implementations.

So I decided to find out.

---

## Why the Game Boy?

The Game Boy is the perfect honesty test. Small enough for one person to finish in a focused session. Roughly 2600 lines of C across 8 source files, a 3-line Makefile. Complex enough that you cannot fake it. Every wrong opcode, every off-by-one PPU cycle, every inverted joypad bit shows up on screen immediately as garbled tiles, silent audio, or crashes that make no sense until suddenly they do.

There is no OS to blame. If the Nintendo logo does not appear, it is because *your* nibble decode is wrong. If Tetris hangs on the title screen, it is because *your* interrupt timing is off. The Game Boy does not care about your resume.

That is why I chose it. Not despite the simplicity, but because of it.

---

## Architecture Overview

PurpleGB is a single-day implementation in C99 with SDL3 for video and input. The architecture is deliberately flat:

| Module | Responsibility | Lines |
|--------|---------------|-------|
| `cpu.c` | SM83 instruction execution, 512 opcodes, flag handling | ~600 |
| `memory.c` | Memory bus, cartridge loading, MBC1/3/5 banking, I/O dispatch | ~400 |
| `ppu.c` | Scanline renderer: background, window, sprites, LCDC/STAT | ~350 |
| `interrupt.c` | IE/IF/IME dispatch, VBlank/STAT/Timer/Serial/Joypad | ~80 |
| `timer.c` | DIV/TIMA/TMA/TAC with clock-select | ~100 |
| `joypad.c` | P1 register, SDL3 key mapping | ~60 |
| `apu.c` | 4-channel audio skeleton (not yet functional) | ~200 |
| `main.c` | SDL3 window, event loop, boot logo, drag-and-drop, tray, FPS | ~700 |

The CPU is a cycle-stepped interpreter. The PPU renders per-scanline, not per-pixel. Memory banking supports MBC1, MBC3, and MBC5 mappers. Audio is stubbed — the registers are there, the SDL3 audio stream is initialized, but the synthesis pipeline is not yet wired.

---

## The Bugs That Matter

### The Striped Screen: Tile Addressing

The first graphical output looked like this:

![First graphical output — black and white stripes](/assets/images/purpleGB/1.png)

*Stripes where Tetris should be. The PPU is alive, the LCD is on, the cartridge is running. But something is deeply wrong with tile fetching.*

The problem was LCDC bit 4. It controls whether background/window tiles use unsigned addressing (`$8000-$8FFF`) or signed addressing (`$8800-$97FF`). I had the logic inverted. Every tile index was being read from the wrong bank, producing the alternating stripes.

Forcing unsigned mode manually produced this:

![Grayscale but still corrupted](/assets/images/purpleGB/1.2.jpg)

*The palette pipeline works. DMG grayscale instead of pure black and white. But the tiles are still wrong. Progress, of a kind.*

### The White Screen of Almost

After fixing tile addressing, everything broke. The screen went white. Not crashed white. The window was open, the event loop running, the ROM loaded. Just white. Every pixel the same color.

![White screen — nothing rendering](/assets/images/purpleGB/2.png)

*The PPU is running, the LCD is on, the framebuffer is being filled with white. Every frame, forever.*

The bug was register synchronization between `memory.c` and `ppu.c`. Game Boy I/O registers live at `$FF00-$FF7F`. The PPU reads them every cycle to know what to render. I had cached copies in the PPU struct that were never updated when the CPU wrote to I/O memory. The PPU rendered with stale values — LCDC still in power-on default, SCX and SCY at zero, the background map pointer pointing nowhere.

Four hours to find. Four hours of printf debugging, verifying every memory write, confirming the CPU was doing exactly what it should. The CPU was fine, the memory bus was fine, the PPU was fine. The connection between them was broken.

That is the kind of bug you only catch when you build the thing yourself.

### Copyright Fragments

After fixing register sync, something magical happened. The screen was no longer white. It was... almost readable. Small text fragments scattered across the top edge, looking like quotes, apostrophes, and pieces of letters.

![First visible tiles — fragments of Tetris copyright](/assets/images/purpleGB/3.png)

*The first recognizable pixels from the cartridge. These are pieces of "© 1989". The background map addressing is still wrong, but tile data is finally reaching the screen.*

Those fragments were the Tetris copyright screen. The game was trying to display "© 1989 Nintendo", and I was receiving quotation marks, apostrophes, and pieces of 9. Like reading a newspaper through frosted glass, but it was something. The tile decoder worked. The background layer rendered. The scroll registers were close.

Then the full screen appeared, and a new problem emerged: it was stuck in a loop. The copyright screen rendered completely, then restarted from the top, then rendered again, then restarted. The VBlank interrupt was firing, but `LY` (the current scanline register) was not incrementing correctly through the VBlank period. The PPU thought it was still on scanline 144, so it kept firing VBlank, which reset the frame counter, which fired VBlank again.

![Copyright screen — stuck in restart loop](/assets/images/purpleGB/4.png)

*The complete copyright screen, finally visible. But it is rewriting itself every few frames, stuck in a loop because LY never advances past the VBlank boundary.*

The fix was a single `break` in the wrong place in the PPU mode cycle. One keyword. Several hours.

### The Colors Were Lying

Tetris reached the title screen. This should have been a celebration. Instead, I got this:

![Title screen with wrong colors](/assets/images/purpleGB/5.png)

*Recognizable, but wrong. The "TETRIS" logo is there, but colors are inverted. Black where light gray should be, light gray where black should be.*

The problem was the BGP register — Background Palette at `$FF47`, a single byte that maps the four DMG colors (0 to 3) to four grayscale shades. Tetris was writing the correct value to BGP, but my emulator was intercepting that write and corrupting it.

Why? Because of a `switch` fallthrough in my I/O write handler. I had grouped several PPU registers together:

```c
// BROKEN: all of these fall through to the DMA handler
case 0x41: case 0x42: case 0x43:
case 0x45: case 0x47: case 0x48: case 0x49:
case 0x46: {
    m->io[0x46] = v;  // Every write becomes a DMA trigger!
    // ... DMA transfer happens here ...
    break;
}
```

`0x46` is OAM DMA, `0x41` is STAT, and `0x47` is BGP. Every time the game wrote to STAT, SCX, SCY, LYC, BGP, OBP0, or OBP1, my emulator treated it as a DMA transfer request. DMA fired, corrupting OAM memory with data from whatever address was in the "source" byte (which was actually the palette value, or scroll position, or STAT flags). That corrupted OAM fed into sprite rendering, which corrupted other memory, which eventually made the game jump to invalid addresses.

One missing `break`. Eight corrupted registers. Countless hours debugging symptoms that had nothing to do with the root cause.

### The Nintendo Logo That Was Not There

The Game Boy boot ROM is famous for its startup sequence. It scrolls the Nintendo logo down the screen, plays the boot sound, and hands control to the cartridge. Since I skip the boot ROM (starting execution at `$0100` instead of `$0000`), I needed to replicate the logo display in software.

The logo is encoded in the cartridge header at `$0104-$0133`, 48 bytes representing the Nintendo logo as a compressed tile map. The boot ROM decodes each nibble through a clever sequence of `RL B` / `RLA` rotations, expanding 4 bits into 8-bit tile line patterns. I simulated this algorithm, precomputed the full 384-byte table, and embedded it as a static array.

The first attempt looked like this.

![First boot logo attempt — corrupted](/assets/images/purpleGB/6.png)

*My first boot logo. The Nintendo logo is almost there. You can see the general shape, the two circular eyes, the curved lines. But the bit patterns are wrong. A single carry flag behavior was incorrect in my nibble expansion.*

It was close. Frustratingly close, maddeningly close. I could see what it should be, but every tile had wrong pixels. The problem was the exact carry flag behavior across four iterations of `RL B` / `RLA`. Z80 documentation is ambiguous in some edge cases, so I had guessed wrong. When I finally matched hardware behavior exactly — tracing each rotation step one by one and comparing against known boot ROM dumps. The logo decoded perfectly.

But it still did not appear on screen.

Because Tetris, like most cartridges, clears VRAM during its initialization routine. My beautifully decoded logo was being erased before the first frame was presented. The fix was a dedicated boot splash state that renders the logo for 90 frames (~1.5 seconds) before starting the CPU:

```c
g->boot_frames = 90;
if (g->boot_frames > 0) {
    ppu_render_frame(&g->ppu, &g->mem);
    g->boot_frames--;
}
```

Simple, and obvious in retrospect. But I spent time convinced my logo decode was still wrong before realizing the cartridge was deleting it.

### The Fix That Fixed Everything

The I/O dispatch fallthrough was not just breaking colors. It was breaking everything. Every write to a PPU register was corrupting OAM, which was corrupting sprites, which was corrupting internal game state, causing crashes, hangs, and graphical glitches that seemed completely unrelated.

When I finally separated the cases — giving STAT, SCX, SCY, LYC, BGP, OBP0, and OBP1 their own proper handlers. Everything aligned at once.

![Tetris title screen — correct](/assets/images/purpleGB/7.png)

*The title screen, finally correct. The TETRIS logo in proper DMG grayscale, the background pattern intact, the window layer (that status bar at the bottom) rendering in the right place.*

And then, the game itself:

![Tetris in gameplay — correct](/assets/images/purpleGB/8.png)

*Tetris, fully playable. Background tiles, window layer, falling block sprites, score display. All rendering correctly after commit `fix: restore LCD register writes and DMA dispatch`.*

I stared at that screen for a while. Not because it was impressive. It is just Tetris, running at 160 by 144 resolution, with four shades of gray. But because I knew exactly what every pixel represented. I knew the opcode that fetched the tile index. I knew the memory address where that tile lived. I knew the bit manipulation that turned two bytes of tile data into eight pixels. I knew the palette lookup that mapped color index 2 to light gray.

I knew because I had built it all, from main() to the last pixel.

---

## Current Status

PurpleGB is a single-day implementation. It is not a showcase emulator. It does not have save states, rewind, netplay, or a fancy shader pipeline. What it has:

| Component | Status |
|-----------|--------|
| SM83 CPU (512 opcodes) | ⚠️ Partial |
| Scanline PPU (BG/Window/Sprites) | ⚠️ Partial |
| Timer (DIV/TIMA/TMA/TAC) | ⚠️ Partial |
| Joypad (P1 register, SDL3 input) | ⚠️ Partial |
| MBC1/3/5 cartridge banking | ⚠️ Partial |
| Interrupt controller (VBlank/STAT/Timer/Serial/Joypad) | ⚠️ Partial |
| Boot logo / Bootix boot ROM path | ⚠️ Partial |
| Drag-and-drop ROM loading | ⚠️ Partial |
| ESC pause overlay | ⚠️ Partial |
| System tray icon | ⚠️ Partial |
| FPS counter | ⚠️ Partial |
| APU (4-channel audio) | ❌ Stubbed — SDL3 stream initialized, synthesis not wired |
| Audio output | ❌ Not functional |

**ROM compatibility:** Tetris runs perfectly. That is the only verified title at this stage. Pokémon Green and other games have not been validated yet.

---

## What This Project Actually Is

PurpleGB will never compete with SameBoy or BGB. Its PPU timing is approximate, not cycle-accurate. Its audio is silent. It will crash on edge-case test ROMs. It was built in a single focused session, not over months of careful hardware testing.

But it taught me something those emulators could not: what it feels like to sit in front of a white screen at 2 AM, knowing that somewhere in 2600 lines of C, a single bit is wrong and you have to find it. What it feels like to see quotation marks and apostrophes scattered on a corrupted screen, and recognize them as fragments of a copyright message you have not yet earned. What it feels like to finally see the Nintendo logo. Not because you loaded a ROM, but because your nibble expansion algorithm, traced through four iterations of RL B and RLA, produced the right bit pattern.

That is why I built it. Not to add another emulator to the world, but to close a gap I had carried for years.

---

## Update: What Changed After This Post

The latest PurpleGB commits did not change the goal of the article; they replaced a few shortcuts that were useful while debugging but wrong as emulator behavior.

- `c304221` replaces the hand-built boot-logo splash path with an embedded Bootix DMG boot ROM. The emulator now starts at `$0000`, runs the boot ROM until it disables itself through `$FF50`, and then exposes cartridge code at `$0100`.
- `d8ca59e` removes cleanup debt: the dead APU model, unused fake boot-logo path, persistent undefined-op tracing, unreachable deferred-DMA state, incomplete tray scaffolding, duplicate ROM-load code, unused PPU SDL include, and unused state fields.
- `1e695fc` fixes PPU color handling by keeping raw BG/window color indices before palette mapping. That matters for sprite priority, because DMG sprite priority checks whether the BG color index is zero, not whether the final grayscale value happens to be white.
- `d6a6a2a` fixes the I/O dispatch bug described above: `$FF41-$FF45`, `$FF47-$FF49`, and `$FF4A-$FF4B` are normal LCD/status/scroll/palette/window registers; only `$FF46` is OAM DMA.

So the historical debugging story above stays as written, but the current emulator no longer relies on the temporary splash-screen workaround or the broken grouped I/O handler that caused the color and DMA corruption.

---

## Acknowledgements

- [Pan Docs](https://gbdev.io/pandocs/) — the canonical Game Boy technical reference. Every register, every flag behavior, every PPU quirk documented in one place.
- [SameBoy](https://sameboy.github.io/) and [BGB](https://bgb.bircd.org/) — reference emulators I compared against when my output looked wrong and I needed to know what "right" looked like.
- [Bootix](https://github.com/Ashiepaws/Bootix) — the boot ROM disassembly in `references/Bootix/bootix_dmg.asm` that let me verify my nibble expansion algorithm against actual hardware behavior.
- [SDL3](https://libsdl.org/) — the framework that handles windowing, input, and the audio stream, so I could focus on the emulation itself.
- The Game Boy emulation community — for decades of reverse engineering, test ROMs, and shared knowledge that makes projects like this possible in a single day.

---

*Source: [github.com/MrPurple666/PurpleGB](https://github.com/MrPurple666/PurpleGB)*
