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

- WMI persistence
- Startup folders
- Improved command-line parsing (LOLBins / script execution)
