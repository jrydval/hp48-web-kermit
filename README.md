# HP-48 Web Kermit

A single-file, no-build-step web app that talks to an HP-48GX / HP-48G+
calculator over its serial port using the Kermit protocol, right from the
browser (Web Serial API). Browse the calculator's file system, download and
upload variables/programs, navigate directories, and delete files — no
desktop software or drivers required beyond a serial-to-USB adapter.

> Status: working prototype, verified against real hardware (see
> [`architektura.md`](architektura.md) for the full development log and
> protocol reverse-engineering notes — currently in Czech).

## Features

- **Connect** to the calculator over a serial port from Chrome/Edge (Web
  Serial API), configurable baud rate (1200–9600), packet size and timeout.
- **Browse** the calculator's directory tree (`REMOTE DIRECTORY`), with
  subdirectories shown separately and clickable to enter them.
- **Download (GET)** any variable or program to your computer — either via
  the download field or the 📥 icon next to each file in the listing.
- **Upload (PUT)** a local file to the calculator under a chosen name.
- **Navigate**: enter a subdirectory, jump straight to `HOME`, or go up one
  level (`UPDIR`).
- **Delete (PURGE)** a variable or (empty) directory, with a confirmation
  prompt and visible error reporting (e.g. attempting to delete a
  non-empty directory).
- **Create a new folder (CRDIR)** under the current directory.
- **Reset connection** (Send-Init + Break) to recover the link after an
  error or interrupted transfer, without needing to restart the Kermit
  server on the calculator itself.
- A live packet log (with an optional "Detailed log" mode for full
  byte-level tracing) for debugging the protocol conversation.
- Correct handling of the calculator's special character set (0x80–0x9F —
  Σ, √, →, α, …) for display, kept strictly separate from the raw bytes
  used on the wire so names with special characters transfer correctly.

## Requirements

- **Browser**: Chrome or Edge on desktop (Windows, macOS, Linux, ChromeOS).
  The Web Serial API is not available in Firefox or Safari, and only
  partially on mobile. A secure context is required (HTTPS, or `localhost`
  during development).
- **Hardware**: an HP-48GX or HP-48G+ with its 4-pin serial port, and a
  USB-to-serial adapter with real RS-232 voltage levels. Cheap TTL
  (0–5 V) adapters are not reliable for this.
- On the calculator: start the Kermit server manually first
  (`I/O` → `SRVR` → `SERVE`) before connecting from the app.

## Usage

1. Open `kermit-test.html` in Chrome or Edge (just open the file, or serve
   it from any static web server — no build step, no dependencies).
2. On the calculator, go to `I/O` → `SRVR` → `SERVE`.
3. Click **Connect** and pick the serial port in the browser's device
   picker. The current directory listing loads automatically once
   connected.
4. Use **List files**, **Download (GET)**, **Send (PUT)**, **New folder**,
   and the navigation buttons to browse and transfer files. Every action
   negotiates its own Send-Init automatically — no manual handshake step
   is needed.
5. If the connection gets stuck (e.g. after an interrupted transfer), use
   **🔄 Reset connection (Break)** to bring the calculator back to an idle
   state without restarting its Kermit server.

## How it works

The app implements the parts of the Kermit protocol the HP-48's built-in
Kermit server actually needs: standard `S`/`F`/`D`/`Z`/`Y`/`N`/`B`/`E`
packets for file transfer, plus the calculator's non-standard `C`
("REMOTE HOST") packet type used for directory listing, navigation, and
deletion — none of which are part of the Kermit specification and were
reverse-engineered against real hardware. See
[`architektura.md`](architektura.md) for the full protocol analysis,
including packet traces, checksum-negotiation behavior, and the sequence
of hardware tests that led to each implementation detail.

## Project structure

- `kermit-test.html` — the entire application (HTML/CSS/JS, self-contained,
  no build step, no external dependencies).
- `architektura.md` — architecture notes and hardware-verification log
  (Czech).

## License

No license has been chosen yet for this project.
