# Baseband Access Project — Notes

> Living notes for the baseband/diag-mode toolchain project.
> Goal: build a safe, licencing-clean set of **MCP servers** that let an AI
> agent drive Qualcomm cellular devices through their diagnostic interfaces
> (EDL/Sahara, Firehose, DIAG/QCDM, AT), plus an off-grid radio leg
> (Meshtastic).

## ⚠️ ALL SOFTWARE HERE IS ALPHA — UNTESTED ON REAL HARDWARE

Every MCP server in this project is **experimental, unverified, and has not
been validated against real hardware yet**. Do not plug anything valuable into
it expecting it to work. Treat all tools as reference implementations for
learning. Flashing the wrong data or losing power mid-write can **brick a
device**. When we finally do a live run it will be on owned, disposable lab
hardware only.

## Status

| Piece              | Status                    | Where                                          |
| ------------------ | ------------------------- | ---------------------------------------------- |
| `sahara-mcp`       | alpha · uploaded          | https://github.com/libertyrights/sahara-mcp    |
| `firehose-mcp`     | alpha · in progress       | —                                              |
| `qcdm-mcp` (Diag)  | alpha · planned           | —                                              |
| `at-mcp` (modem)   | alpha · planned           | —                                              |
| Meshtastic MCP     | alpha · install (official) via uvx | https://github.com/meshtastic/meshtastic-mcp |
| ADB / forensics    | already present           | MCP `android-adb`, `android-forensics`         |

## Goal statement

- Learn and exercise the full Qualcomm device diagnostic stack from EDL on up,
  safely, on owned lab hardware.
- Expose each protocol as an MCP server so agent workflows can: enumerate
  devices, handshake, identify the SoC, load a Firehose programmer, flash /
  read partitions, talk to the modem (Diag NV/EFS, AT), and capture radios.
- Licence-clean end to end: original code MIT; reference BSD-3-Clause
  (`linux-msm/qdl`) and credit it; **no GPL code imported** (e.g.
  `bkerler/edl` is GPL-3.0 — consult for behavior only, never copy).

## Protocol stack (down → up)

1. **EDL mode** — bootloader download mode. Enter via edl cable (D+ shorted to
   GND) + `Vol-Up`+`Power` ~20 s, or battery pull tricks; newer UFS devices
   may need shorting the UFS `CLK` test point. Presents USB VID `05C6`.
2. **Sahara** — PBL downloads a Firehose programmer (`prog_*.elf`) signed by
   the right tool bianry. Always present in ROM. Handles HELLO, command mode
   (chip id, serial, PK hash) and image transfer. → `sahara-mcp`
3. **Firehose** — XML streaming protocol the programmer speaks after load:
   `program`, `read`, `erase`, `patch`, `configure`, GPT access. Framing is a
   fastboot-like packet protocol over the same USB bulk endpoints. →
   `firehose-mcp`
4. **DIAG / QCDM** — serial diagnostic protocol (usually `ttyUSB`/com port;
   exposed on rooted phones as `/dev/diag` or via USB modem). NV items, EFS2
   filesystem, QCN backups, memory read/write, radio-layer capture. →
   `qcdm-mcp`
5. **AT commands** — OS-independent modem control (IMEI, modem state, network,
   data). Works on any serial modem. → `at-mcp`
6. **Meshtastic** — LoRa off-grid mesh for the radio leg (emergency comms
   companion to the cellular work). Official MCP server exists.

## Machine setup (this laptop)

- OS: Windows; shell: PowerShell 5.1. NOT a git repo at the workspace root —
  each MCP project repo is separate.
- Python 3.14.7 available as `python`.
- git 2.55 + gh 2.98 installed; gh authed as `libertyrights`
  (`gh` git ops via HTTPS). Commit identity:
  `libertyrights <libertyrightsassn@gmail.com>`.
- USB plan: Zadig/WinUSB for EDL interfaces so pyusb/libusb can claim them.

## Libraries / references

- `linux-msm/qdl` — `src/sahara.c`, `src/firehose.c` **BSD-3-Clause** — primary
  protocol reference (used for `sahara-mcp`).
- `bkerler/edl` — **GPL-3.0** — behavioral reference only; do not import.
- `P1sec/QCSuper` — Diag/QCDM protocol docs + capture tool (reference for
  `qcdm-mcp`); not imported.
- `grmrgecko/qcdm-efs2` — single-file EFS2-over-DIAG client (reference).
- `strongtz/edl-ng` — MIT, .NET Sahara/Firehose (cross-check framing).
- `meshtastic/meshtastic-mcp` — official Meshtastic MCP server.

## Next steps

1. Finish `firehose-mcp` (fetch `qdl/src/firehose.c` for framing facts).
2. Write `qcdm-mcp` and `at-mcp`.
3. Push each to GitHub under `libertyrights`, MIT + credits.
4. Wire Meshtastic MCP into the agent config.
5. First on-hardware run: EDL → Sahara handshake → chip info → Firehose GPT
   print. Verify only on owned lab hardware.