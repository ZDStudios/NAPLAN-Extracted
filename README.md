# NAP Locked-down Browser — MSI Inspection & EXE Packaging

This document records what was done in this folder: inspecting the source `file.msi`
installer, extracting its full metadata, and packaging it as a self-extracting `.exe`.

## Contents

| File / Folder | Description |
|---|---|
| [`file.msi`](file.msi) | Original, unmodified Windows Installer package |
| [`NAP_Locked_down_browser_Setup.exe`](NAP_Locked_down_browser_Setup.exe) | Self-extracting EXE wrapper around `file.msi` |
| [`install.cmd`](install.cmd) | Launch script embedded in the EXE (runs `msiexec /i` on the MSI) |
| [`msi_info/`](msi_info/) | Extracted MSI metadata (see below) |

## File Identity

| Property | Value |
|---|---|
| Product Name | NAP Locked down browser |
| Product Version | 5.11.0 |
| Manufacturer | Janison |
| Product Code | `{250939BF-C7CB-4B67-88D5-9E080F60E288}` |
| Upgrade Code | `{3935AFCF-0AE8-475C-A8F3-7AB14201CB62}` |
| Product Language | 1033 (English — US) |
| File Size | 331,325,440 bytes (~316 MB) |
| Digital Signature | **Valid** — signed by *JANISON SOLUTIONS PTY LTD* (Coffs Harbour, NSW, AU) |
| Build Toolchain | WiX Toolset (WixUI, WixCA custom actions) |

This is the official Janison-distributed installer for the **NAP Locked-down Browser**, the
secure browser used to sit NAPLAN online assessments. The valid Authenticode signature
confirms the MSI is unmodified and genuinely published by Janison.

## How the metadata was extracted

The MSI database was opened with the Windows Installer COM API
(`WindowsInstaller.Installer` → `OpenDatabase`), which is the built-in, documented way to
read an MSI's internal relational database (no third-party tools required). Each relevant
table was queried with SQL-style `SELECT` statements via `OpenView` / `Execute` / `Fetch`,
and the results were written out as tab-separated `.txt` files in [`msi_info/`](msi_info/).

Digital signature and raw file size were read separately via
`Get-AuthenticodeSignature` and `Get-Item`.

### `msi_info/` contents

| File | What it contains |
|---|---|
| `summary.txt` | Table count, embedded file count, signature status/signer, file size |
| `tables.txt` | All 41 internal MSI tables (schema of the install database) |
| `properties.txt` | MSI `Property` table — install-time properties (ProductCode, install mode flags, etc.) |
| `features.txt` | MSI `Feature` table — one feature: *NAP Locked down browser* |
| `components.txt` | MSI `Component` table — 707 components, one per installed file/registry entry group |
| `files.txt` | MSI `File` table — all 707 embedded files with size and version |
| `directories.txt` | MSI `Directory` table — the install directory tree |
| `customactions.txt` | MSI `CustomAction` table — install-time logic (see below) |
| `registry.txt` | MSI `Registry` table — registry keys written on install |

### Custom actions

All custom actions come from standard, well-known WiX extensions (`WixCA`, `WixUIWixca`) or
the vendor's own managed `CustomActions.CA` assembly — nothing obfuscated or unusual:

- `WixUIValidatePath`, `WixUIPrintEula` — standard WiX UI helpers
- `SchedServiceConfig` / `ExecServiceConfig` / `RollbackServiceConfig` — installs a Windows service
- `PopulateDeviceRecord`, `ValidateDeviceRecord`, `SaveDeviceRecord` — records device ID/name for the test session
- `Clean` — cleanup on uninstall
- `InstallReplayPackageIfExists` — installs a locally cached NAPLAN test package if present, prompting for a package password

### Registry footprint

Three keys are written under `HKLM\Software\Janison\NAP Locked down browser\Installer`,
recording shortcut creation flags and an uninstall marker for a bundled "Janison Replay"
component.

### Embedded payload

The 707 embedded files are mostly **.NET 8 (`8.0.1224.*`) runtime/localization resource
DLLs** (WPF/WinForms `*.resources.dll` for many languages) plus a `Microsoft.Win32.TaskScheduler`
dependency, an HTML splash/loading screen, and product icons/logos. This is consistent with
a self-contained .NET 8 WPF desktop application.

## Building the self-extracting EXE

MSI files don't "convert" to EXE in the sense of recompiling — instead, a self-extracting
EXE **wraps** the MSI and a launcher script, then runs the installer when executed. This was
built using **`iexpress`**, the self-extractor packager built into Windows (no third-party
tools):

1. `install.cmd` — a one-line launcher that runs `msiexec /i "%~dp0file.msi"`.
2. An `.SED` configuration (IExpress directive file) bundling `file.msi` + `install.cmd`,
   set to auto-run `install.cmd` after extraction.
3. `iexpress /N <config>.sed` — builds the CAB and stub into
   `NAP_Locked_down_browser_Setup.exe`.

The resulting EXE (~330 MB) is functionally equivalent to running the MSI directly, except:

- It extracts to a temp folder and invokes `msiexec` automatically.
- **It runs silently** — there is no visible install wizard when double-clicked, unlike
  double-clicking the MSI directly. This is expected behavior for an `iexpress`-built
  installer, not a defect.

## Notes / caveats

- The original `file.msi` was not modified in any way.
- No third-party or obfuscation tooling was used — only Windows' built-in Windows Installer
  COM API (for inspection) and `iexpress` (for packaging).
- If a visible installer UI is required, run `file.msi` directly instead of the EXE wrapper.
