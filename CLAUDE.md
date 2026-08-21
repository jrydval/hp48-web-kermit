# CLAUDE.md

Persistent instructions for Claude Code working in this repository. Read
this at the start of every session — it captures conventions and history
that would otherwise have to be rediscovered each time.

## What this is

**HP-48 Web Kermit** — a single-file, no-build-step web app
(`kermit-test.html`) that talks to an HP-48GX / HP-48G+ calculator over
its serial port using the Kermit protocol, from the browser (Web Serial
API). Lets you browse the calculator's file system, download/upload
variables and programs, navigate directories, create/delete folders, and
delete variables — no desktop software or drivers beyond a serial-to-USB
adapter.

This project was migrated here from a Claude Projects session on
claude.ai (project "Kermit Web Client pro HP-48GX"); there is no
automatic sync back to that Project — treat this repo as the source of
truth going forward.

## Files

- `kermit-test.html` — the entire application (HTML/CSS/JS, self-contained,
  vanilla JS, no build step, no external dependencies, no npm packages).
  UI text, comments, and log messages are in English.
- `architektura.md` — **the primary design/history document, written in
  Czech.** A numbered log of every hardware test and protocol discovery,
  in chronological order (chapter 11, "Ověřeno na hardwaru"). This is the
  ground truth for *why* the code does what it does — read it before
  changing any protocol-level logic (packet building, checksum handling,
  navigation/deletion commands). Keep writing to it in Czech, in the same
  numbered-entry style, whenever you make a protocol-relevant change or
  the user reports a new hardware-test result. Don't translate it to
  English unless the user explicitly asks.
- `README.md` — short English overview for GitHub (features, requirements,
  usage, links back to `architektura.md` for the full history). Keep this
  in sync (feature list, usage steps) whenever user-visible behavior
  changes — it's the first thing an outside reader sees.

## Working conventions (established over many sessions — keep following them)

1. **Hardware-first verification loop.** New HP-48 protocol behavior is
   normally: (a) tested first with the reference `kermit` (C-Kermit) CLI
   tool by the user, packet trace captured; (b) the trace gets analyzed
   precisely; (c) matching functionality gets implemented in
   `kermit-test.html`; (d) the user tests on real hardware and reports
   back exact log output. There is no simulator and no automated test
   suite — everything is verified against a physical HP-48GX. Don't
   guess at calculator behavior; if something is unverified, say so
   explicitly (in code comments and in `architektura.md`, e.g. "**Zatím
   netestováno na hardwaru**").
2. **Document as you go.** Every protocol discovery, bugfix, or
   UI-affecting change gets a new numbered entry appended to
   `architektura.md` (Czech), describing: what was tried/reported, root
   cause analysis, and the fix. This history has repeatedly mattered —
   e.g. a stray stack value during one early test masked a real bug
   (`HOME EVAL` failing on a clean stack) for many sessions until it was
   caught later on real hardware.
3. **Syntax-check after every edit.** This file has no build step or
   linter wired up, so after editing the `<script>` block, extract and
   check it before calling the work done:
   ```bash
   awk '/<script>/{flag=1;next}/<\/script>/{flag=0}flag' kermit-test.html > /tmp/check.js
   node --check /tmp/check.js
   ```
4. **Keep all DOM element IDs stable.** The JS wires up behavior purely
   by `document.getElementById`. UI restructuring (layout, CSS, visual
   grouping) is fine and has happened before (e.g. the sidebar-dashboard
   redesign), but existing `id="..."` attributes referenced from JS must
   not change without updating every reference — grep for the id first.
5. **Raw HP-48 bytes vs. display Unicode — never mix them.** The
   calculator uses byte range 0x80–0x9F as its own math/Greek symbol set
   (Σ, √, →, α, …), not standard C1 control codes. `hp48ToUnicode()`
   converts these for *display only* (rendered HTML/log text). Text that
   has been through `hp48ToUnicode()` must never be fed back into
   protocol-building code (`buildPacket`, `sendHostCommand`, etc.) —
   doing so silently corrupts data (a real bug that shipped once: a
   variable named `ΣPAR` failed to delete because the Unicode-mapped
   name got truncated back to a single byte). Directory-listing entries
   (`e.name`) are kept as raw HP-48 byte strings for exactly this
   reason; `hp48ToUnicode()` is applied only at final render time.
   Outgoing text (RPL commands, filenames) gets quoted via `quoteText()`
   / `quoteBytes()` before transmission.
6. **Command names vs. literals in `REMOTE HOST` (`C` packet).** The
   `C` packet's DATA behaves as if typed on the calculator's command
   line and confirmed with Enter. A bare command name (`HOME`, `UPDIR`)
   runs itself and must NOT get `EVAL` appended (a real bug: `goHome()`
   used to send `"HOME EVAL"`, which left nothing on the stack for the
   trailing `EVAL` to consume, and errored). A literal like `{ DEV }` or
   `'name'` is pushed as data and needs a trailing `EVAL`/verb
   (`{ DEV } EVAL`, `'name' PURGE`, `'name' CRDIR`).
7. **Language:** the user (Jan) communicates in Czech; keep replies and
   `architektura.md` in Czech. UI copy, code comments, and `README.md`
   are English (translated deliberately in an earlier session — don't
   revert this).

## Protocol quick reference

(See `architektura.md` for the full derivation — this is just a cheat
sheet.)

- Packet types in use: `S`/`I` (Send-Init), `F` (File-Header), `D` (Data),
  `Z` (EOF), `Y` (ACK), `N` (NAK), `B` (Break), `E` (Error), `R`
  (Get-request), `G` (Generic — used for `REMOTE DIRECTORY`), `C`
  (Command — non-standard, used for `REMOTE HOST`, i.e. navigation,
  create/delete), `X` (unlabeled virtual-file header, appears when the
  calculator negotiates CHKT=3 on an inner exchange).
- A single "command" packet (R/G/C) sent right after Send-Init reuses
  Send-Init's own seq number (does NOT auto-increment) — this is an
  exemption specific to that one packet, discovered the hard way. A
  genuine file transfer (real PUT) increments seq normally:
  S→F→D→D…→Z→B.
- This app's own `buildSendInitData()` always proposes CHKT='1' (1-byte
  checksum), so this app's own transfers never need CRC-16. The
  calculator's inner Send-Init (during the `C`-packet "virtual file"
  response) may itself offer CHKT=3 — the app detects and logs this but
  doesn't implement CRC-16 validation for it.
- `REMOTE CD` and other generic verbs are rejected by the calculator
  ("Invalid Server Cmd."); only `REMOTE HOST <rpl-text>` works for
  navigation/create/delete, via the `C` packet.
- The `C` packet's response is always just a snapshot of whatever's on
  the calculator's stack at that moment — not a "result" of the command.
  When it starts with `"Error:"`, the command failed at the RPL level
  even though the Kermit transfer itself completed cleanly (no `E`
  packet, no NAK) — `sendHostCommand()` detects this string and surfaces
  it as an error.

## Things intentionally not done yet

- No license chosen for the repo.
- No automated tests (no simulator for the calculator side exists).
- Several features are implemented but not yet hardware-verified by the
  user through the app itself — check `architektura.md`'s latest entries
  for "**Zatím netestováno na hardwaru**" markers before assuming
  something works.
