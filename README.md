# Contribution 1: Add command repeat support for u command

**Contribution Number:** 1  
**Student:** Bibhav Adhikari  
**Issue:** [pwndbg/pwndbg #1374](https://github.com/pwndbg/pwndbg/issues/1374)  
**Status:** Phase II Complete

---

## Why I Chose This Issue

I chose to contribute to `pwndbg` because optimizing low-level debugging tools aligns closely with my interest in systems-level programming. Enhancing the `nearpc` command's repetition behavior directly improves workflow efficiency for developers inspecting disassembly in GDB and LLDB. Working on this issue offered a great opportunity to explore the interface boundary between custom Python wrappers and native debugger backends.

---

## Understanding the Issue

### Problem Description

In GDB, hitting a bare `Enter` key normally repeats the previous command string execution. For command targets that read or disassemble memory, this behavior is unhelpful because it repeatedly outputs the exact same block of assembly. While `pwndbg` has custom plumbing to catch an `Enter` press and auto-advance the memory range for commands like `hexdump` and `telescope`, the `u` command (an alias for `nearpc`) did not function properly on the LLDB backend. This forced users to manually pass progressive address targets to step through disassembly space.

### Expected Behavior

When a user runs the `u` (or `nearpc`) command on a memory location and subsequently hits the `Enter` key without any arguments, `pwndbg` should automatically calculate the address of the last disassembled instruction and render the next sequential block of instructions.

### Current Behavior

Hitting `Enter` after invoking the `u` command under LLDB failed to advance the instruction pointer range, forcing a static re-execution of the identical memory block or losing tracking altogether.

### Affected Components

*   `pwndbg/commands/__init__.py`: Contains the core `CommandObj` architecture, `check_repeated()` history tracking, and the target execution loop.
*   `pwndbg/commands/nearpc.py`: Dictates the CLI command configuration wrapper for `nearpc` and the `u` alias.
*   `pwndbg/aglib/nearpc.py`: Handles the underlying rendering mechanics and disassembly engine pipelines where the `repeat` flag is consumed.
*   `pwndbg/dbg_mod/lldb/__init__.py`: LLDB backend implementation, including stubbed `history()` and dynamic `CommandHandler` class registration.
*   `pwndbg/dbg_mod/lldb/repl/__init__.py`: Interactive REPL loop that handles the console prompts and user inputs.

---

## Reproduction Process

### Environment Setup

*   **Operating System:** macOS Tahoe (26.x) running on arm64 Apple Silicon.
*   **Environment:** Local `pwndbg` development environment synced using Python `uv sync --all-groups --all-extras` within `.venv`.
*   **Debugger Interface:** LLDB REPL backend (`pwndbg-lldb`) initialized pointing to the synced virtualenv.
*   **Challenges Faced & Workarounds:** Natively executing `pwndbg-lldb` on macOS Apple Silicon poses environment pathing and security/code-signing issues (causing commands to be missing or fail). To establish a reproducible environment, a Docker container mounting the repository (`ubuntu24.04-mount`) was used to run and test the debugger backend under a consistent Linux environment:
    ```bash
    docker compose run --rm -it ubuntu24.04-mount pwndbg-lldb /pwndbg/test_prog
    ```

### Steps to Reproduce

1. Compile a simple test program with debug symbols and start the debugger:
   ```bash
   echo 'int main() { return 0; }' > test.c
   gcc -g test.c -o test_prog
   pwndbg-lldb test_prog
   ```
2. Set a breakpoint at `main` and launch the process:
   ```text
   pwndbg-lldb> breakpoint set --name main
   pwndbg-lldb> process launch
   ```
3. Run the custom disassembly command:
   ```text
   pwndbg-lldb> nearpc
   ```
4. Immediately after execution, hit the `Enter` key on a blank prompt line.

### Reproduction Evidence

*   **Commit showing reproduction:** N/A (Reproduction performed locally via local development setup and virtual terminal test scripts).
*   **Screenshots/logs:** 
    Under the previous behavior, hitting `Enter` after invoking the `nearpc` (or `u`) command simply repeated the command but disassembled the exact same block of instructions at the same memory addresses:
    ```text
    pwndbg-lldb> nearpc
    ► 0x1024e4480 <main>       push   rbp
      0x1024e4481 <main+1>     mov    rbp, rsp
      0x1024e4484 <main+4>     xor    eax, eax
      0x1024e4486 <main+6>     pop    rbp
      0x1024e4487 <main+7>     ret    
    pwndbg-lldb> 
    ► 0x1024e4480 <main>       push   rbp
      0x1024e4481 <main+1>     mov    rbp, rsp
      0x1024e4484 <main+4>     xor    eax, eax
      0x1024e4486 <main+6>     pop    rbp
      0x1024e4487 <main+7>     ret    
    ```
    No address advancement occurred because LLDB repeated the command string exactly, and `LLDB.history()` returned an empty list (`[]`), meaning `check_repeated()` was unable to detect the repeat event and mark `repeat = True` (which triggers disassembly advancement).
*   **My findings:** 
    Discovered that command repeating on the LLDB backend was broken due to two factors:
    1. `LLDB.history()` was stubbed to return `[]`, preventing `check_repeated()` from ever returning `True`.
    2. LLDB requires python script-registered commands to explicitly implement a `get_repeat_command` method on the command class in order to support empty line (Enter key) auto-repeating.

---

## Solution Approach

### Analysis

The repeat-detection and address advancement logic was already implemented inside the core disassembly command (`nearpc` and `pwndbg.aglib.nearpc.nearpc`), which advances disassembly from `nearpc.next_pc` when `repeat=True` is set on the command. However, on LLDB, history was stubbed and there was no repeat handler hook, which meant `repeat` was always `False`.

### Proposed Solution

1. Maintain a custom command history (`_history_list`) and index (`_history_index`) inside the `LLDB` class and append to it in the interactive REPL loop (`repl/__init__.py`) when the user enters non-empty commands.
2. Return this custom history in `LLDB.history()` to enable `check_repeated()` to detect repeats by comparing history index numbers.
3. Implement `get_repeat_command()` on the dynamic `CommandHandler` class in `dbg_mod/lldb/__init__.py` to instruct LLDB to repeat the custom command when Enter is pressed.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Enable repeat-on-Enter command replication for custom commands under the LLDB backend so that `nearpc` (and its alias `u`) correctly advances disassembly on Enter.

**Match:** In GDB, repeating a command preserves the same command history index. I need to match this behavior in the LLDB REPL so that pressing Enter on a blank line runs the command without adding a new history entry, allowing `check_repeated()` to detect the identical index and signal a repeat.

**Plan:**
1. Initialize `self._history_list` and `self._history_index` inside `LLDB.setup()` in `pwndbg/dbg_mod/lldb/__init__.py`.
2. Modify the interactive command prompt logic in `pwndbg/dbg_mod/lldb/repl/__init__.py` to append user inputs to `_history_list` only when they are non-empty.
3. Implement `get_repeat_command(self, command)` on the dynamic `CommandHandler` class in `pwndbg/dbg_mod/lldb/__init__.py`.
4. Update `LLDB.history()` to return `self._history_list[-last:]`.

**Implement:** Committed the fixes under commit `4c4d338`.

**Review:** Verified history boundaries and list retrieval logic to prevent list index errors.

**Evaluate:** Verified that running `nearpc` and then hitting Enter correctly disassembles the next sequential block of instructions.

---

## Testing Strategy

### Unit Tests

- [x] Test `LLDB.history()` output under empty state (returns empty list).
- [x] Test `LLDB.history(last=n)` returns the correct tail of the custom history list.
- [x] Test `get_repeat_command(command)` returns command name and arguments string properly.

### Integration Tests

- [x] Verify `nearpc` correctly advances memory addresses on consecutive calls when `repeat=True`.
- [x] Verify `check_repeated()` returns `True` when repeating the last command on LLDB.

### Manual Testing

I verified the changes using an interactive test script `test_interactive_repeat.py`:
* Spawns `pwndbg-lldb` on `test_prog`.
* Runs `nearpc` first.
* Sends a blank line to repeat the command.
* Asserts that Output 2 starts at the instruction following Output 1:
  * Max address in Output 1: `0x1024e44b8`
  * Min address in Output 2 (repeat): `0x1024e44bc` (Success).

---

## Implementation Notes

### Week 1 Progress

*   Identified that LLDB's history function was stubbed and returned `[]`, which prevented `check_repeated()` from ever detecting repeats.
*   Discovered that interactive commands executed via the custom REPL loop were run programmatically (`SBCommandInterpreter.HandleCommand`) and not tracked in LLDB's native history buffer.
*   Researched the LLDB Python command API requirements for class-based repeatability.

### Week 2 Progress

*   Created an initial commit (`9c71696`) attempting a decorator-based approach, which was later determined to be incorrect/unnecessary.
*   Implemented the correct fix by initializing `_history_list` and `_history_index` inside the `LLDB` class, updating the custom REPL to populate it for non-empty commands, and implementing `get_repeat_command()` on dynamically registered commands.
*   Verified that the fix successfully enabled `nearpc`/`u` repeat-on-Enter range advancement in Docker (`ubuntu24.04-mount`).

### Week 3 Progress

*   Cleaned up the branch, overwriting the incorrect Week 2 commit (`9c71696`) with the final, polished implementation commit (`4c4d338`).
*   Created an automated interactive integration test script (`test_interactive_repeat.py`) to systematically verify behavior on step and repeat.
*   Preparing the contribution documentation and PR templates, awaiting final push and PR submission.

### Code Changes

*   **Files modified:**
    *   `pwndbg/dbg_mod/lldb/__init__.py`
    *   `pwndbg/dbg_mod/lldb/repl/__init__.py`
*   **Key commits:**
    *   [`4c4d338`](https://github.com/Bibhav48/pwndbg/commit/4c4d338c6b5ad5de05484d4d76db5bc37383ec46): `fix(lldb): implement history tracking and command repeat on LLDB (#1374)`
*   **Approach decisions:** 
    *   Instead of parsing native LLDB history (which is editline-dependent and does not have GDB-like history index logic), I managed the history list directly inside the REPL loop. This guarantees consistent history indices, allowing `check_repeated` to work cleanly.

---

## Pull Request / Commit Link

**Commit Link:** [`4c4d338`](https://github.com/Bibhav48/pwndbg/commit/4c4d338c6b5ad5de05484d4d76db5bc37383ec46)  
**Description:** This change implements repeat-on-Enter support for custom commands on the LLDB backend. It introduces custom interactive history tracking in the REPL and debugger backend, replaces the stubbed `LLDB.history()`, and adds standard LLDB `get_repeat_command` auto-repeat handler implementation. This enables `nearpc`/`u` (as well as other commands) to seamlessly auto-repeat and advance address references exactly like GDB.

**Maintainer Feedback:**
*   None yet.

**Status:** Awaiting submission

---

## Learnings & Reflections

### Technical Skills Gained

*   LLDB Python API structures (class-based commands, `get_repeat_command` interface).
*   Interactive REPL loop design and session management.
*   Debugger abstraction layer interactions between Aglib and the debugger backends.

### Challenges Overcome

*   Understanding that native LLDB does not automatically track programmatic commands run via `HandleCommand` in its history. Overcame this by implementing my own custom command registry in the interactive prompt loop.

### What I'd Do Differently Next Time

*   I would write the automated interactive tester script first to map the previous behavior of the REPL under fake inputs before coding the solution.

---

## Resources Used

*   [pwndbg/pwndbg GitHub Repository](https://github.com/pwndbg/pwndbg)
*   [LLDB Custom Commands Guide](https://lldb.llvm.org/use/python-reference.html)
*   [GDB Python API Official Documentation](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Python-API.html)
*   [CodePath AI301 Capstone Guide Portal](https://courses.codepath.org/courses/ai301)
