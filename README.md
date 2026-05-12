# Velociraptor Dead Disk Forensics Artifacts

This repository contains custom Velociraptor artifacts designed for **dead disk forensic analysis**.

These artifacts focus on improving accuracy and visibility when analyzing **offline Windows systems (disk images)** using `raw_reg` and `ntfs` accessors.

---

## Overview

Default Velociraptor artifacts are primarily designed for live response and may not always provide reliable results when used against offline disk images.

Common challenges in dead disk analysis include:

- Incorrect registry hive interpretation
- Missing or incomplete persistence data
- Unresolved environment variables (e.g. `%SystemRoot%`)
- Difficulty mapping execution paths to actual binaries

This repository provides artifacts that address these limitations by:

- Directly parsing offline registry hives
- Directly parsing offline Scheduled Task XML files
- Normalizing Windows path patterns
- Extracting execution paths from persistence mechanisms
- Enriching results with hash and signature information

---

## Artifacts

### 1. Windows.Forensics.Services.DeadDisk

Enumerates Windows services from an offline `SYSTEM` hive.

#### Key Features

- Accurate ControlSet resolution (`Select\Current`)
- Service enumeration from offline hive
- Path normalization:
  - `%SystemRoot%`
  - `%windir%`
  - `\SystemRoot`
  - `system32`
- Extracts:
  - ImagePath
  - Resolved executable path
  - Service DLL path
- Optional:
  - File hash
  - Authenticode information

---

### 2. Windows.Forensics.RunKeys.DeadDisk

Enumerates Run / RunOnce persistence from offline registry hives.

#### Key Features

- Supports:
  - HKLM Run / RunOnce
  - HKCU Run / RunOnce (all users)
- Works directly on:
  - `SOFTWARE` hive
  - `NTUSER.DAT`
- One registry value per row (normalized output)
- Extracts executable path from command line
- Basic path normalization:
  - `%SystemRoot%`
  - `%windir%`
  - `\SystemRoot`
  - `system32`
- Enriches results with:
  - File hash
  - Authenticode information

#### Notes

- `ResolvedPath` represents the extracted launcher path from the command line.
- It may not always represent the final payload (e.g. `cmd.exe`, `powershell.exe`, `rundll32.exe`).
- Hash and signature data may be empty if:
  - Path is relative or unresolved
  - File does not exist in the disk image
- File type does not support Authenticode

---

### 3. Windows.Forensics.ScheduledTasks.DeadDisk

Detects Scheduled Task persistence from offline task XML files.

#### Key Features

- Reads task definitions directly from:
  - `C:/Windows/System32/Tasks/**`
- Does not rely on:
  - live WMI
  - live Task Scheduler APIs
- One task action per row
- Extracts:
  - task metadata
  - execution command
  - arguments
  - working directory
  - COM handler data
  - payload path (conservative normalization)
- Enriches results with:
  - SHA256
  - Authenticode information
  - file metadata when the referenced payload exists
- Handles missing payloads safely

#### Notes

- `PayloadPath` is intentionally conservative.
- Relative paths, user-scoped environment variables, and bare commands are not aggressively expanded.
- For launcher-style actions such as `cmd.exe`, `powershell.exe`, and `rundll32.exe`,
  `PayloadPath` may represent the launcher executable rather than the final payload.

---

### 4. Windows.Forensics.LogonPoints.DeadDisk

Enumerates practical logon-related persistence points from offline Windows hives and startup folders.

#### Key Features

- Focuses on Autoruns-style logon coverage not already handled by `RunKeys.DeadDisk`
- Supports:
  - `Policies\Explorer\Run`
  - `RunOnceEx`
  - `Terminal Server Install` Run / RunOnce / RunOnceEx
  - `UserInitMprLogonScript`
  - `Active Setup` `StubPath`
  - startup folder files and shortcuts
- Works directly on:
  - `SOFTWARE` hive
  - `SYSTEM` hive
  - `NTUSER.DAT`
  - `C:\ProgramData\...\Startup`
  - `C:\Users\*\...\Startup`
- One registry value or startup item per row
- Enriches results with:
  - file metadata
  - file hash
  - Authenticode information when available

#### Notes

- This artifact intentionally does not duplicate classic `Run` / `RunOnce` coverage.
- `ResolvedPath` is conservative and may represent a launcher executable instead of a final payload.
- `RunOnceEx` style keys can include metadata values such as `Title`; these are returned as registry data rather than suppressed.
- Current version does not include:
  - Group Policy script registry trees
  - `Winlogon`-specific entries such as `Shell` and `Userinit`

---

### 5. Windows.Forensics.WMI.DeadDisk

Enumerates carved WMI permanent event subscription evidence from an offline Windows disk image.

#### Key Features

- Reads WMI repository files directly from:
  - `C:\Windows\System32\wbem\Repository\OBJECTS.DATA`
  - `C:\Windows\System32\wbem\Repository\INDEX.BTR`
  - `C:\Windows\System32\wbem\Repository\MAPPING*.MAP`
  - `C:\Windows\System32\wbem\Repository\RECOVER.BTR`
  - `C:\Windows\System32\wbem\Repository\FS\**`
- Does not rely on:
  - live WMI APIs
  - registry-only assumptions about WMI persistence
- Provides:
  - carved `FilterToConsumerBinding` relationship rows
  - consumer name and type
  - filter name and query when recoverable
  - evidence text for analyst review
  - repository file inventory
  - SHA256 enrichment
  - raw observed WMI-related strings as fallback triage evidence

#### Notes

- This artifact reports carved repository facts only.
- `CarvedBindings` is the primary source for Autoruns-like review.
- `ObservedWMIStrings` is fallback triage evidence, not final persistence attribution.
- Full WMI subscription reconstruction requires a dedicated WMI repository parser.

---

## Parameters (RunKeys)

| Name              | Description |
|-------------------|------------|
| `UserHiveGlobs`   | Path pattern for NTUSER.DAT |
| `SoftwareHivePath`| Path to SOFTWARE hive |
| `CalculateHashes` | Enable hash calculation |
| `CertificateInfo` | Enable Authenticode collection |

---

## Parameters (ScheduledTasks)

| Name | Description |
|------|-------------|
| `TasksPath` | Path pattern for task XML files |
| `TaskNameRegex` | Filter task names |
| `CalculateHashes` | Enable SHA256 calculation |
| `CertificateInfo` | Enable Authenticode collection |

---

## Parameters (LogonPoints)

| Name | Description |
|------|-------------|
| `UserHiveGlobs` | Path pattern for NTUSER.DAT |
| `SoftwareHivePath` | Path to SOFTWARE hive |
| `SystemHivePath` | Path to SYSTEM hive |
| `ControlSet` | Optional ControlSet override |
| `StartupFolderGlobs` | Startup folder glob list |
| `CalculateHashes` | Enable hash calculation |
| `CertificateInfo` | Enable Authenticode collection |

---

## Parameters (WMI)

| Name | Description |
|------|-------------|
| `WMIObjectsDataGlobs` | Path patterns for `OBJECTS.DATA` files used by binding carving |
| `WMIRepositoryGlobs` | Path patterns for WMI repository files |
| `CarveBindings` | Enable carved `FilterToConsumerBinding` relationship output |
| `BindingRecordRegex` | YARA rule for finding binding records |
| `BindingContextBytes` | Context bytes around each binding hit |
| `BindingHitsPerFile` | Maximum binding hits per repository file |
| `EvidenceTextBytes` | Maximum evidence text bytes shown per carved binding |
| `ScanRepositoryStrings` | Enable YARA string scanning |
| `CalculateHashes` | Enable SHA256 calculation |
| `NumberOfHits` | Maximum YARA hits per repository file |
| `ContextBytes` | Context bytes around each YARA hit |
| `WMIYaraRule` | YARA rule for observable WMI strings |

---

## Example Usage

### Services

```vql
SELECT * FROM Windows.Forensics.Services.DeadDisk(
  SystemHivePath="C:/Windows/System32/Config/SYSTEM"
)
```

### RunKeys

```vql
SELECT * FROM Windows.Forensics.RunKeys.DeadDisk(
  SoftwareHivePath="C:/Windows/System32/config/SOFTWARE"
)
```

### ScheduledTasks

```vql
SELECT * FROM Windows.Forensics.ScheduledTasks.DeadDisk(
  TasksPath="C:/Windows/System32/Tasks/**"
)
```

### LogonPoints

```vql
SELECT * FROM Windows.Forensics.LogonPoints.DeadDisk(
  SoftwareHivePath="C:/Windows/System32/config/SOFTWARE",
  SystemHivePath="C:/Windows/System32/config/SYSTEM"
)
```

### WMI

```vql
SELECT * FROM Windows.Forensics.WMI.DeadDisk()
```

---

### Examples

Sample outputs are provided in the examples/ directory:
- sample_services.csv
- sample_services.json
- sample_HKCU_Run.csv
- sample_HKCU_Run.json

---

### Design Philosophy

These artifacts are designed with the following principles:
- Dead disk first (no live system dependency)
- Accuracy over completeness
- Readable, normalized output
- Minimal assumptions about execution context

---

### Future Work

- Winlogon-specific persistence
- Group Policy script persistence
- WMI repository parser integration for full subscription reconstruction
- Improved command-line parsing (LOLBins / script execution)
