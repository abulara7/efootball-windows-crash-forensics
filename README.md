# eFootball Windows Crash Forensics

![eFootball Windows Crash Forensics](assets/efootball-crash-forensics.png)

## Native Crash Investigation & Verified Remediation

A technical case study documenting the investigation of a reproducible `eFootball.exe` startup crash on Windows 11 24H2.

The application presented a completely silent failure:

```text
Steam
  ↓
eFootball.exe
  ↓
Black screen
  ↓
Process termination
  ↓
No visible error
```

Instead of repeatedly reinstalling drivers, Windows, DirectX, or the game, the investigation was approached as a native Windows crash-analysis problem.

---

## Case Summary

| Component        | Observed Environment           |
| ---------------- | ------------------------------ |
| OS               | Windows 11 Pro 64-bit          |
| Windows Build    | 26100 / 24H2                   |
| Game             | eFootball 5.5.1.0              |
| Distribution     | Steam                          |
| CPU              | AMD Ryzen 5 5500               |
| GPU              | NVIDIA GeForce GTX 1660 Ti 6GB |
| NVIDIA Driver    | 32.0.16.1088 / 610.88          |
| RAM              | 32 GB                          |
| Secure Boot      | Enabled                        |
| Memory Integrity | Disabled                       |
| Failure          | `0xC00000FD`                   |
| Failure Type     | `STATUS_STACK_OVERFLOW`        |

---

# 1. Initial Symptom

The game previously worked correctly.

After the Windows environment changed, launching eFootball produced:

1. Steam launched the executable.
2. A black screen appeared.
3. The process terminated.
4. No useful error dialog was displayed.

Conventional troubleshooting had already been exhausted:

* Steam file integrity verification
* Visual C++ runtime verification
* DirectX runtime verification
* Windows update rollback
* Game configuration reset
* Fullscreen optimization changes
* Administrator execution
* Windowed launch testing
* GPU detection verification
* Windows Security checks

None of these resolved the crash.

---

# 2. Event Viewer

Windows reported:

```text
Faulting application: eFootball.exe
Version: 5.5.1.0

Faulting module: KERNELBASE.dll

Exception code:
0xC00000FD
```

`0xC00000FD` corresponds to:

```text
STATUS_STACK_OVERFLOW
```

This immediately changed the investigation.

The problem was no longer treated as a missing DLL or a conventional graphics configuration failure.

---

# 3. Crash Dump Analysis

A full user-mode dump was captured and analyzed using WinDbg.

The dump showed:

```text
Problem Class:
STACK_OVERFLOW
```

and:

```text
FAILURE_BUCKET_ID:
STACK_OVERFLOW_c00000fd_eFootball.exe!Unknown
```

The stack contained a repeating sequence around the same eFootball code region.

The important observation was that the same internal address appeared repeatedly until the thread stack was exhausted.

Approximately thousands of recursive frames were present before termination.

---

# 4. KERNELBASE.dll Was Not the Root Cause

One of the most important findings was avoiding a misleading conclusion.

The exception manifested around:

```text
KERNELBASE!MultiByteToWideChar
```

However, the repeated call chain originated inside:

```text
eFootball.exe
```

Therefore:

```text
KERNELBASE.dll
```

was treated as the location where the stack exhaustion became visible, not automatically as the source of the bug.

This distinction was critical.

---

# 5. Symbol Interpretation

WinDbg initially displayed the repeating region using a symbol resembling:

```text
agsCheckDriverVersion
```

This could easily lead to the incorrect conclusion that AMD GPU Services or AMD graphics hardware was responsible.

However:

* The system used an NVIDIA GTX 1660 Ti.
* The Ryzen 5 5500 has no integrated GPU.
* No AMD graphics modules were present in the process.
* NVIDIA and DirectX modules were present.
* The repeating stack itself remained inside eFootball.

Therefore the symbol name was not treated as proof of an AMD-related root cause.

The actual executable code region had to be investigated instead.

---

# 6. Recursive Failure Pattern

Assembly analysis revealed a repeating control-flow pattern in the affected region.

Conceptually:

```text
Initial path/string processing
        ↓
eFootball internal routine
        ↓
delimiter/path processing
        ↓
recursive re-entry
        ↓
same routine
        ↓
same routine
        ↓
same routine
        ↓
...
        ↓
~5,400 stack frames
        ↓
STATUS_STACK_OVERFLOW
```

The routine was consuming stack space on every iteration.

Eventually the thread reached the stack boundary and Windows raised:

```text
0xC00000FD
```

---

# 7. The Important Clue: Filesystem Paths

Further investigation shifted attention from GPU configuration toward external path input.

The Windows Documents location was not a conventional local path.

The affected environment contained an Arabic-named OneDrive Documents directory:

```text
...\OneDrive\المستندات\...
```

The system ANSI code page was standard rather than UTF-8.

The evidence pointed toward a compatibility problem between the game's legacy path/string processing and the external filesystem path being supplied during startup.

The relevant failure mechanism was an invalid/unexpected path representation reaching a recursive path-processing routine that failed to terminate.

---

# 8. Remediation

The executable was **not patched**.

No DRM or anti-cheat component was bypassed.

No Windows system DLL was replaced.

No NVIDIA driver was modified.

The remediation focused on removing the problematic external input.

### Documents path

The Windows Documents shell-folder mappings were restored to a conventional local path:

```text
C:\Users\<user>\Documents
```

The existing eFootball/KONAMI data was migrated to the corrected location.

The active Windows shell environment was refreshed.

An existing conflicting per-application compatibility override was also removed.

---

# 9. Before / After

### Before

```text
Windows Documents
        ↓
OneDrive
        ↓
Arabic-named directory
        ↓
eFootball startup path processing
        ↓
Recursive path handling
        ↓
~5,400 frames
        ↓
0xC00000FD
        ↓
Crash
```

### After

```text
Windows Documents
        ↓
Local standard path
        ↓
eFootball startup
        ↓
Normal initialization
        ↓
Game process remains alive
```

---

# 10. Reproduction & Validation

The investigation included a controlled baseline reproduction.

### Baseline

The original crash was successfully reproduced:

```text
0xC00000FD
KERNELBASE.dll
eFootball.exe
```

A new crash dump was generated.

### Validation Run 1

After path remediation:

```text
eFootball launched successfully.

Process initialized:
~1.67 GB RAM
172 threads

KONAMI save data was successfully written.
```

### Validation Run 2

The game was terminated cleanly and launched again.

```text
eFootball launched successfully.

Process initialized:
~1.59 GB RAM
177 threads

Process remained responsive.
```

No new:

```text
0xC00000FD
```

crash was observed during validation.

---

# 11. Final Result

```text
STATUS: RESOLVED
```

The original black-screen startup failure was eliminated without:

* modifying the game executable
* modifying Windows system DLLs
* downgrading the NVIDIA driver
* disabling Secure Boot
* disabling Windows security
* bypassing DRM
* bypassing anti-cheat
* reinstalling Windows
* reinstalling the game

The remediation targeted the external filesystem/path condition that was feeding the failing startup path.

---

# 12. Investigation Methodology

The investigation followed this progression:

```text
Symptom
   ↓
Event Viewer
   ↓
Exception Code
   ↓
Crash Dump
   ↓
WinDbg
   ↓
Stack Analysis
   ↓
Repeated Frames
   ↓
Assembly Inspection
   ↓
Module Correlation
   ↓
Input Boundary Investigation
   ↓
Filesystem Path Discovery
   ↓
Controlled Remediation
   ↓
Independent Reproduction
   ↓
Verification
```

This approach is intentionally different from conventional troubleshooting.

The goal was not:

> "Try another driver and see what happens."

The goal was:

> "Identify exactly what the process was doing when it failed, determine what input caused that execution path, remove the trigger, and reproduce the result."

---

# 13. Key Lessons

### 1. Faulting module ≠ root cause

Seeing:

```text
KERNELBASE.dll
```

does not automatically mean `KERNELBASE.dll` is corrupted.

### 2. Exception codes matter

`0xC00000FD` immediately pointed the investigation toward stack exhaustion rather than missing runtime components.

### 3. Repeating stack frames are extremely valuable

Thousands of identical or near-identical frames can expose recursion that is invisible from the application's UI.

### 4. Symbol names require validation

A symbol such as:

```text
agsCheckDriverVersion
```

should not automatically be interpreted as proof that AMD hardware or AMD drivers are responsible.

### 5. External state can trigger native crashes

The executable itself may be perfectly intact while an unexpected path, environment value, configuration state, or OS-provided string drives it into an invalid execution path.

### 6. A fix needs reproduction

A registry change being accepted by Windows is not proof of a fix.

The actual application must be launched and the original failure condition must disappear.

---

# 14. Safety Principles

This investigation deliberately avoided:

```text
❌ Binary patching
❌ DRM bypass
❌ Anti-cheat bypass
❌ Windows DLL replacement
❌ Driver deletion
❌ Random registry modifications
❌ Blind compatibility flags
❌ OS reinstallation
```

The remediation was:

```text
Reversible
Minimal
Application-focused
Evidence-driven
Validated by real execution
```

---

# 15. Evidence

The repository is intended to document the investigation methodology and sanitized diagnostic evidence.

Sensitive information should be removed before publishing:

* Windows usernames
* Steam IDs
* absolute personal paths
* OneDrive account information
* crash dump files containing personal data
* registry exports containing unrelated user configuration

---

## Final Takeaway

A black screen with an immediate crash can look like a graphics-driver problem.

In this case, the useful breakthrough came from looking below the surface.

The final path was:

**Black Screen → Stack Overflow → Recursive Call Chain → Path Processing → Non-standard Documents Location → Controlled Remediation → Verified Successful Startup**

This is why native crash debugging is often less about finding the right setting and more about finding the right evidence.
