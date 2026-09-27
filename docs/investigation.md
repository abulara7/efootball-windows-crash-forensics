# Investigation

## Overview

This document describes the investigation of a reproducible startup crash affecting eFootball 5.5.1.0 on Windows 11 24H2.

The application previously operated normally. The failure appeared during startup and presented as a black screen followed by immediate process termination without a useful user-facing error message.

Rather than relying on repeated trial-and-error configuration changes, the investigation was approached as a native Windows crash-analysis problem.

---

## 1. Initial Symptom

The observed startup sequence was:

```text
Steam
    ↓
eFootball.exe
    ↓
Black screen
    ↓
Process termination
    ↓
No useful visible error
```

The crash was reproducible.

The objective was therefore to identify:

1. The actual exception.
2. The execution path that produced it.
3. Whether the reported faulting module was the actual root cause.
4. What external condition was influencing the failing execution path.
5. Whether a controlled remediation could eliminate the original failure.

---

## 2. Environment

| Component        | Observed Environment           |
| ---------------- | ------------------------------ |
| Operating System | Windows 11 Pro 64-bit          |
| Windows Version  | 24H2                           |
| Windows Build    | 26100                          |
| Game             | eFootball 5.5.1.0              |
| Distribution     | Steam                          |
| CPU              | AMD Ryzen 5 5500               |
| GPU              | NVIDIA GeForce GTX 1660 Ti 6GB |
| NVIDIA Driver    | 32.0.16.1088 / 610.88          |
| RAM              | 32 GB                          |
| Secure Boot      | Enabled                        |
| Memory Integrity | Disabled                       |

---

## 3. Preliminary Troubleshooting

Before the crash was analyzed at the native level, several conventional troubleshooting paths had already been investigated.

These included:

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

None of these eliminated the reproducible startup failure.

This was an important signal that the investigation should move below conventional application configuration troubleshooting.

---

## 4. Event Viewer Evidence

Windows Event Viewer identified:

```text
Faulting application:
eFootball.exe

Version:
5.5.1.0

Faulting module:
KERNELBASE.dll

Exception code:
0xC00000FD
```

The exception code corresponds to:

```text
STATUS_STACK_OVERFLOW
```

This changed the investigation direction.

Instead of assuming a graphics-driver or missing-runtime problem, the next objective became understanding why the process was exhausting its thread stack.

---

## 5. Crash Dump Collection

A user-mode crash dump was captured from the reproducible failure.

The dump was subsequently analyzed with WinDbg.

The analysis confirmed a stack-overflow failure and provided a much more useful view of the execution state than the original black-screen symptom.

---

## 6. Stack Analysis

The stack contained a repeating sequence involving the same eFootball internal code region.

The important observation was not simply that a crash occurred, but that execution was repeatedly returning to the same area until the available stack space was exhausted.

The observed pattern was consistent with uncontrolled recursion.

The stack contained approximately thousands of recursive frames, with the investigation estimating roughly:

```text
~5,400 frames
```

before stack exhaustion.

---

## 7. Investigation Direction

The investigation then progressed through several layers:

```text
Startup Symptom
      ↓
Event Viewer
      ↓
Exception Code
      ↓
Crash Dump
      ↓
WinDbg
      ↓
Stack Repetition
      ↓
Assembly / Code-Region Analysis
      ↓
Input Boundary Investigation
      ↓
Filesystem Path Investigation
      ↓
Controlled Remediation
      ↓
Independent Validation
```

This progression was important because it prevented the investigation from becoming a sequence of unrelated configuration experiments.

---

## 8. The Filesystem Clue

The investigation eventually shifted attention toward filesystem path input.

The affected Windows environment used a non-standard Documents location involving OneDrive and an Arabic-named directory:

```text
...\OneDrive\المستندات\...
```

The system ANSI code page was also verified as a standard non-UTF-8 configuration.

The evidence pointed toward an interaction between the external filesystem path representation and legacy path/string processing occurring during eFootball startup.

The resulting path-processing state was associated with the recursive execution pattern observed in the crash dump.

---

## 9. Controlled Remediation

The remediation did not modify the eFootball executable.

The following approach was used:

1. Restore the Windows Documents shell-folder mapping to a standard local Documents path.
2. Migrate the existing KONAMI/eFootball data to the corrected location.
3. Remove the conflicting per-application compatibility override identified during the investigation.
4. Refresh the Windows shell environment.
5. Relaunch eFootball.
6. Perform a second clean launch to validate persistence.

---

## 10. Validation

The original crash was reproduced before remediation.

After remediation, eFootball successfully launched twice during validation.

The game successfully initialized and wrote its save data to the restored local Documents path.

No subsequent `0xC00000FD` crash was observed during the validation runs.

---

## 11. Investigation Outcome

The investigation established a reproducible relationship between:

```text
Non-standard Documents path
        ↓
Startup path processing
        ↓
Recursive execution
        ↓
Stack exhaustion
        ↓
0xC00000FD
```

The controlled path remediation eliminated the observed startup failure in the tested environment.

---

## Conclusion

The investigation demonstrates the value of moving from symptom-based troubleshooting to evidence-driven native crash analysis.

The key breakthrough was not another driver installation or compatibility flag.

It was identifying what the process was actually doing when the failure occurred, tracing the repeated execution pattern, investigating the external input feeding that path, and then validating the remediation through real application execution.
