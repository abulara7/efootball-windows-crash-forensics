# Validation Results

## Baseline Reproduction

The original startup failure was reproduced before remediation.

Observed:

```text
Application:
eFootball.exe

Exception:
0xC00000FD

Exception Type:
STATUS_STACK_OVERFLOW

Faulting Module:
KERNELBASE.dll
```

A crash dump was successfully generated for analysis.

---

## Validation Run 1

After applying the filesystem/path remediation:

```text
Launch Result:
SUCCESS

Application:
eFootball.exe

Observed Process State:
~1.67 GB RAM
172 threads

Additional Observation:
KONAMI/eFootball data was successfully written to the restored
local Documents path.
```

No `0xC00000FD` crash was observed.

---

## Validation Run 2

The application was cleanly terminated and launched again.

```text
Launch Result:
SUCCESS

Application:
eFootball.exe

Observed Process State:
~1.59 GB RAM
177 threads

Process State:
Responsive
```

No recurrence of the original stack-overflow crash was observed.

---

## Event Validation

The post-remediation validation did not reproduce the original:

```text
0xC00000FD
STATUS_STACK_OVERFLOW
```

failure.

No corresponding application crash was observed during the validation runs.

---

## Result

```text
BASELINE:
CRASH REPRODUCED

REMEDIATION:
APPLIED

VALIDATION RUN 1:
PASS

VALIDATION RUN 2:
PASS

FINAL STATUS:
RESOLVED
```

---

## Validation Principle

The investigation did not consider a configuration change to be a successful fix by itself.

Success required real application execution after remediation.

The final validation sequence was:

```text
Reproduce
    ↓
Analyze
    ↓
Remediate
    ↓
Launch
    ↓
Terminate
    ↓
Relaunch
    ↓
Confirm no recurrence
```
