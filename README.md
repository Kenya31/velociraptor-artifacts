# Windows.Forensics.Services.DeadDisk

Enumerates Windows services from an offline SYSTEM hive.

This artifact is designed for **dead disk forensic analysis**, providing improved accuracy over default Velociraptor artifacts by correctly resolving the active ControlSet and normalizing service execution paths.

---

## Overview

When analyzing Windows systems using disk images (dead disk forensics), the default Velociraptor artifact:

- `Windows.System.Services`

may not always return complete or accurate results.

Common issues include:

- Incorrect ControlSet usage
- Missing or incomplete service entries
- Unresolved environment variables in ImagePath (e.g. `%SystemRoot%`, `%windir%`)
- Difficulty identifying actual executable and DLL paths

This artifact addresses these issues by:

- Resolving the correct ControlSet from `Select\Current`
- Enumerating services directly from the offline `SYSTEM` hive
- Normalizing common Windows path variables
- Extracting executable paths and service DLLs
- Optionally calculating hashes and Authenticode information

---

## Features

- Accurate ControlSet resolution for offline analysis
- Works directly on `SYSTEM` hive (no live system dependency)
- Path normalization:
  - `%SystemRoot%`
  - `%windir%`
  - `\SystemRoot`
  - `system32`
- Extracts:
  - Service ImagePath
  - Resolved executable path
  - Service DLL path (if present)
- Optional:
  - File hashing
  - Authenticode signature information

---

## Parameters

| Name              | Description |
|-------------------|------------|
| `SystemHivePath`  | Path to the SYSTEM hive |
| `ControlSet`      | Optional override (e.g. ControlSet001). If empty, resolved automatically |
| `Calculate_hashes`| Calculate file hashes |
| `CertificateInfo` | Collect Authenticode information |
| `NameRegex`       | Filter services by name |

---

## Example Usage

```vql
SELECT * FROM Windows.Forensics.Services.DeadDisk(
  SystemHivePath="C:/Windows/System32/Config/SYSTEM"
)