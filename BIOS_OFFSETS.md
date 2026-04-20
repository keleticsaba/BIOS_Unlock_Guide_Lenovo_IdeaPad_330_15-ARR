# Lenovo IdeaPad 330 15-ARR — BIOS Variable Offsets
## BIOS: 7VCN49WW (Insyde H2O)
## Extracted from: bioss.fd → SetupUtility PE32, CbsSetupDxeRV PE32

---

## EFI Variables on this system

| Variable Name | GUID | Live Size |
|---|---|---|
| `Setup` (SystemConfig) | `a04a27f4-df00-4d42-b552-39511302113d` | 700 bytes |
| `AmdSetup` | `3a997502-647a-4c82-998e-52ef9486a247` | 294 bytes |
| `AMD_PBS_SETUP` | `a339d746-f678-49b3-9fc7-54ce0f9df226` | 128 bytes |

---

## GRUB setup_var commands

**Insyde BIOS uses `setup_var2` (not `setup_var`) — takes VarStore index as first arg.**

For `Setup` variable (VarStore 0x1234 = index 1 in IFR):
```
setup_var2 0x<offset> 0x<value>
```

For `AmdSetup`/CBS variable (VarStore 0x5000 = index 0 in IFR):
```
setup_var2 0x<offset> 0x<value>   # with correct VarStore context
```

**Always read before writing:**
```
setup_var2 0x238
```

---

## Setup Variable (`Setup` / SystemConfig) — VarStore 0x1234

| Offset | Setting | Values | Current |
|---|---|---|---|
| `0x9E` | **Above 4GB MMIO** | 0=Disabled, 1=Enabled (default) | check |
| `0x238` | **AMD SVM Technology** (virtualization) | 0=Disabled, 1=Enabled (default) | 0x01 |
| `0x23A` | **PowerXpress / Switchable Graphics** | 0=UMA only, 1=Discrete+iGPU (default) | 0x01 |
| `0x23C` | **Discrete GPU mode** | 0=UMA only, 1=Discrete (default) | 0x01 |
| `0x23E` | **Secure Rollback Prevention** | 0=Disabled, 1=Enabled (default) | check |
| `0x240` | **Fn Key Swap** | 0=Legacy Fx, 1=Hotkey mode (default) | 0x01 |
| `0x242` | **USB Power Share** | 0=Disabled, 1=Enabled (default) | check |
| `0x244` | **USB Charge in Battery Mode** | 0=Disabled, 1=Enabled (default) | 0x01 |
| `0x246` | **Battery Charging Logo** | 0=Disabled, 1=Enabled (default) | check |
| `0x248` | **System Performance Mode** | 0=Quiet, 1=Performance (default) | 0x01 |
| `0x24A` | **Fan/Thermal Mode** | 0=Quiet, 1=Performance (default) | 0x01 |
| `0x24C` | **Boot Mode** (Legacy support) | 0=UEFI, 1=Legacy (default) | check |
| `0x62`  | **SATA Port mode** | 0=Auto, 1=eSATA, 2=Legacy | check |
| `0x80`  | **SATA Controller mode** | 2=RAID, 3=AHCI (default), 4=IDE | 0x03 |
| `0x21E` | **Secure Boot** state | 0=Disabled, 1=Enabled (default) | check |

---

## AmdSetup Variable (CBS — AMD CPU settings) — VarStore 0x5000

| Offset | Setting | Values | Notes |
|---|---|---|---|
| `0x4`  | **Core Performance Boost** | 0=Disabled, 1=Auto (default) | Disables boost/turbo |
| `0x5`  | **Global C-state Control** | 0=Disabled, 1=Auto, 2=?, 3=? | IO C-states |
| `0x6`  | **OC Mode** | 0-5, 0=Auto | Enables manual OC |
| `0x6D` | **Downcore Control** | 0=Auto, 1=2core, 2=3core, 3=4core | Disable cores |
| `0x6E` | **SMT (Hyperthreading)** | 0=Disabled, 1=Auto | Needs power cycle |
| `0x7F` | **Memory Overclock** | 0=Auto, values = speed preset | DRAM OC |
| `0x81` | **tCL** (memory timing) | 0x8–0xFF | Manual timing |
| `0x82` | **tRCDrd** (memory timing) | 0x8–0xFF | Manual timing |

### Pstate tuning (per-core P-state, manual OC)
| P-state | FID offset | DID offset | VID offset | Notes |
|---|---|---|---|---|
| P0 (max boost) | `0x14` | `0x15` | `0x16` | COF = 200MHz × FID / DID |
| P1 | `0x20` | `0x21` | `0x22` | |
| P2 | `0x2C` | `0x2D` | `0x2E` | |
| P3 | `0x38` | `0x39` | `0x3A` | |

**FID range: 0x10–0xFF, DID range: 0x08–0x30**
Example: 3.6GHz = FID=0x48 (72), DID=0x08 (8) → 200 × 72 / 8 = 1800... use DID=0x0A for divisor

---

## SVM Lock / SMM Code Lock — VarStore 0x1234 (Setup variable)

| Offset | Setting | Values | Notes |
|---|---|---|---|
| `0xEF`  | **SVM Lock** | 0=Disabled, 1=Enabled | Locks SVM enable bit — disable to allow OS to toggle SVM |
| `0x14D` | **SMM Code Lock** | 0=Disabled, 1=Enabled | Locks SMM region — disable for SMM research/patching |

---

## GRUB USB Boot Instructions

1. Flash `grubx64.efi` from [grub-mod-setup_var](https://github.com/datasone/grub-mod-setup_var) to USB as `/EFI/BOOT/BOOTx64.efi`
2. Disable Secure Boot (BIOS → Security)
3. Boot USB → GRUB shell

**Read current value (always do this first!):**
```
setup_var2 0x238
```

**Write value:**
```
setup_var2 0x238 0x1
```

**For Insyde H2O, also try `setup_var3` if `setup_var2` fails.**

---

## Source files in this directory
- `bioss.fd` — extracted BIOS ROM
- `setup_ifr.txt` — SetupUtility IFR (SetupConfig variable / main BIOS settings)
- `cbs_ifr.txt` — AMD CBS IFR (AmdSetup variable / CPU/memory settings)  
- `amd_pbs_ifr.txt` — AMD PBS IFR (AMD_PBS_SETUP variable / platform settings)
- `fv_main.bin.dump/` — full UEFI module tree
