# 7YP3X-L1N3F33D

**HID Keyboard File Transfer — BadUSB / Flipper Zero / NetHunter**

> Made by **MrBros**

A one-way file transfer system that encodes any file into a stream of typed text strings. A BadUSB device (Flipper Zero, NetHunter, or any DuckyScript-compatible hardware) injects the strings into a target machine's browser. The target receives, verifies, and reassembles the file — no USB drive, no network share, no software installation required on the target.

Built for air-gapped machines, locked-down school or work computers, and VMs with no shared clipboard. If you can plug in a keyboard, you can get a file in.

It was tested in a school environment of loading a word prodject dockument (file input --> phone --> Encoder html --> Ducky script(7kb chunks) --> school rpi 4 client --> win 10 session over lan --> decoder in website -- file output)

I would want to add a demo gif or vid but i wont for personal reasons if someone has tested this prodject please can you make a demo vid demonstrating this prodject.

used claud ai for this prodject would like to convert to fully user made.

tested with flipper zero

---

## Files

| File | Role | Internet needed |
|---|---|---|
| `encoder.html` | **TX-T3RM1N4L v4.0** — Operator machine. Encode, encrypt, export. | Fonts + JSZip only |
| `decoder.html` | **7YP3X::RX v4.2** — Full-featured client decoder. | None — 100% offline |
| `decoder-lite.html` | **7YP3X::RX-L1T3 v4.1** — Minimal client decoder. | None — 100% offline |

> **Important:** Never use `encoder.html` on the target machine. It loads Google Fonts. Use either decoder — they have zero external requests.

---

## How it works

1. Drop your file into **TX-T3RM1N4L** on your own device. Configure chunk size, compression, encryption. Click **ENCODE FILE**.
2. Download the output as a HID strings `.txt` or **DuckyScript** file.
3. Open `decoder.html` or `decoder-lite.html` on the target machine's browser. Focus the paste textarea.
4. Run the DuckyScript on your Flipper Zero / NetHunter. Each chunk is typed as a line of hex characters and auto-processed on ENTER.
5. When all chunks arrive, click **ASSEMBLE & DOWNLOAD**. The file is SHA-verified and saved.

---

## Protocol — ibv1 (13 fields)

Each chunk is a single semicolon-delimited string. Only `0-9 a-f ;` characters — **zero shift keys on any keyboard layout**.

```
ibv1;{idx};{total};{chunkHash};{fileHash};{comp};{compAlgo};{enc};{salt};{iv};{hashAlgo};{hexdata};{nameHex}
```

| Field | Description |
|---|---|
| `ibv1` | Protocol marker — never changes |
| `idx` | 0-based chunk index |
| `total` | Total chunk count |
| `chunkHash` | SHA hash of this chunk's raw bytes (post-encryption) |
| `fileHash` | SHA hash of the original file (pre-compression, pre-encryption) |
| `comp` | `0` or `1` |
| `compAlgo` | `none` / `deflate-raw` / `gzip` / `deflate` |
| `enc` | `0` or `1` |
| `salt` | PBKDF2 salt hex (`0` if not encrypted) |
| `iv` | AES-GCM IV hex (`0` if not encrypted) |
| `hashAlgo` | `sha256` / `sha384` / `sha512` |
| `hexdata` | Chunk bytes as lowercase hex |
| `nameHex` | Original filename as UTF-8 hex (optional — absent = `recovered_file`) |

### Chunk sizing

HID string length ≈ `rawBytes × 2 + 200` chars. The Flipper Zero sweet spot is **2–7 KB HID strings**. The encoder auto-calculates chunk size on file drop, clamped to `[1024–3328]` raw bytes.

---

## Quick Start

1. Open `encoder.html`. Run the **self-test** under the TOOLS tab first.
2. Transfer `decoder.html` to the target machine (USB, email, or inject it with a basic DuckyScript STRING loop).
3. On the target: open the decoder in Chrome or Edge. Click the paste textarea.
4. On the operator: drop your file in. Leave chunk size at auto. Pick a speed preset.
5. Download the DuckyScript. Load it on your Flipper Zero or NetHunter.
6. Plug in, run the script. Watch the chunk map fill up.
7. When all chunks are green — **ASSEMBLE & DOWNLOAD**.

---

## Stealth Mode

All three files include a black-screen overlay that hides the page instantly. The transfer keeps running in the background.

| Platform | Trigger |
|---|---|
| Desktop | Keyboard combo (default: `Ctrl+Shift+H`, customisable) |
| Mobile / tablet | Triple-tap any blank area (3 taps in under 600ms) |

**Status LED** — when stealth is active and the LED is enabled, a tiny coloured dot in the corner mirrors transfer status:

- Orange = receiving
- Green = complete
- Yellow = waiting
- Red = error
- Blue = assembling

**Auto-hide** — activates stealth after N minutes of inactivity. Set in the settings panel (0 = off).

**Panic wipe** (decoder only) — destroys all chunk data instantly on a hotkey, no confirmation. Default: `Ctrl+Shift+W`.

**Custom page title** — rename the browser tab to anything innocent (e.g. "Calculator"). Persists across reloads.

---

## Encryption

> **BETA** — run the self-test with encryption enabled before any real transfer.

- AES-256-GCM with PBKDF2 key derivation (210,000 iterations, SHA-256)
- New random salt + IV generated on every encode
- Password is **never** logged, never stored in localStorage, never embedded in transfer chunks

**Procedure:**

1. Enable encryption in panel 04. Enter or generate a password.
2. After encoding — copy the password, salt, and IV from the result panel. **Store them before you start injection. If lost, the file is unrecoverable.**
3. On the decoder: when all chunks arrive, enter the password and click ASSEMBLE.
4. Wrong password = AES-GCM auth tag failure — clear error, no silent corruption.

---

## Injection Guide

### Speed presets

| Preset | Chars/sec | Delay/chunk | Notes |
|---|---|---|---|
| Flipper (safe) | 300 | 120ms | Recommended default |
| Flipper (fast) | 550 | 80ms | May miss chars on slow targets |
| NetHunter (safe) | 250 | 200ms | Conservative for NetHunter HID |
| NetHunter (fast) | 700 | 80ms | For fast targets only |

### Pre-injection checklist

- [ ] Self-test passed in TX-T3RM1N4L
- [ ] `decoder.html` open on target in Chrome or Edge
- [ ] Paste textarea has keyboard focus
- [ ] DuckyScript delay matches speed estimator
- [ ] Chunk size >= 1 KB
- [ ] If encrypted — password noted and shared with decoder operator
- [ ] Browser tab is not in the background (some browsers throttle background JS)

### Recovering failed chunks

1. After injection, click **RE-VERIFY ALL** to hash-check every stored chunk.
2. In TX-T3RM1N4L, identify failed indices from the chunk manifest.
3. Enter them into the **Re-export** panel (e.g. `0,5,10-20`).
4. Download the targeted DuckyScript and re-inject only those chunks.
5. Re-verify, then assemble.

---

## Troubleshooting

**Hash failures on arrival** — injection speed too high. Lower chars/sec, increase chunk delay, re-inject the failed chunks.

**"IndexedDB unavailable"** — you're likely in incognito/private mode. Open the decoder in a normal tab.

**Chunks not auto-processing on ENTER** — the paste textarea must have focus. Click it before running the script. Add `DELAY 2000` at the start of your DuckyScript.

**"Wrong transfer" error** — a chunk from a different encode arrived. Click **CLEAR TRANSFER** before starting a new session.

**Assembly fails: "File hash MISMATCH"** — run RE-VERIFY ALL to find corrupted chunks, then re-inject them.

**Decryption fails** — wrong password. Check for trailing spaces. Make sure you copied it exactly from the result panel.

---

## Settings Reference

### TX-T3RM1N4L — `localStorage` key: `typex_cfg`

| Key | Default | Description |
|---|---|---|
| `theme` | `iron` | Active colour theme (12 options) |
| `chunkSize` | auto | Raw bytes per chunk — auto-set on file drop |
| `hashAlgo` | `sha256` | `sha256` / `sha384` / `sha512` |
| `compAlgo` | `deflate-raw` | Algorithm when compression is enabled |
| `compress` | `false` | Whether compression defaults to on |

### TX-T3RM1N4L stealth — `localStorage` key: `typex_stealth`

| Key | Default | Description |
|---|---|---|
| `enabled` | `false` | Stealth overlay active |
| `combo` | `Ctrl+Shift+H` | Toggle hotkey |
| `showLed` | `false` | Status LED visible on black screen |
| `animOff` | `false` | Particle canvas disabled |

### 7YP3X::RX — `localStorage` key: `typex_rx_cfg`

| Key | Default | Description |
|---|---|---|
| `title` | `7YP3X::RX` | Page title / h1 label |
| `stealth` | `false` | Stealth enabled |
| `combo` | `Ctrl+Shift+H` | Stealth toggle combo |
| `autohide` | `0` | Auto-hide after N minutes (0 = off) |
| `panic` | `false` | Panic wipe hotkey enabled |
| `pcomb` | `Ctrl+Shift+W` | Panic wipe combo |
| `showLed` | `false` | Status LED on stealth overlay |

### 7YP3X::RX-L1T3 — `localStorage` key: `typex_rxl_cfg`

| Key | Default | Description |
|---|---|---|
| `title` | `7YP3X::RX-L1T3` | Page title / h1 label |
| `stealth` | `false` | Stealth enabled |
| `combo` | `Ctrl+Shift+H` | Stealth toggle combo |
| `autohide` | `0` | Auto-hide timer in minutes |
| `showLed` | `false` | Status LED on stealth overlay |

---

## Security Notes

- Passwords are never logged, never stored in localStorage, never embedded in chunks
- AES-256-GCM + PBKDF2/210k iterations — standard encryption
- File hash verified before download — byte-perfect integrity guarantee
- Per-chunk hashes verified on receipt — corruption caught immediately
- Cross-transfer protection — chunks from different encodes are rejected
- All crypto runs in a Web Worker — main thread never blocked
- Decoder files have zero external dependencies — nothing phoned home

---

## Disclaimer

> **Made for home and personal use only.**
> This tool is intended for transferring your own files to devices you own or have explicit permission to access.
> Using BadUSB / HID injection devices on machines you do not own or are not authorised to access is illegal in most jurisdictions.
> The author accepts no responsibility for misuse.

---

*Made by MrBros — 7YP3X-L1N3F33D*
