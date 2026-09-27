# Root Cause Analysis

## Executive Summary

The reproduced eFootball startup crash was ultimately associated with the application's handling of a non-standard Windows Documents path.

The affected path was located under OneDrive and contained an Arabic directory name:

```text
...\OneDrive\المستندات\...
```

The environment used a standard ANSI code page rather than UTF-8.

The investigation linked this path representation to the recursive execution pattern observed in the eFootball process.

The resulting recursion exhausted the thread stack and produced:

```text
0xC00000FD
STATUS_STACK_OVERFLOW
```

---

## Root-Cause Chain

The observed failure chain was:

```text
Non-standard Documents location
        ↓
OneDrive-backed path
        ↓
Arabic directory component
        ↓
Legacy ANSI/path processing
        ↓
Unexpected path representation
        ↓
Path parsing failure
        ↓
Recursive re-entry
        ↓
~5,400 stack frames
        ↓
Stack exhaustion
        ↓
0xC00000FD
```

---

## Why the Documents Path Mattered

The game successfully launched after the Documents environment was restored to a conventional local path.

This provided an important controlled experiment.

The remediation did not involve changing:

* The eFootball executable
* The NVIDIA driver
* Windows system DLLs
* Secure Boot
* DRM
* Anti-cheat components

Instead, the external filesystem condition feeding the startup path was changed.

The original failure disappeared after that condition was removed.

---

## Encoding Consideration

The system ANSI code page was verified during the investigation.

The environment was not using:

```text
Beta: Use Unicode UTF-8
```

The evidence indicated that the application's legacy string/path handling was sensitive to the representation of the non-ASCII filesystem path.

The investigation therefore treated the issue as a path/string compatibility condition rather than simply labeling it as a generic Unicode problem.

---

## Why the Failure Became a Stack Overflow

The path-processing routine did not terminate correctly under the failing input condition.

Instead, execution repeatedly re-entered the same internal processing path.

Conceptually:

```text
Input path
    ↓
Parse
    ↓
Invalid/unexpected state
    ↓
Re-enter parser
    ↓
Same state
    ↓
Re-enter parser
    ↓
...
```

Each recursive call consumed additional stack space.

Eventually:

```text
Stack limit reached
        ↓
STATUS_STACK_OVERFLOW
        ↓
0xC00000FD
```

---

## Evidence Supporting the Root-Cause Model

The investigation relied on multiple independent observations:

### 1. Reproducibility

The original crash could be reproduced under the affected environment.

### 2. Exception Classification

The crash consistently produced:

```text
0xC00000FD
```

### 3. Stack Pattern

WinDbg showed thousands of repeating frames.

### 4. External Path Condition

The affected environment used a OneDrive-backed Documents path containing an Arabic directory component.

### 5. Controlled Remediation

Restoring Documents to a standard local path removed the observed failure condition.

### 6. Independent Validation

The game successfully launched twice after remediation.

---

## Scope of the Conclusion

This case study documents the observed failure mechanism in the tested environment.

It does not claim that every eFootball installation with OneDrive, Arabic folder names, or non-UTF-8 Windows settings will reproduce the same crash.

The conclusion is intentionally scoped to the evidence collected from this specific reproducible failure.

---

## Root-Cause Statement

> In the tested Windows 11 environment, eFootball's startup path processing entered an uncontrolled recursive state when handling the affected non-standard Documents filesystem path. The recursion exhausted the thread stack and resulted in `0xC00000FD (STATUS_STACK_OVERFLOW)`. Restoring the Documents location to a conventional local path removed the triggering condition and resulted in successful subsequent launches.
