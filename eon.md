# Eon Platform — technical dossier

Co-created by JuanKLimoN & Claude. Contact: juan@eonplatform.eu.org
Puerto Santa Cruz, Santa Cruz, Argentina.

This is the long-form version of the website, for anyone (human or AI) who wants
the details. Every figure comes from the projects' own engineering logs or git
history. Items the logs mark as *not yet verified* are labelled that way here too.
Security findings that are still open, and private network details, are left out on purpose.

## 1. What Eon is

AI customer service on WhatsApp for small businesses in Latin America: clubs,
hotels, tradespeople. The bots answer, sign people up, collect payments
(Mercado Pago link or bank transfer with the receipt read by AI) and keep records,
in Rioplatense Spanish.

Products:
- **Sonriar**: the company and product. Sales bot (mascot: "el Guardián") plus a
  central panel with customers, licences, live status of each installation and
  suspension for non-payment.
- **Sedes**: what Sonriar delivers to each club. Member sign-up over WhatsApp,
  payment by Mercado Pago or transfer (Claude reads the receipt), digital
  membership card with QR, door check by scanning it, fee reminders, audit, backups.
  In use at a real sports club (not named; no figures published).
- **Multiservicios Santa Cruz**: province-wide directory of trades over WhatsApp.
  People describe the problem; the bot detects trade and town and returns providers.
- **Hotel bot**: guest assistant with reservations, OCR on receipts, Whisper for
  voice notes, payment links, FAQ/promotions/gallery managed from a panel.
- **Multi-tenant platform** (TypeScript, in development): isolated organizations,
  RBAC (SUPERADMIN / ADMIN / OPERATOR), multi-session WhatsApp, CRM, audit log
  with CSV export, Socket.io real time, per-organization prompt, JWT with refresh
  tokens, AES-256-GCM for stored WhatsApp credentials.

## 2. How Claude is used

**In the product.** One shared module (`comun/ia_claude.js`), plain `fetch`, no SDK,
used by the hotel bot, Sedes, Sonriar and Multiservicios:
- converts the bots' OpenAI-style messages to the Anthropic Messages format
  (system prompts merged; base64 images become image blocks, so Claude reads
  transfer receipts and ID photos);
- treats `stop_reason: refusal` and empty text as failures;
- gives Claude a deadline (configurable, 8 s by default);
- on failure or timeout, falls through to a parallel race across several
  NVIDIA-hosted models (first good answer wins); at 15 s a safe canned reply goes
  out instead of silence;
- with no Anthropic key, every bot runs exactly as before on the backup path.

**In development.** Everything is written with Claude Code: backend, panels,
bots, the C# desktop apps, the Android app, this website. Work runs in Claude Code
cloud sessions and lands as pull requests. Claude Code also runs full-repository
security reviews, with every finding ranked and tracked as fixed or pending.

## 3. Case: memory corruption in DS4Windows 3.9.9 (EonPS4Controller)

- Symptom: `AccessViolationException` (0xc0000005) in `coreclr.dll`, in unrelated places.
- Isolation (editor open 45 s, controller connected): 4/8 crashes; service stopped
  0/8; GC nearly idle 0/8. Needs live I/O *and* a GC pass.
- Cause: overlapped `ReadFile` on a managed `byte[]` pinned only during the call;
  the GC could move it while the kernel still wrote to the old address; on a
  timed-out wait the read stayed pending without `CancelIoEx`. Same pattern on
  the output path.
- Fix: pin for the whole operation (`fixed`) and `CancelIoEx` when the wait ends early.
- Result: 7/16 crashes without the fix vs 0/12 with it (Fisher's exact p ≈ 0.01).
  Repeatable stress script included.

## 4. Case: Bluetooth audio into the PS4 controller (EonPS4Controller)

- Hand-written SBC codec in C# (32 kHz encoder; 16 kHz mono decoder for the
  controller mic), Bluetooth CRC-32, raw `0x11`/`0x17` reports, 16 ms engine loop.
- WASAPI loopback and per-process capture through hand-rolled COM interop, no
  NuGet packages; per-app routing (the game to the controller, the rest on the PC);
  controller mic exposed to Windows through a virtual cable with adaptive buffering.
- Problem: music-reactive lights made audio stutter. Measured with a write-latency
  histogram: slow writes (>8 ms) went from ~3 % to 33 %. A lock and an A/B on a
  volume flag were ruled out. Cause: Bluetooth sharing radio time with 2.4 GHz
  Wi-Fi, so the link becomes fragile under load. Curve on a fragile link: cliff at
  ~8 light reports/s; a healthy link handled 40/s.
- Fix: adaptive limit. Every ~1.5 s, slow-write fraction > 15 % steps the light/rumble
  budget down (unlimited → 12 → 6 → 3 per second); < 6 % for 4 windows steps up.
  Token bucket, burst 3, newest state wins. Light and rumble share report `0x11`.
- Result under load: slow writes 33 % → 4–9 %; resyncs per phase 3 → 0; audio
  steady at 62–63 reports/s. Not yet verified: how it sounds with real music.
- Also: audio-reactive lights (beat detection, spectrum, palettes), rumble driven by
  sound mixed with the game's own rumble, continuous gyro auto-calibration, and
  controller alerts. The test project compiles the same .cs files the app ships;
  codec vectors come from the original Python reference.

## 5. Other projects

- **EonTone** (C#): multitrack looper played with the controller. Karplus-Strong
  strings with Jaffe-Smith tuning (< 0.1 cent), FM electric piano, PolyBLEP saws
  through a state-variable filter, modal marimba, small Freeverb; Media Foundation
  decoding over COM; reads the controller through EonPS4Controller's local DSU/UDP
  server so it never opens the device.
- **LAYA** (Python): offline Spanish voice assistant. Voice detection → local
  Canary speech-to-text → decision layer → actions; fallback chain: learned
  commands → local model → Claude (opt-in). It offers to save what Claude solved
  as a new command.
- **EonCT** (Node + PowerShell): a tablet controls the PC from the browser
  (gyro air-mouse, touchpad, keyboard, mic, PC audio, file transfer). PCs announce
  themselves over UDP every 3 s; a native Android app (Java, built without Gradle:
  aapt2 → javac → d8 → zipalign → apksigner) discovers them, because a web page
  cannot listen to UDP.
- **eonSC**: when the main Wi-Fi drops, the PC shares its backup connection as its
  own hotspot, with a background watchdog.
- **Brisa**: high-contrast game that teaches young children to use a mouse.

## 6. Origin

The founder started on a Commodore 64 at age five in the countryside of Puerto
Santa Cruz, where mobile 4G arrived only a few years ago. The first Eon bot ran on
an Android tablet inside Termux, with whisper.cpp compiled on the device. Git
history of eon-bot: first commit 2026-07-19, 233 commits by 2026-10-09. The desktop
projects lived outside git until recently, so their history is not counted.

## 7. Languages and tools

C#, .NET 8, WPF/XAML, Win32, COM interop · JavaScript, Node.js, TypeScript, React 18,
Tailwind, Vite, Express, Socket.io, Prisma, SQLite, PostgreSQL, Vitest · Python ·
Java, Android SDK · PowerShell, Bash, Batch · Baileys · Claude API, Claude Code ·
NVIDIA-hosted models · whisper.cpp, Whisper, Tesseract OCR, Canary · WASAPI, Media
Foundation · HidHide, ViGEmBus · Termux.

## 8. Roadmap

Keep the club deployment solid and sell Sedes through Sonriar; bring Multiservicios
and the hotel bot to real users; publish screenshots, a demo video and a measured
evaluation of receipt reading; migrate the standalone bots onto the multi-tenant core.
