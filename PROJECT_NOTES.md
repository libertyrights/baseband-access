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
| `sahara-mcp`       | alpha · uploaded · GPL-3.0| https://github.com/libertyrights/sahara-mcp    |
| `firehose-mcp`     | alpha · uploaded · GPL-3.0| https://github.com/libertyrights/firehose-mcp  |
| `qcdm-mcp` (Diag)  | alpha · uploaded · GPL-3.0| https://github.com/libertyrights/qcdm-mcp      |
| `at-mcp` (modem)   | alpha · uploaded · GPL-3.0| https://github.com/libertyrights/at-mcp        |
| `edl-mcp` (façade) | alpha · uploaded · GPL-3.0| https://github.com/libertyrights/edl-mcp       |
| `fastboot-mcp`     | alpha · uploaded · GPL-3.0| https://github.com/libertyrights/fastboot-mcp  |
| `qcn-mcp` (QCN/NV) | alpha · uploaded · GPL-3.0| https://github.com/libertyrights/qcn-mcp       |
| Meshtastic MCP     | installed · wired (uv tool, pinned `mcp<2`) | https://github.com/meshtastic/meshtastic-mcp |
| ADB / forensics    | already present           | MCP `android-adb`, `android-forensics`         |

All seven of our MCP servers ship a `tests/` pytest harness (mocked USB/serial,
no hardware needed) — currently **passing** locally. They are still alpha and
untested against real devices.

**Licence:** all seven MCP servers are **GPL-3.0** (copyleft — derivatives must
stay open). Protocol facts reference `linux-msm/qdl` (BSD-3-Clause, compatible
with GPL). `bkerler/edl` (GPL-3.0) consulted for behavior only; not copied.

## Safety model (uniform across servers)

Every tool that writes to a device takes `dry_run: bool = True` and
`confirm: bool = False`. Unless **both** `dry_run=False` and `confirm=True` are
passed, the tool performs **nothing** (no USB/serial port is even opened) and
returns a payload describing exactly what it *would* do:

```json
{"dry_run": true, "executed": false, "would_do": {...},
 "message": "DRY RUN — nothing written; set dry_run=False and confirm=True to proceed."}
```

Read-only tools (`*_list_devices`, `*_chipinfo`, `diag_nv_read`, `at_signal`,
`qcn_info`, `edl_parse_gpt_file`, …) are unguarded. The **façade `edl-mcp`**
always resolves the SoC via its chipset DB before suggesting a Firehose loader,
and its `edl_backup_partitions` is read-only on the device (backup-first).

## Goal statement

- Learn and exercise the full Qualcomm device diagnostic stack from EDL on up,
  safely, on owned lab hardware.
- Expose each protocol as an MCP server so agent workflows can: enumerate
  devices, handshake, identify the SoC, load a Firehose programmer, flash /
  read partitions, talk to the modem (Diag NV/EFS, AT), and capture radios.
- Licence-clean end to end: all our code is **GPL-3.0** (copyleft, so
  derivatives must stay open source too); protocol facts reference BSD-3-Clause
  (`linux-msm/qdl`) and MIT (`grmrgecko/qcdm-efs2`) work, which both license
  freely into GPL projects. No GPL *code* is copied without the project
  already being GPL.

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

Convenience layers built on the above:

- **`edl-mcp`** — high-level façade over steps 2–3: one call chains Sahara →
  Firehose, auto-picks a loader hint from the **chipset DB**
  (`msm_id`/`oem_id`/`model_id` → SoC name), and parses GPT locally. Ships a
  user-overridable DB at `~/.config/edl-mcp/chipsets.json`. Chipset names are
  best-effort and marked uncertain until verified.
- **`fastboot-mcp`** — bootloader-stage ops (getvar, flash, erase, `oem` /
  `flashing unlock`, reboot) via the official `fastboot` CLI.
- **`qcn-mcp`** — offline QCN/NV backup tooling (parse, list, extract IMEI,
  diff, patch with dry-run). Nothing here touches a device.

## Machine setup (this laptop)

- OS: Windows; shell: PowerShell 5.1. NOT a git repo at the workspace root —
  each MCP project repo is separate.
- Python 3.14.7 available as `python`.
- git 2.55 + gh 2.98 installed; gh authed as `libertyrights`
  (`gh` git ops via HTTPS). Commit identity:
  `libertyrights <libertyrightsassn@gmail.com>`.
- USB plan: Zadig/WinUSB for EDL interfaces so pyusb/libusb can claim them.
- `uv` 0.13.0 installed via `pip install uv`. Meshtastic MCP installed as a uv
  tool: `uv tool install meshtastic-mcp --with "mcp<2"` → shims in
  `C:\Users\mark\.local\bin\` (the `--with "mcp<2"` pin works around an upstream
  break: `mcp` 2.x renamed `FastMCP`→`MCPServer` and `meshtastic-mcp` imports the
  old path). Run its CLI with `PYTHONUTF8=1` to avoid a cp1252 console crash.
- `git`/`gh` are installed but **not on the default shell PATH** — they live at
  `C:\Program Files\Git\cmd\git.exe` and `C:\Program Files\GitHub CLI\gh.exe`.
  Prefix `PATH` (or use full paths). Credential helper is
  `!gh auth git-credential` and pushes need `GIT_TERMINAL_PROMPT=0`.
- `pytest` installed; each repo has a `tests/` harness that runs with no devices.
- MCP client config: `C:\Users\mark\.config\opencode\opencode.jsonc` (registers
  `sahara-mcp`, `firehose-mcp`, `qcdm-mcp`, `at-mcp`, `edl-mcp`, `fastboot-mcp`,
  `qcn-mcp`, `meshtastic`). Restart the client to pick up new servers.

## Libraries / references

- `linux-msm/qdl` — `src/sahara.c`, `src/firehose.c` **BSD-3-Clause** — primary
  protocol reference (used for `sahara-mcp`).
- `bkerler/edl` — **GPL-3.0** — behavioral reference only; do not import.
- `P1sec/QCSuper` — Diag/QCDM protocol docs + capture tool (reference for
  `qcdm-mcp`); not imported.
- `grmrgecko/qcdm-efs2` — single-file EFS2-over-DIAG client (reference).
- `strongtz/edl-ng` — MIT, .NET Sahara/Firehose (cross-check framing).
- `meshtastic/meshtastic-mcp` — official Meshtastic MCP server (installed).
- `JohnBel/EfsTools` — QCN/NV item id definitions (reference for `qcn-mcp`).
- QCN NV item ids (IMEI 550, MEID 1943, ESN 0, …) cross-checked against ORCT /
  `dl12345` / `fipso` write-ups — facts only.

## Next steps

1. ~~Finish `firehose-mcp`~~ — done.
2. ~~Write `qcdm-mcp` and `at-mcp`~~ — done; plus `edl-mcp`, `fastboot-mcp`,
   `qcn-mcp`.
3. ~~Push each to GitHub under `libertyrights`~~ — done, GPL-3.0.
4. ~~Wire Meshtastic MCP into the agent config~~ — done (pinned `mcp<2`).
5. **Still open:** first on-hardware run on owned lab hardware only — EDL →
   Sahara handshake → chip info → loader → Firehose GPT print → backup GPT →
   only then any write. Use the `edl-mcp` façade with `dry_run`/`confirm`.
6. **Still open:** verify the chipset DB entries against real devices and grow
   the `~/.config/edl-mcp/chipsets.json` override file.