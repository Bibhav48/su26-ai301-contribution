# Contribution 1: Add command repeat support for u command

**Contribution Number:** 1  
**Student:** Bibhav Adhikari  
**Issue:** [pwndbg/pwndbg #1374](https://github.com/pwndbg/pwndbg/issues/1374)  
**Status:** Phase II Complete

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
* Operating System: macOS Sequoia (15.x) running on arm64 Apple Silicon.
* Environment: Local `pwndbg` development environment synced using Python `uv sync --all-groups --all-extras` within `.venv`.
* Debugger Interface: LLDB REPL backend (`pwndbg-lldb`) initialized pointing to the synced virtualenv.

### Steps to Replicate (Symmetric for Previous vs. Current Behavior)
1. Start a debug session pointing `pwndbg-lldb` to a compiled target binary:
   ```bash
   pwndbg-lldb /Users/bibhav/.gemini/antigravity-ide/scratch/test_prog
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

---

### Previous Behavior (Stubbed history / no auto-repeat handler)
*   **Result:** LLDB does not recognize `nearpc` as a repeatable Python command and falls back to repeating the last native debugger command (`process launch`).
*   **Observed Output:**
    ```text
    pwndbg-lldb> 
    error: a process is already being debugged
    pwndbg-lldb>
    ```

---

### Current Behavior (Implemented history / with auto-repeat handler)
*   **Result:** LLDB calls `get_repeat_command()` to repeat `nearpc`, and `CommandObj` detects the enter press via `LLDB.history()`. This sets the `repeat=True` flag, successfully disassembling the next block of instructions.
*   **Observed Output:**
    ```text
    pwndbg-lldb> 
    WARNING: your terminal doesn't support cursor position requests (CPR).
       0x1000004bc <printf+8>    br     x16
       0x1000004c0 ◂— adr    x8, 0x1000d916b
       ... (next sequential disassembly block) ...
    pwndbg-lldb> 
    ```

---

## Solution Approach

### Analysis
The underlying repeat-detection and address advancement logic was already implemented inside the core disassembly command (`nearpc` and its helper `pwndbg.aglib.nearpc.nearpc`). When `repeat=True` is passed (propagated from `CommandObj.repeat`), the disassembly engine automatically reads the cached last disassembled address `nearpc.next_pc` instead of the current `$pc`. 

However, on the LLDB backend, command repeating did not work for two reasons:
1. **Stubbed History:** `LLDB.history()` in [pwndbg/dbg_mod/lldb/__init__.py](file:///Users/bibhav/Projects/pwndbg/pwndbg/dbg_mod/lldb/__init__.py) was stubbed to return `[]`. Because of this, `check_repeated()` inside the core `CommandObj` always returned `False`, and `CommandObj.repeat` was never set.
2. **Missing Auto-Repeat Handler:** LLDB requires python script-registered commands to explicitly implement a `get_repeat_command` method on the command class in order to support empty line (Enter key) auto-repeating. Without this, pressing Enter on an empty line repeated the last native LLDB command (like `process launch`), instead of repeating the custom python command.

### Proposed Solution
1. **Implement custom history tracking:** Since commands executed programmatically via our custom LLDB REPL (`repl/__init__.py`) are not added to LLDB's native command history, we maintain a custom history list (`_history_list`) and index (`_history_index`) inside the `LLDB` class itself. We append to this history list whenever a non-empty interactive command is entered by the user in the REPL loop, and have `LLDB.history()` return this list.
2. **Implement `get_repeat_command()`:** Implement `get_repeat_command(self, command)` on the dynamic `CommandHandler` class in [pwndbg/dbg_mod/lldb/__init__.py](file:///Users/bibhav/Projects/pwndbg/pwndbg/dbg_mod/lldb/__init__.py) to return the command name and arguments string, instructing LLDB to repeat the custom command on an empty line.

---

## Testing Strategy

## Automated Verification
I created a robust automated integration script `test_interactive_repeat.py` under the scratch directory:
- Spawns `pwndbg-lldb` on a compiled ARM64 target binary.
- Sets a breakpoint at `main` and runs the process.
- Executes `nearpc` once and parses the output instruction addresses.
- Presses `Enter` (sends a blank line) to invoke the repeat.
- Parses the repeated disassembly instruction addresses.
- Asserts that the disassembly range advanced sequentially (the minimum address in the second disassembly block is greater than or equal to the maximum address in the first block).

### Test Results
Executing `test_interactive_repeat.py` produces a successful pass:
```text
Launching pwndbg-lldb...
pwndbg-lldb> target create '/Users/bibhav/.gemini/antigravity-ide/scratch/test_prog'
[+] Setting breakpoint...
breakpoint set --name main
Breakpoint 1: where = test_prog`main + 24 at test.c:8:5, address = 0x00000001024e4494
pwndbg-lldb> 
[+] Launching process...
process launch
...
[+] Sending first 'nearpc' command...
nearpc
   0x1024e4480 <main+4>      stp    x29, x30, [sp, #0x10]
   0x1024e4484 <main+8>      add    x29, sp, #0x10
   ...
   0x1024e44b8 <printf+4>    ldr    x16, [x16]
pwndbg-lldb> 
[+] Sending empty line (Enter) to repeat...
   0x1024e44bc <printf+8>    br     x16
   0x1024e44c0               adr    x8, 0x1025bd16b
pwndbg-lldb> 

=================== OUTPUT 1 ===================
 nearpc
   0x1024e4480 <main+4>      stp    x29, x30, [sp, #0x10]
   0x1024e4484 <main+8>      add    x29, sp, #0x10
   ...
   0x1024e44b8 <printf+4>    ldr    x16, [x16]

=================== OUTPUT 2 ===================
   0x1024e44bc <printf+8>    br     x16
   0x1024e44c0               adr    x8, 0x1025bd16b

Instruction addresses in output 1: ['0x1024e4480', ..., '0x1024e44b8']
Instruction addresses in output 2: ['0x1024e44bc', '0x1024e44c0']
Max address in first run: 0x1024e44b8
Min address in repeat run: 0x1024e44bc

SUCCESS: Enter-repeat actually worked and advanced disassembly!
```

---

## Implementation Notes

### Progress Update
1. **Identified Root Cause:** Traced the issue to LLDB's stubbed `history()` method, lack of `get_repeat_command()` on dynamic python commands, and the fact that REPL-executed commands are not added to LLDB's native history.
2. **Implemented Fixes:**
   - Initialized and maintained a custom interactive command history list and index inside `LLDB` class and `repl/__init__.py`.
   - Replaced stubbed `LLDB.history()` to return the custom history.
   - Added `get_repeat_command()` to dynamically registered command classes.
3. **Verified Success:** Verified that repeating `nearpc` or `u` by hitting Enter interactively now repeats and advances the memory range correctly on LLDB (matching GDB's behavior).

### Code Changes
- **Files modified:**
  - [pwndbg/dbg_mod/lldb/__init__.py](file:///Users/bibhav/Projects/pwndbg/pwndbg/dbg_mod/lldb/__init__.py)
  - [pwndbg/dbg_mod/lldb/repl/__init__.py](file:///Users/bibhav/Projects/pwndbg/pwndbg/dbg_mod/lldb/repl/__init__.py)
- **Commits:**
  - `fix(lldb): implement history() to enable command repeat on LLDB (#1374)`

---

## Pull Request
**PR Link:** [GitHub PR URL when submitted]  
**PR Description:** This PR implements repeat-on-Enter support for custom commands on the LLDB backend. It introduces custom interactive history tracking in the REPL and debugger backend, replaces the stubbed `LLDB.history()`, and adds standard LLDB `get_repeat_command` auto-repeat handler implementation. This enables `nearpc`/`u` (as well as other commands) to seamlessly auto-repeat and advance address references exactly like GDB.

**Maintainer Feedback:**
- None yet

**Status:** Ready to Submit

---

## Learnings & Reflections

### Technical Skills Gained
- LLDB Python Command API (class-based commands, `get_repeat_command` interface).
- Debugger engine abstraction layers in `pwndbg`.
- Interactive debugger test scripting using `pexpect` under virtual terminal environments.

---

## Resources Used
- [pwndbg/pwndbg GitHub Repository](https://github.com/pwndbg/pwndbg)
- [LLDB Custom Commands Guide](https://lldb.llvm.org/use/python-reference.html)
- [GDB Python API Official Documentation](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Python-API.html)
- [CodePath AI301 Capstone Guide Portal](https://courses.codepath.org/courses/ai301)
