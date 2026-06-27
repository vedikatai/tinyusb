# TinyUSB 1.0 Roadmap Audit

**Generated:** 2026-06-27 06:43 UTC  
**Tree commit:** `0a25cc27d7d3699536f6df01e80af3eb0423ce58` (`0a25cc27d7d3`) on branch `tinyusb-1.0-roadmap`  
**Upstream:** [hathach/tinyusb](https://github.com/hathach/tinyusb) `master`  
**Public fork target:** [vedikatai/tinyusb](https://github.com/vedikatai/tinyusb) only (never push to hathach)  
**Toolchain:** Arm GNU Toolchain 14.2.Rel1 (`arm-none-eabi-gcc 14.2.1 20241119`) + CMake/Ninja on macOS arm64

---

## Executive summary (one screen)

TinyUSB HEAD (`0a25cc27d7d3`) is a mature **USB 2.0 FS/HS device + host** stack with ~34k LOC portable DCD/HCD, ~16.5k LOC class drivers, and active maintenance on core paths (device, host, common, dwc2, stm32_fsdev, rp2040, wch). **1.0 should ship as a stability + compliance + footprint freeze of USB 2.0**, not as SuperSpeed/USB4.

| Decision | Recommendation |
|---|---|
| **Ship in 1.0** | Device core (`usbd`), host core (`usbh`+hub single-tier proven), CDC/HID/MSC/MIDI1/DFU/Vendor, major DCD ports (dwc2, stm32_fsdev, rp2040, nrf5x, musb, rusb2, chipidea, EHCI/OHCI where HIL exists) |
| **Deprecate / optional in 1.1** | Low-activity class niches: **BTH**, **Printer**, **USBTMC** (device-only; host missing #2714), **Sony CXD56** portable, **template** portable, possibly **MTP** until HIL green |
| **Post-1.0 foundation (USB 3.2 / USB4)** | New speed tier in `tusb_types` / DCD API (not bolted onto FS/HS only structs); Type-C / PD stays in `src/typec` (254 C lines, 3 contrib/3y) as optional; SuperSpeed needs new portable IP (xHCI-like / vendor SS controllers) — **out of 1.0 scope** |
| **Stability investment before 1.0** | STM32 (fsdev+dwc2), RP2040/RP2350, ESP32/dwc2 ISO, CH32/WCH, nRF — highest issue heat last 12 months |
| **Blockers this audit** | Full example×all-boards matrix not complete (env deps); **USBCV not installed**; **cppcheck/scan-build not installed**; **ceedling not on PATH**; tag 0.16–0.18 CMake differs (footprint via Make in progress); **vedikatai/tinyusb was empty and was created as a new repo (not a GitHub fork of hathach)** — push of this branch establishes history |

**Effort to 1.0 stable (estimate):** **22–34 developer-weeks** (see Phase effort table). Highest leverage: compliance + host hub depth + dwc2 ISO stall + RP2040 host size (#3688) + HIL gate.

---

## Phase 1 — Stack state of the union

### 1.1 Subsystem metrics (recursive where noted)

Measured with `find` + `wc -l` and `git log --since='3 years ago'` on `0a25cc27d7d3`.

| Subsystem path | C LOC | H LOC | Asm | Last commit (path) | Contrib 3y | Notes |
|---|---:|---:|---:|---|---:|---|
| `src/portable` (all) | 34205 | 14572 | 0 | 2026-06-25 | 74 | Largest surface; IP-specific |
| `src/class` (all) | 16583 | 11750 | 0 | 2026-06-24 | 64 | All class drivers |
| `src/host` | 2747 | 966 | 0 | 2026-06-22 | 14 | Younger than device |
| `src/device` | 1756 | 1488 | 0 | 2026-06-22 | 22 | Core usbd |
| `src/common` | 834 | 2909 | 0 | 2026-06-24 | 42 | Types, FIFO, utils |
| `src/osal` | 0 | 1853 | 0 | 2026-06-22 | 10 | Headers only |
| `src/typec` | 254 | 486 | 0 | 2026-06-24 | 3 | PD/Type-C starter |
| `src/tusb.c` + headers | 570+193+887 | — | 0 | (root) | — | Init / options |

#### Class drivers

| Class | C | H | Last | Contrib3y | Open issues (keyword search, approx) | MCU coverage |
|---|---:|---:|---|---:|---:|---|
| audio | 1905 | 1759 | 2026-06-18 | 10 | 24 open mention audio/uac | High (ISO-capable ports) |
| bth | 298 | 117 | 2025-10-31 | 5 | low | Low |
| cdc | 3084 | 1351 | 2026-06-06 | 11 | part of 109 cdc/hid/msc open | Near-universal |
| dfu | 584 | 273 | 2026-03-11 | 2 | low | Medium |
| hid | 1237 | 2939 | 2026-06-15 | 19 | part of 109 | Near-universal |
| midi | 2632 | 870 | 2026-05-19 | 5 | active (MIDI2 host PRs) | Medium |
| msc | 1486 | 709 | 2026-06-24 | 8 | part of 109 | High |
| mtp | 574 | 1081 | 2026-06-03 | 8 | low | Low–medium |
| net | 1540 | 309 | 2026-06-16 | 9 | NCM iOS issues | Medium |
| printer | 328 | 213 | 2026-05-04 | 2 | low | Low |
| usbtmc | 866 | 460 | 2025-10-31 | 5 | #2714 host missing | Low |
| vendor | 522 | 278 | 2026-03-12 | 7 | medium | High |
| video | 1527 | 816 | 2026-06-19 | 9 | medium | Medium (ISO) |

#### Portable IP (selected; full list in audit data)

| Portable | C | H | Last | Contrib3y |
|---|---:|---:|---|---:|
| synopsys/dwc2 | 3141 | 3815 | 2026-06-11 | 24 |
| renesas/rusb2 | 1931 | 2045 | 2026-04-17 | 8 |
| mentor/musb | 1913 | 1020 | 2026-06-16 | 7 |
| st/stm32_fsdev | 1834 | 1024 | 2026-06-24 | 14 |
| raspberrypi/rp2040 | 1770 | 185 | 2026-05-15 | 9 |
| wch (tree) | 1730 | 685 | 2026-06-22 | 13 |
| dialog/da146xx | 1241 | 0 | 2026-04-17 | 4 |
| sunxi | 1227 | 643 | 2026-02-28 | 4 |
| nxp/khci | 1224 | 0 | 2025-11-19 | 6 |
| microchip/samd | 1198 | 0 | 2025-11-19 | 5 |
| analog/max3421 | 1107 | 63 | 2025-05-21 | 3 |
| nordic/nrf5x | 1096 | 0 | 2026-05-08 | 7 |
| ehci | 1040 | 509 | 2026-03-05 | 5 |
| ohci | 714 | 397 | 2025-12-13 | 4 |
| valentyusb/eptri | 671 | 39 | 2025-11-19 | 3 |
| sony/cxd56 | 455 | 0 | 2025-10-22 | 5 |
| template | 295 | 0 | 2025-10-22 | 4 |

**Open issues (upstream):** 199 open issues total (`github__search_issues` `repo:hathach/tinyusb is:issue is:open`, 2026-06-27). Label `Bug 🐞` exact search returned 0 (emoji label encoding); keyword `label:"Bug 🐞"` created last 12 months → **73** issues (includes closed in search window — use for heat only).

### 1.2 Subsystem × MCU family matrix (support derived from `hw/bsp/*/family.cmake` + `src/portable`)

Legend: **S** = portable present (supported in tree), **U** = family exists but weak/untested evidence in this audit, **—** = unsupported.

Condensed by **IP block** (not every STM32 letter):

| Subsystem / IP | STM32 FSDEV | STM32/ESP DWC2 | RP2040 | nRF5x | SAMD | Chipidea HS | MUSB | RUSB2 | WCH | EHCI/OHCI | Valenty | CXD56 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Device core | S | S | S | S | S | S | S | S | S | S | S | S |
| Host core | U | S | S | —/U | U | S | S | S | S | S | — | — |
| CDC/HID/MSC | S | S | S | S | S | S | S | S | S | S | S | U |
| Audio/Video ISO | S* | S* | S* | S* | U | S* | U | U | S* | U | — | — |
| Net (NCM/ECM) | S | S | S | S | U | U | U | U | U | U | — | — |
| Type-C | ST typec only | — | — | — | — | — | — | — | — | — | — | — |

\* ISO quality varies; known regressions on CH32 after xfer_isr move (#3425 closed), dwc2 ISO activate stall (#3689 open).

**BSP families counted:** 80+ under `hw/bsp/` (stm32*, at32*, ch32*, lpc*, sam*, nrf, rp2040, espressif, maxim, ra, rx, …). Board counts leaders: espressif 15, samd2x_l2x 14, nrf 12, imxrt 11, stm32h7 10, stm32f4 10, rp2040 8.

### 1.3 Five lowest-activity / lowest-coverage subsystems (1.1 deprecation candidates)

1. **`src/class/bth`** — 298 C, last 2025-10-31, 5 contrib/3y; niche Bluetooth HCI transport.  
2. **`src/class/printer`** — 328 C, 2 contrib/3y; Printer Class 1.1 only (`src/class/printer/printer.h:44`).  
3. **`src/portable/sony/cxd56`** — 455 C, last 2025-10-22; single SoC.  
4. **`src/portable/template`** — 295 C scaffolding.  
5. **`src/class/usbtmc`** — last 2025-10-31; device-only; host request #2714 still open → either complete host or mark device-only experimental.

Honorable mentions: **analog/max3421** (last 2025-05-21), **typec** (3 contrib — keep as post-1.0 seed, not 1.0 gate).

### 1.4 Five MCU families with most issue heat (last 12 months, keyword search)

| Rank | Family / IP | Issues created >2025-06-27 (search) | Evidence |
|---:|---|---:|---|
| 1 | STM32 / fsdev / dwc2 | 26 (`stm32 OR dwc2 OR fsdev`) | e.g. #3696 G4 IRQ, #3689 ESP32-S2 dwc2 ISO |
| 2 | RP2040 / RP2350 | 19 | e.g. #3188 MSC panic, #3688 host code size |
| 3 | ESP32 / Espressif | 18 | dwc2 shared with STM32 HS; NCM/iOS |
| 4 | CH32 / WCH | 3 (+ 8 open PRs riscv/ch32) | #3425 audio ISO; active port PRs 3682, 3642, … |
| 5 | NXP (imxrt/lpc/kinetis) | 4 | #3243 hub removal on imxrt |

**nRF** showed only 1 hit in keyword search but is tier-1 for BLE+USB products — invest via HIL not issue count alone.

### 1.5 USB-IF / usb.org spec revisions vs `src/` comments

Fetched from USB-IF document library / web summary (https://www.usb.org/documents, https://www.usb.org/document-library/…); **HTML scrape of documents index did not yield machine-readable revision table in-session — revisions below are from USB-IF library page titles as of audit date:**

| Spec | Latest revision (USB-IF) | TinyUSB references | Staleness |
|---|---|---|---|
| **USB 2.0** | Still **Rev 2.0** (2000) + consolidated ECN package **2025-06-03** (`usb_20_20250603.zip`) | `usbd.c:840` §9.3.1; `tusb_types.h:268` Table 9-7; `tusb_option.h:613` §7.1.20; hub Table 11-16 | Core chapter refs valid; **ECNs (double ISO IN, eUSB2 repeater 2024–2025) not obviously implemented** — track for 1.1+ |
| **USB 3.2** | **Revision 1.1 (June 2022)** | Essentially **absent** in `src/` (no SuperSpeed stack) | Expected; post-1.0 |
| **USB4** | **USB4 Spec Version 2.0** (library page ~2026-04-02; Nov 2025 zip) | Absent | Post-1.0 / research |

Class-level pins in tree:

| Class | Implemented (from headers/comments) | Current industry | 1.0 policy |
|---|---|---|---|
| HID | **1.11** (`hid.h` §6.2.2.x) | Still 1.11 | **Pin 1.11** |
| CDC ACM | PSTN **1.2** (`cdc_device.c:417`) | 1.2 | Pin 1.2 |
| MSC | BOT/SCSI (classic) | Same | Pin BOT |
| Audio | **UAC2** primary + UAC1 paths (`audio_device.c`) | UAC3 exists | **1.0 = UAC2**; UAC3 post-1.0 |
| Video | UVC descriptors (`video.h` 3.9.x) | UVC 1.5 | Document 1.5 target; verify bcdUVC |
| MIDI | **1.0 + 2.0 UMP** (`midi2_device.c` USB-MIDI 2.0 §3.2.2) | MIDI 2.0 | Ship both; coexistence host = open design (#3740) |
| DFU | 1.1 typical | 1.1 | Pin 1.1 |
| Printer | **1.1** (`printer.h:44`) | 1.1 | Optional/deprecate |
| Net | ECM/NCM/RNDIS | NCM preferred | Prefer NCM; RNDIS legacy |
| BTH | Bluetooth Core **5.3** cite (`bth_device.h:45`) | 5.4+ | Optional |

---

## Phase 2 — Build matrix audit

### Scope actually executed

- **Boards attempted:** `stm32f407disco`, `stm32f411blackpill`, `stm32f103_bluepill`, `feather_m0_express`, `nrf52840dk`, `stm32f439nucleo`  
- **Examples:** 16 device + 4 host priority set (not every example × every board — full Cartesian would be 30×200+ cells; blocked by missing SDKs/deps for most families)  
- **Output file:** `/tmp/tinyusb-build-matrix.json`  
- **Commit recorded in JSON:** see file `commit` field  

### Results summary (partial run)

| Metric | Value |
|---|---|
| Cells attempted | 120 (live file may still be updating) |
| Success (stm32f4 + nrf device) | Strong — nearly all priority device examples on f407/f411/f439/nrf52840dk |
| `stm32f103_bluepill` | compile_fail cluster (missing `get_deps` stm32f1 / SDK — **missing_submodule**, not core bug) |
| `feather_m0_express` | compile_fail cluster (samd deps not fetched — **missing_submodule**) |
| Host on `nrf52840dk` | compile_fail (nRF port is device-oriented; host unsupported or misconfigured — **unsupported/BSP**) |
| `device/cdc_dual_ports` @ `stm32f407disco` | compile_fail (investigate; worked on f411 — possible HS vs FS config) |

### Failure classification (policy)

| Class | Meaning | Action for 1.0 |
|---|---|---|
| missing_toolchain | No arm-none-eabi / riscv gcc | CI image gap |
| missing_submodule | `hw/mcu` / Pico SDK / ESP-IDF | `tools/get_deps.py` + docs |
| compile_fail | Real C error | Fix before 1.0 |
| link_fail | Undefined ref | Often class/OS mismatch |
| linker_script_issue | Region overflow | Board memory map |
| bsp_misconfiguration | Wrong OPT_MCU / IRQ | Port fix |

### 0.18.0 comparison

CMake on tags **0.16.0–0.18.0 failed** with current scripts (API drift). **Make-based rebuild in progress** → `/tmp/tinyusb-footprint-compare.json`. HEAD MinSizeRel sizes (cmake, measured):

| Example | Board | ELF bytes | text | static RAM (data+bss) | Source |
|---|---|---:|---:|---:|---|
| device/cdc_msc | stm32f411blackpill | 354636 | 18340 | 11356 | footprint job HEAD |
| device/hid_composite | stm32f407disco | 331832 | 15560 | 21488 | HEAD |
| device/msc_dual_lun | stm32f407disco | 333796 | 16272 | 38384 | HEAD |
| device/cdc_msc | stm32f407disco | 360620 | 18744 | 30392 | HEAD |

**Requested boards not fully satisfied this session:**

| Requested | Status |
|---|---|
| hid_composite @ **rp2040** | **Blocked:** Pico SDK not present (`PICO_SDK_PATH` unset) — use stm32f407disco proxy sizes above |
| msc_dual_lun @ **samd21** | **Blocked:** `feather_m0_express` cmake fail without Microchip CMSIS pack |
| net_lwip_webserver @ **stm32f429disco** | Board not in tree as that exact name; closest built: **stm32f439nucleo** / **stm32f407disco** net_lwip_webserver (elf ~652088 on f407) |

---

## Phase 3 — USB compliance audit

### 3.1 Control / enumeration timing (static review of `src/device/usbd.c`)

| # | File:line | Spec | Current behavior | Compliant behavior | Risk if non-compliant |
|---|---|---|---|---|---|
| C1 | `usbd.c:969-975` SET_ADDRESS delegates entirely to `dcd_set_address`; stack sets `_usbd_dev.addressed=1` **before** status may complete | USB 2.0 **§9.4.6** / **§9.2.6.3**: device must not use new address until **after** Status stage completes; Status must complete within **2 ms** (response timing budget is host-driven; device must accept SETUP) | Ports differ: **stm32_fsdev** correctly latches address in `dcd_edpt0_status_complete` (`dcd_stm32_fsdev.c:477-484`); **dwc2** sets DCFG address **then** queues ZLP (`dcd_dwc2.c:522-527`) — address may be live **before** status ACK on wire | Prefer fsdev pattern universally; document DCD contract | Intermittent SET_ADDRESS failures on some PHYs / analyzers; USBCV chapter 9 failures |
| C2 | `usbd.c:840-843`, `889-890` Status stage EP selection cites **§9.3.1** | Correct for wLength==0 → Status IN | Keep | Low if DCDs honor |
| C3 | `usbd.c:780-795` Suspend/Resume skips callbacks if not connected; remote wakeup flag passed to `tud_suspend_cb` | USB 2.0 **§7.1.7.6** / **§9.4.5** remote wakeup | Logic sound; depends on DCD detecting 3 ms idle | False suspend storms if DCD noisy (comment acknowledges) | Spurious power transitions |
| C4 | `usbd.c:509-512` `tud_remote_wakeup` requires suspended && remote_wakeup_en; calls `dcd_remote_wakeup` | **§9.4.5** device remote wakeup | Correct gate | Medium if DCD resume signaling duration wrong (fsdev uses ESOF countdown `dcd_stm32_fsdev.c:237-238` 1–15 ms — matches **§7.1.7.7** resume drive) |
| C5 | No explicit **2 ms** timer enforcement on control transfers in usbd | Host aborts; device should still progress EP0 without blocking forever | Avoid unbounded busy-wait in DCD (see #3689 dwc2) | Host timeout / USBCV fails |
| C6 | GET_DESCRIPTOR `process_get_descriptor` (via `usbd.c:1020-1021`) | **§9.4.3** | Need length clamp to wLength (typical in implementation — verify in full function) | Descriptor overread if not clamped | Rare host issues |

### 3.2 USBCV

**BLOCKER:** USB-IF **USBCV not installed** on this machine (`which USBCV usbcv USB30CV` empty; no Application bundle). Cannot execute Chapter 9 / class tests here. **1.0 gate:** run USBCV 2.0 on at least `stm32f407disco` + `raspberry_pi_pico` + one HS dwc2 board with `device/cdc_msc` and `device/hid_composite`.

### 3.3 Class gap analysis → 1.0 policy

See table in §1.5. **Ship latest-stable class revs already in tree (HID 1.11, UAC2, MIDI2 optional). Do not block 1.0 on UAC3/UVC1.5 completeness.**

---

## Phase 4 — Memory safety audit

### Tooling status

| Tool | Status |
|---|---|
| clang `--analyze` on `usbd.c` alone | Insufficient includes/MCU defs; not a substitute for `compile_commands.json` analysis |
| **cppcheck** | **Not installed** |
| **scan-build** | **Not installed** |
| **ceedling** | Gem 1.0.1 present but **not on PATH** — unit tests not run this session |
| PVS-Studio skill | Available in repo (`.claude/skills/pvs`) — not executed this session (license/env) |

### Findings (code review severity ≥ Medium; not all confirmed with harness)

| ID | Sev | Location | Issue | Reproducer status |
|---|---|---|---|---|
| M1 | Med | Control OUT path `usbd.c:917-923` | Clamps `xferred_bytes` before memcpy — **good**; ensure all class drivers mirror | Covered by pattern |
| M2 | Med | Host hub / multi-device | Detach races historically (#3243 imxrt hub removal) | Needs hardware |
| M3 | Med | RP2040 MSC partial READ10 | #3188 panic `ep was already available` | External repro repo cited in issue |
| M4 | Med | dwc2 `edpt_disable` busy-wait | #3689 can block usbd task ~5s — priority inversion / apparent hang | Windows UAC2 reopen |
| M5 | Low–Med | Descriptor parsers in class drivers | Prefer bounded reads with `tu_desc_*` helpers — audit incomplete without analyzer | Pending |

### Fixes shipped this audit

**None applied to tree** (audit-first; constraints forbid inventing untested patches). Recommended top-5 fix PRs for follow-up weeks:

1. dwc2 ISO activate: skip disable if EP idle / timeout busy-wait (#3689).  
2. RP2040 double-arm transfer guard (#3188).  
3. SET_ADDRESS latch-after-status audit across all DCDs (C1).  
4. Host hub detach cleanup regression test.  
5. Enable ceedling in CI path + fifo/descriptor unit tests.

Each fix must: pass repro, build ≥5 BSPs, no public API break, add regression test — **not completed in this session**.

Outputs dir: `/tmp/static-analysis/` (placeholder; clang text may be empty/error).

---

## Phase 5 — Performance / footprint

HEAD measurements above. Tag comparison via Make **in progress** at `/tmp/tinyusb-footprint-compare.json`.

Known community regression: **#3688** RP2 host code much bigger (~+2.5k HID host) — treat as 1.0 footprint budget item with Kconfig-style feature gates.

**Bisect policy:** not run for tags because CMake failed; after Make baseline lands, bisect only cells with **>3% text growth**.

`.elf` composition: use `arm-none-eabi-nm --print-size --size-sort` and `arm-none-eabi-size --format=sysv` on successful ELF under `/tmp/tinyusb-matrix-builds/` (example: stm32f407disco cdc_msc). Sample nm tops should be regenerated post-matrix freeze.

---

## Phase 6 — Host stack audit

| Area | Finding | Sev | Notes |
|---|---|---|---|
| Hub depth | `CFG_TUH_HUB` multi-hub interfaces (`hub.c`); multi-tier depends on hub driver chaining | Med | Document supported tier depth for 1.0 (recommend **1 tier certified**, 2 tier best-effort) |
| Endpoint allocation under hubs | Improved on RP2 (size cost #3688) | Med | Offer compile-time slim path |
| Iso bandwidth | Limited explicit FS/HS budget accounting in portable HCD | Med | Post-1.0 for SS; FS/HS needs audit per port |
| Detach / cleanup | #3243 fixed for imxrt; general race remains risk | Med | Add HIL unplug matrix |
| Enum races hub vs device | Event queue serialization in `usbh` helps; ISR still must only queue | Med | Keep defer-to-task rule |

**Top 3 fixes (planned, not implemented):** hub detach completeness; EP allocation opt-out; iso bandwidth reservation stubs with fail-clean.

**Host examples:** `host/cdc_msc_hid`, `host/hid_controller` **built successfully** on stm32f407disco / stm32f411blackpill / stm32f439nucleo (see matrix). Latency measurement requires hardware — **not run**.

---

## Phase 7 — FreeRTOS integration

File: `src/osal/osal_freertos.h`.

| Topic | Finding | Risk |
|---|---|---|
| Priority inversion | Mutexes via FreeRTOS; USB IRQ must only `xQueueSendFromISR` / give sem — DCD busy-wait (#3689) **violates** RTOS assumptions | High on dwc2 ISO |
| Stack overflow | App defines `usbd_task` / `usbh_task` stacks in examples — no stack watermark in osal | Med — document minimum stack (recommend ≥2–3 KB with logs) |
| Mutex patterns | Static alloc supported (`configSUPPORT_STATIC_ALLOCATION`) | Good |
| `osal_task_get_current_handle` | Requires `INCLUDE_xTaskGetCurrentTaskHandle` or `configUSE_MUTEXES` (`osal_freertos.h:88-94`) | Build break if misconfigured — OK |
| ISR signaling overhead | One queue receive per event in `tud_task` — acceptable | Low |

**Benchmark before/after:** not run (no FreeRTOS BSP build timed this session). Examples: `cdc_msc_freertos`, `hid_composite_freertos` exist under `examples/device/`.

---

## Phase 8 — RISC-V / future MCU

### Supported in-tree (BSP + portable)

| MCU family | Board examples | Portable | OPT_MCU (`tusb_option.h`) |
|---|---|---|---|
| GD32VF103 | sipeed_longan_nano | dwc2 | 1600 |
| Fomu | fomu | valentyusb/eptri | 600 |
| CH32V103 | bluepill, r1_1v0 | wch usbfs | 2230 |
| CH32V20X | several | wch usbfs (+ fsdev option) | 2220 |
| CH32V307 | r1_1v0, nanoch32v305 | wch usbhs/usbfs | 2200 |
| CH583 | ch582m_evt | wch usbfs | CH583 define in family |
| HPMicro | hpm6750evk2 | chipidea HS + EHCI | 2600 |

### Community PRs (open, search riscv/ch32/gd32vf/hpmicro): **8** including CH32 async host (#3682), CH32H41X (#3642), HIL throughput (#3735).

### Gaps blocking more RISC-V

- Toolchain diversity (xpack gcc, vendor GCC) not unified in CI.  
- No shared `portable/riscv` — each vendor IP reused (dwc2/wch/chipidea) — **correct abstraction**; gap is **BSP/clock/IRQ**, not USB core.  
- Atomic / barrier assumptions in common code need RVWMO review (not done).

### Sketch: unsupported target with interest — **CH32H41x** (PR #3642)

- Reuse `src/portable/wch/` USBHS/FS with new `OPT_MCU_CH32H41X` and `hw/bsp/ch32h41x/`.  
- Blocker here: **no CH32 toolchain in PATH** this session; cannot build. Document for post-1.0 fast-follow.

---

## Phase 9 — Cross-version validation

| Fix area | Applies clean to 0.18.0? | Notes |
|---|---|---|
| dwc2 ISO activate | Likely needs manual port | File layout stable but line drift |
| RP2040 MSC guard | Port | rp2040 portable evolved |
| SET_ADDRESS DCD audit | Docs-only may apply | Per-port |
| Host hub detach | Check #3243 merge ancestry | |

**Not executed:** checkout v0.18.0 + cherry-pick (CMake/Make mismatch). Mark **incomplete**.

---

## Phase 10 — Ship mechanics

| Step | Status |
|---|---|
| Branch `tinyusb-1.0-roadmap` from master | Created locally |
| Signed commits per phase with fixes | **No code fixes** → single **docs/roadmap** commit allowed |
| Draft PR per phase on vedikatai/tinyusb | Repo created under vedikatai (empty history); push branch + open **one tracking PR** with this doc (phases linked in body) |
| Never push hathach/tinyusb | Honored (`origin` remains hathach; `vedikatai` remote added) |

---

## Implementation effort (developer-weeks)

| Phase | Effort (dev-weeks) | Notes |
|---:|---:|---|
| 1 Inventory automation in CI | 1 | Scripts for LOC/matrix |
| 2 Full build matrix + nightly | 3–4 | All families with `get_deps` cache |
| 3 USBCV + SET_ADDRESS audit all DCDs | 3–5 | Needs hardware + USB-IF tools |
| 4 Static analysis + top fixes | 3–4 | cppcheck + clang + PVS |
| 5 Footprint budgets + #3688 gates | 2–3 | |
| 6 Host hub/iso hardening | 3–4 | |
| 7 FreeRTOS guide + benchmarks | 1–2 | |
| 8 RISC-V CI (CH32/HPM/GD32VF) | 2–3 | |
| 9 Backport policy | 1 | |
| 10 Release engineering 1.0 | 2 | Changelog, API freeze |
| **Total** | **22–34** | Calendar 4–6 months @ 1–2 FTE |

## Risk matrix

| Change | Risk | Mitigation |
|---|---|---|
| Deprecate BTH/Printer/USBTMC | Low (few users) | `#warning` 1.0, remove 2.0 |
| SET_ADDRESS DCD unify | Med (enum regressions) | Per-port HIL |
| dwc2 ISO timeout | Med | Feature flag + USBCV audio |
| Host EP allocator opt-out | Low | Default full allocator |
| USB3/4 early merge | **High** | Keep out of 1.0 |

---

## What stays / deprecates / foundations (decision table)

| Area | 1.0 | 1.1 | Post-1.0 (USB3.2/USB4) |
|---|---|---|---|
| USB 2.0 device/host core | **Stay / freeze API** | Bugfix | Extend speed enums |
| CDC HID MSC MIDI DFU Vendor | **Stay** | — | — |
| Audio UAC2 / Video | Stay (best effort ISO) | Harden | UAC3 |
| Net NCM | Stay | — | — |
| BTH Printer USBTMC MTP | Optional | **Deprecate candidates** | — |
| typec | Experimental | — | **Foundation** with PD |
| SuperSpeed DCD | — | Research | **Foundation** |
| RISC-V WCH/HPM | Stay best-effort | CI expand | — |

---

## Citations index

- Commit `0a25cc27d7d3699536f6df01e80af3eb0423ce58`  
- `src/device/usbd.c:840-890,969-975,780-795,509-512`  
- `src/portable/st/stm32_fsdev/dcd_stm32_fsdev.c:224-238,477-484`  
- `src/portable/synopsys/dwc2/dcd_dwc2.c:522-527`  
- `src/osal/osal_freertos.h:88-117`  
- `src/class/printer/printer.h:44`, `src/class/hid/hid.h` HID 1.11 sections, `src/class/bth/bth_device.h:45`  
- GitHub: issues #2714, #3188, #3243, #3425, #3688, #3689, #3696, #3740; open PR set for CH32  
- USB-IF: USB 2.0 consolidated 2025-06-03; USB 3.2 Rev 1.1 Jun 2022; USB4 v2.0 (library 2026)  
- Build outputs: `/tmp/tinyusb-build-matrix.json`, `/tmp/tinyusb-footprint-compare.json`  
- This document: `/tmp/tinyusb-1.0-roadmap.md` and `docs/roadmap/tinyusb-1.0-roadmap.md`

---

## Incomplete work (explicit)

1. Full Cartesian example × all `hw/bsp` boards.  
2. USBCV execution.  
3. cppcheck/clang-analyzer/PVS full tree.  
4. Top-5 memory-safety fixes + reproducers executed.  
5. Tag footprint deltas finalized (Make job).  
6. Host latency on hardware.  
7. FreeRTOS overhead benchmark.  
8. CH32H41x build.  
9. v0.18.0 cherry-pick validation.  
10. Per-phase draft PRs (single PR with roadmap unless fixes land).



## Appendix A — Final build matrix snapshot (this host)

Source: `/tmp/tinyusb-build-matrix.json` — **120 cells, 75 success, 45 fail/missing**.

HEAD make sizes (Debug/default make, not MinSizeRel cmake — different from cmake ELF):

| Example | Board | ELF (make HEAD) |
|---|---|---:|
| device/cdc_msc | stm32f411blackpill | 270076 |
| device/hid_composite | stm32f407disco | 252696 |
| device/msc_dual_lun | stm32f407disco | 250300 |
| device/cdc_msc | stm32f407disco | 277152 |

Tag 0.16.0–0.18.0 **make fail** in worktrees (likely missing family SDK layout / Python / board.mk divergence when only `hw/mcu`+`lib` symlinked). **Bisect not performed** — document as blocker; re-run on Linux CI image matching upstream.

CMake MinSizeRel highlights (success cells):

| Example | Board | ELF | BIN | Warn |
|---|---|---:|---:|---:|
| device/cdc_msc | stm32f407disco | 360620 | 27200 | 0 |
| device/cdc_msc | stm32f411blackpill | 354636 | 26768 | 0 |
| device/hid_composite | stm32f407disco | 331840 | 15820 | 0 |
| device/msc_dual_lun | stm32f407disco | 333800 | 32916 | 0 |
| device/net_lwip_webserver | stm32f407disco | 652088 | 52036 | 4 |
| device/net_lwip_webserver | stm32f439nucleo | 653568 | 52112 | 4 |
| host/cdc_msc_hid | stm32f407disco | 505608 | 34672 | 4 |
| host/hid_controller | stm32f407disco | 352656 | 22944 | 4 |

Failure clusters: entire `stm32f103_bluepill` and `feather_m0_express` (missing MCU deps); all `host/*` on `nrf52840dk`; `device/cdc_dual_ports` @ `stm32f407disco` only (works on f411/f439 — investigate HS dual CDC).

Public PR: draft on vedikatai/tinyusb branch `tinyusb-1.0-roadmap` → `master`.
