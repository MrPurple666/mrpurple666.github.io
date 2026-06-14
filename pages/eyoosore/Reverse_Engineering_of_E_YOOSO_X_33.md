---
title: "Reverse Engineering the E-YOOSO X-33"
permalink: /projects/eyoosore-en/
header:
  image: /assets/images/eyoosore1.png
---

# Reverse Engineering the E-YOOSO X-33: From Windows Driver to Linux Control App

**A journey through USB HID, Ghidra, and undocumented protocols**

---

I bought an E-YOOSO X-33 programmable gaming mouse. It's a respectable piece of hardware. PMW3335 sensor, 16 programmable buttons, RGB lighting, 5 DPI levels, macro recording, the works. But like most of these Chinese-manufactured peripherals, the configuration software only runs on Windows.

I use Linux (CachyOS, an Arch derivative). So I did what any self-respecting Linux user would do: I decided to build a Linux configuration tool from scratch. What followed was a deep dive into USB HID, Windows PE reverse engineering, EEPROM protocols, and GTK4 development.

This post documents the journey.

---

## The Target

The official Windows software is distributed as an Inno Setup installer: `E-YOOSO X-33 Setup v3.1 20231115(2).exe`. Inside:

- `OemDrv.exe`: the main configuration application
- `Lowerdev.dll`: a thin HID communication layer
- `InitSetup.dll`, `MenuEx.dll`: supporting UI libraries
- `Cfg.ini`: configuration storage
- `Text/`: multilingual string tables (English, Chinese, Russian, Portuguese)

The `Cfg.ini` gave me the first clues:

```ini
VID=0x25A7
PID=0xFA08
PID2=0xFA07
Sensor=0x3335
DM=5
DPI=1000,2000,4000,8000,16000
K1_1=0x02,0x31,0x00,0x01
K2_1=0x02,0x32,0x00,0x02
...
K16_1=0x01,0x13,0x00,0x07
```

Two PIDs: `0xFA08` for wired mode, `0xFA07` for the wireless 2.4GHz receiver. The `K{n}_{m}` entries encode 16 button mappings in format `type, keycode, extended_param, hardware_idx`. The sensor is PMW3335.

## Extracting the Windows Driver

Unpacking Inno Setup installers on Linux is straightforward with `innoextract`:

```bash
innoextract "E-YOOSO X-33 Setup v3.1 20231115(2).exe"
```

This gave me the full application tree: DLLs, the main executable, skin images, and the text files. I converted the UTF-16 LE XML files to readable text and had a complete map of every UI string the Windows app uses.

## First Contact: USB Enumeration

The mouse appears on the USB bus as two different devices depending on connection mode:

```
Wired:     ID 25a7:fa08  "2.4G Dual Mode Mouse"
Wireless:  ID 25a7:fa07  "Compx 2.4G Wireless Receiver"
```

Both expose two HID interfaces:

- **Interface 0**: standard HID mouse (5 buttons + wheel + horizontal scroll)
- **Interface 1**: HID keyboard + vendor-specific configuration

The configuration interface's HID report descriptor revealed the communication channels:

```
Report 0x01: INPUT  (7B)    keyboard modifier + key codes
Report 0x02: INPUT  (7B)    vendor status / battery
Report 0x03: INPUT  (1B)    system control
Report 0x05: INPUT  (2B)    consumer control
Report 0x06: FEATURE(7B)    general config (profile / polling)
Report 0x08: FEATURE(16B)   vendor data (initially thought to be buttons)
Report 0x09: INPUT  (16B)   vendor input
```

My first assumption was that `SET_REPORT` on report `0x08` would configure button mappings. I spent two days testing every combination of buffer sizes, report IDs, and transfer types. The writes always returned success at the USB level. But the values never stuck. Readback always returned all zeros.

## The `@property` That Cost Me Hours

During development, I accidentally dropped the `@property` decorator from `is_connected` during a refactor. This meant `self._mouse.is_connected` returned a bound method object instead of `True`/`False`. The `_scan_devices` timer never detected the mouse, showing "Disconnected" even with the device plugged in and the CLI proving communication worked.

One missing decorator. Hours of debugging. Classic.

## Lowerdev.dll: A Dead End

I disassembled `Lowerdev.dll` and found that it's purely a thin wrapper:

```c
// These are literally just forwarded calls
SetFeature       -> HidD_SetFeature
GetFeature       -> HidD_GetFeature
SetOutputReport  -> HidD_SetOutputReport
GetInputReport   -> HidD_GetInputReport
```

Zero protocol logic. The real magic is in `OemDrv.exe`.

## Ghidra to the Rescue

I loaded `OemDrv.exe` into Ghidra (with the MCP bridge for programmatic access, which was a game-changer). The binary is a Delphi-compiled Win32 GUI application. It's huge, with spaghetti control flow typical of VCL event-driven code.

The first breakthrough came from tracing the HID wrapper loader:

```c
void LoadHidWrappers(HMODULE hLowerdev) {
    g_HidApi.OpenHidDevice   = GetProcAddress(hLowerdev, "OpenHidDevice");
    g_HidApi.SetFeature      = GetProcAddress(hLowerdev, "SetFeature");
    g_HidApi.GetFeature      = GetProcAddress(hLowerdev, "GetFeature");
    g_HidApi.SetOutputReport = GetProcAddress(hLowerdev, "SetOutputReport");
    g_HidApi.GetInputReport  = GetProcAddress(hLowerdev, "GetInputReport");
}
```

The application opens **two** HID handles, not one:

```c
// Command handle
HANDLE hCmd = OpenHidDevice(VID, PID, usagePage=0xFF02, usage=2);

// Notify handle  
HANDLE hNotify = OpenHidDevice(VID, PID, usagePage=0xFF01, usage=0);
```

This was key. `0xFF02` and `0xFF01` correspond to the two vendor-specific collections in the HID report descriptor. The Windows driver opens them by usage page, not by interface number. On Linux, I mapped the command node to `/dev/hidraw3` (wireless) or `/dev/hidraw5` (wired). Whichever hidraw node responds to a `GET_FEATURE` on report `0x06`.

## The Real Protocol: EEPROM Commands

The next breakthrough came from decompiling the command packet helper:

```c
void SendCommandPacket(HANDLE hCmd, uint8_t cmd, int body...) {
    uint8_t buf[17];
    buf[0] = 0x08;  // report ID
    buf[1] = cmd;   // command ID
    // ... body bytes ...
    buf[16] = 0x55 - sum(buf[0..15]);  // checksum
    SetFeature(hCmd, buf, 17);
}
```

The actual communication is a **17-byte command protocol**, not standard HID feature reads:

| Byte | Purpose |
|------|---------|
| 0 | Report ID (always `0x08`) |
| 1 | Command ID |
| 2-15 | Command-specific payload |
| 16 | Checksum (`0x55 - sum(bytes[0..15])`) |

Commands I identified:

| Command | Function |
|---------|----------|
| `0x03` | GetConnectStatus (preflight before any write) |
| `0x07` | WriteEEPROM |
| `0x08` | ReadEEPROM |
| `0x09` | Commit / Apply |

Commands `0x01` (handshake), `0x02`, and `0x04` also exist but I haven't fully mapped them yet.

Replies come back as `0x09`-prefixed packets on the same hidraw node:

```
09070000020253000000000000000000ee
|  | |  |
|  | |  +-- checksum
|  | +----- echoed payload
|  +------- status byte
+---------- reply marker (0x09)
```

The key insight: every EEPROM write is acknowledged by a reply packet with the same format as the request.

## EEPROM Memory Map

From the main "Apply" function in `OemDrv.exe`, I reconstructed the EEPROM layout:

| Address | Size | Content |
|---------|------|---------|
| `0x0000` | 2B | Polling rate code + checksum |
| `0x000C` | 4B x 5 | DPI table (5 levels, 4 bytes each) |
| `0x002C` | 4B x 5 | DPI color table (R,G,B + checksum per level) |
| `0x0060` | 4B x 16 | Key matrix (16 buttons, 4 bytes each) |
| `0x0100`+ | variable | Per-key extra blobs (keyboard combos) |
| `0x0300`+ | variable | Macro storage (16 slots) |

The DPI table uses a sensor-specific lookup. I extracted the full `0x3335` DPI code table from `OemDrv.exe`:

```python
SENSOR_3335_DPI_TO_CODE = {
    100: 0x00, 200: 0x02, 300: 0x03, 400: 0x04, 500: 0x05,
    ...
    4000: 0x2F, ..., 16000: 0xBD,
}
```

The key matrix entry format encodes the function:

```c
// Mouse function entries
LeftButton    -> 0x000101
RightButton   -> 0x000201
FireKey       -> 0x000401
SniperButton  -> 0x000801

// Simple keyboard key
SingleKey     -> 0x000005  // plus a 7-byte blob at (index+8)*0x20
```

## The Wake Problem

Wireless mice sleep to save battery. When sleeping, the receiver accepts command packets but never sends ACKs. I discovered this empirically: writes that worked after mouse movement failed when the mouse had been idle for a few minutes.

The fix: open the mouse input node (the one that returns `Broken pipe` for feature reports but streams input reports when the mouse moves) and read from it before each write operation. This generates USB traffic that wakes the mouse's radio:

```python
def _wake_mouse(self):
    """Read from input node to wake wireless mouse."""
    for _ in range(3):
        try:
            data = os.read(self._notify_fd, 64)
            if data:
                return True
        except BlockingIOError:
            pass
        time.sleep(0.1)
    return True
```

## Building the Linux Application

The Linux tool is a GTK4 / libadwaita application in Python. Architecture:

```
eyooso_mouse/
├── mouse_protocol.py    # HID descriptor parsing, EEPROM packet protocol,
│                        # hidraw ioctl transport, DPI/sensor tables
├── config_manager.py    # Cfg.ini parser, JSON profile save/load
├── main.py              # GTK4 AdwApplicationWindow with tab navigation
└── ui/
    ├── main_page.py     # Interactive mouse visual (Cairo rendering),
    │                    # button mapping panel
    ├── advanced_page.py # DPI editor (5 levels + color swatches),
    │                    # polling rate, sensitivity, scroll
    ├── lighting_page.py # LED modes (steady/breathing/neon/streaming/off),
    │                    # brightness, speed, color picker
    └── macro_page.py    # Macro recording, editing, import/export
```

Technical details worth noting:

- Uses raw **hidraw** kernel interface (`/dev/hidraw*`) via `fcntl.ioctl` with `HIDIOCGFEATURE` / `HIDIOCSFEATURE`, not `pyusb` or `libusb`. The hidraw path handles report IDs correctly (first byte of the buffer) and returns complete packets.
- Implements the exact checksum algorithm from the disassembled binary: `0x55 - sum(bytes[0:15])`.
- Retry logic with adaptive timeouts for wireless ACK latency.
- 31 pytest unit tests covering packet building, checksums, DPI table lookup, button matrix encoding, and config serialization.

## What Works (June 2026)

| Feature | Wireless (0xFA07) | Wired (0xFA08) |
|---------|-------------------|-----------------|
| Profile switching | ✅ | ✅ |
| Polling rate write | ✅ | ✅ |
| DPI table write (0x3335) | ✅ | ✅ |
| DPI color write | ✅ | ✅ |
| Key matrix write (mouse fns) | ✅ | ✅ |
| Simple keyboard keys | ✅ | ✅ |
| Complex key combos | ❌ | ❌ |
| Macro recording/write | ❌ | ❌ |
| LED mode write | ❌ | ❌ |
| Battery read | ❌ | ❌ |

## What's Still Missing

The macro protocol and complex key combos use a binary blob format I haven't fully decoded. The LED write path exists in the decompiled `OemDrv.exe` but references addresses I haven't mapped. Battery and firmware version readback require the notify handle thread, which I've only partially implemented.

The Windows driver also implements a firmware update mechanism (`ShowFWUpdate` in `Cfg.ini`) and a sniper-button DPI menu, neither of which I've ported yet.

## What I Learned

This project taught me more about USB HID than I ever expected to know:

- **USB control transfers vs hidraw**: `pyusb` control transfers work, but hidraw ioctls are cleaner on Linux. The kernel assembles complete HID report packets for you.
- **HID report descriptors tell you what's declared, not what works**: the device accepts `SET_REPORT` with report IDs that don't appear in the descriptor, as long as the buffer format matches the transport.
- **Wireless receivers cache state**: feature report reads return cached values, not live device state. The mouse's actual config lives in its internal EEPROM and is forwarded wirelessly.
- **Ghidra's decompiler is remarkably good**: even on Delphi-compiled binaries with messy VCL control flow, the reconstructed pseudocode was accurate enough to port packet construction logic directly to Python.
- **Sleeping devices are a real problem**: wireless peripherals aggressively sleep to save power, and you need to generate USB traffic on the input node to wake them before configuration commands are accepted.

## Where We Stand — And Why It's Not Done Yet

Let me be honest: **this project is still experimental and not fully functional as a daily driver.**

The Linux tool compiles, the code is solid, the packet protocol is correctly implemented, and the EEPROM writes are acknowledged by the mouse. But there's a gap between "the mouse ACKs the write" and "the mouse actually changes its behavior."

Here's what's still blocking us:

1. **The commit condition is fragile.** Sometimes the device accepts all writes and the profile switches instantly. Other times, especially when the mouse is in wireless mode and hasn't been moved recently, the same exact packet sequence gets ACK'd at the USB level but the mouse firmware ignores the changes. The official Windows driver seems to have a more complex handshake sequence that we haven't fully decoded, possibly involving command `0x01` (which sends challenge bytes) and a response validation step.

2. **The mouse's internal state machine is undocumented.** After a write-commit cycle, the mouse enters a "programming mode" briefly. During that window, the receiver stops forwarding mouse input reports to the host. If you move the mouse during a config write, you get no cursor movement for about 1.5 seconds. The Windows driver accounts for this timing. Our Linux implementation doesn't yet.

3. **Wireless vs. wired behave differently.** When connected via USB cable (PID `0xFA08`), the mouse talks directly to the host and the command node is always responsive. But via the 2.4GHz receiver (PID `0xFA07`), the receiver acts as a proxy. It buffers commands and forwards them to the mouse when the radio wakes up. This buffering introduces latency and sometimes outright packet loss. We've added retries, but we don't yet have the receiver's buffer management protocol.

4. **The macro and LED protocols are still black boxes.** We know the EEPROM addresses where macro data lives (starting at `0x0300`) and we can see the blob-builder functions in the decompiled binary. But the internal format (how key events, delays, and loop controls are encoded) is still being reverse-engineered. Same for the LED mode writes.

I'm not giving up, but I also won't pretend this is close to done. The gap between "the mouse ACKs the write" and "the button actually changes what it does" is not a small one. It's the difference between talking to the receiver and talking to the firmware running inside the mouse itself. That firmware doesn't have a datasheet. It doesn't have an SDK. The only documentation is a disassembled Delphi binary and a lot of trial and error.

The missing pieces are:
- handshake validation
- internal buffer management in the receiver
- the macro encoding format

These are, individually, weeks of reverse engineering. And any one of them could easily be the piece that everything else depends on.

**Next post: I'll share whatever progress I make. No promises on timeline. This is weekend hacking, not a startup. But I'm stubborn enough to keep picking at it.**

See you then. Probably not soon. But eventually.

---

*Written by a Linux user who got tired of rebooting to Windows just to remap a mouse button.*
