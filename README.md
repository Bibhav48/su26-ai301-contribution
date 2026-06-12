# Contribution 1: Add command repeat support for u command

**Contribution Number:** 1  
**Student:** Bibhav Adhikari  
**Issue:** [pwndbg/pwndbg #1374](https://github.com/pwndbg/pwndbg/issues/1374)  
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose to contribute to `pwndbg` because optimizing low-level debugging tools aligns closely with my interest on systems-level programming. Enhancing the `nearpc` command's repetition behavior directly improves workflow efficiency for developers inspecting disassembly in GDB.

---

## Understanding the Issue

### Problem Description
In GDB, hitting a bare `Enter` key normally repeats the exact previous string execution. For command targets that read or disassemble memory, this behavior is unhelpful because it repeatedly outputs the exact same block of assembly. While `pwndbg` has custom plumbing to catch an `Enter` press and auto-advance the memory range for commands like `hexdump` and `telescope`, the `u` command (an alias for `nearpc`) does not. This forces users to manually pass progressive address targets to step through disassembly space.

### Expected Behavior
When a user runs the `u` (or `nearpc`) command on a memory location and subsequently hits the `Enter` key without any arguments, `pwndbg` should automatically calculate the address of the last disassembled instruction and render the next sequential block of instructions.

### Current Behavior
Hitting `Enter` after invoking the `u` command fails to advance the instruction pointer range, forcing a static re-execution of the identical memory block or losing tracking altogether.

### Affected Components
* `pwndbg/commands/__init__.py`: Contains the core `CommandObj` architecture, `check_repeated()` history tracking, and the target execution loop where the new `@repeatable` decorator will be centralized.
* `pwndbg/commands/nearpc.py`: Dictates the CLI command configuration wrapper for `nearpc` and the `u` alias.
* `pwndbg/aglib/nearpc.py`: Handles the underlying rendering mechanics and disassembly engine pipelines where the `repeat` flag is consumed.
* `pwndbg/commands/hexdump.py` & `pwndbg/commands/telescope.py`: Submodules currently duplicating ad-hoc tracking patterns that will be refactored to use the new decorator framework.

---

## Reproduction Process

### Environment Setup
*To be completed during Phase II environment configuration.*

### Steps to Reproduce
1. Launch a debugging target using the local development instance of `pwndbg`.
2. Execute the disassemble command on a target boundary: `u 0x4000` (or a known functional symbol).
3. Press `Enter` immediately afterward on a blank prompt.
4. *Observed result:* The disassembly frame fails to advance to the next instruction sequences.

### Reproduction Evidence
- **Commit showing reproduction:** [Pending local environment setup]
- **Screenshots/logs:** [Pending verification]
- **My findings:** Initial analysis shows that `CommandObj.check_repeated()` correctly sets `self.repeat = True` globally, confirming the repeat signal is active. However, `nearpc` currently lacks the underlying state tracking logic required to advance the address pointer.

---

## Solution Approach

### Analysis
The underlying repeat-detection infrastructure is already centrally handled inside `pwndbg/commands/__init__.py`. When an empty instruction is submitted, `CommandObj.check_repeated()` pulls the prior trace from `pwndbg.dbg.history(1)` to flag the state. Commands like `hexdump` manually intercept this by tracking a `<cmd>.last_address` state variable attached directly to the function wrapper. 

The root cause of the missing feature is that `nearpc` doesn't do this bookkeeping. Rather than copying this manual state assignment pattern into `nearpc.py` and increasing structural duplication, the correct architectural fix is to centralize this asset management into a core wrapper.

### Proposed Solution
1. **Build a `@repeatable` Decorator:** Design a reusable function decorator inside `pwndbg/commands/__init__.py` that can safely intercept commands, manage local state cache profiles like `last_address`, and seamlessly auto-feed advanced arguments back to the function block when a repeat is flagged.
2. **Apply to `nearpc`:** Cleanly tag `nearpc` with the new decorator so it catches the ending pointer offset from the disassembly engine and sequences fluidly.
3. **Refactor Existing Implementations:** Transition `hexdump` and `telescope` over to the new decorator to delete dead boilerplate code and prove global feature compatibility.

### Implementation Plan
Using the UMPIRE framework (adapted):

**Understand:** Ensure hitting `Enter` increments disassembly addresses based on prior frame outputs.

**Match:** Mirror the core intent of `hexdump.py` address offset shifting, but convert the implementation pattern into a generic decorator strategy within `CommandObj`.

**Plan:**
1. Setup and fork validation tracking inside the local repository workspace.
2. Draft the structural `@repeatable` decorator layout within `pwndbg/commands/__init__.py`.
3. Modify `pwndbg/commands/nearpc.py` to route execution variables through the decorator.
4. Verify pointer math handles variable-width instruction bounds cleanly across distinct system architectures.
5. Apply pattern refactoring to cleanup `hexdump` code footprints.

**Implement:** [Pending branch creation]  
**Review:** [Pending self-review checklist]  
**Evaluate:** Launch an active session within GDB to manually assert seamless command repetition across a minimum of 10 sequential iterations.

---

## Testing Strategy

### Unit Tests
- [ ] Test case 1: Validate decorator correctly saves and preserves state attributes on an execution profile.
- [ ] Test case 2: Verify decorator falls back safely to baseline arguments when an explicit parameter overrides a repeat state.

### Integration Tests
- [ ] Integration scenario 1: Assert `u` tracking advances memory blocks cleanly across varying structural platforms (x86/64 vs ARM targets).

### Manual Testing
*To be logged during Phase III testing phases.*

---

## Implementation Notes

### Week 1 Progress
Identified core logic intersections across `pwndbg/commands/__init__.py` and officially claimed open issue #1374.

### Week 2 Progress (Current)
Completed the initial repository tracking, conducted a deep architectural code review of the codebase, identified core logic intersections within `pwndbg/commands/__init__.py`, and officially claimed the open issue.

### Week 3 Progress
*To be updated during execution.*

### Code Changes
- **Files modified:** None (Phase I)
- **Key commits:** None (Phase I)
- **Approach decisions:** Chose an architectural decorator approach over an isolated, local fix inside `nearpc.py` to fulfill long-term framework maintainer goals and remove structural duplication across `hexdump.py`.

---

## Pull Request
**PR Link:** [GitHub PR URL when submitted]  
**PR Description:** [Draft or final PR description - much of the content above can be adapted]  

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained
*To be completed following PR review cycles.*

### Challenges Overcome
*To be completed following PR review cycles.*

### What I'd Do Differently Next Time
*To be completed following PR review cycles.*

---

## Resources Used
- [pwndbg/pwndbg GitHub Repository](https://github.com/pwndbg/pwndbg)
- [GDB Python API Official Documentation](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Python-API.html)
- [CodePath AI301 Capstone Guide Portal](https://courses.codepath.org/courses/ai301)
