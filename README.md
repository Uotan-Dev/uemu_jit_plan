# Draft for JIT Plan

Designed for https://github.com/Uotan-Dev/uotan_riscv_emu

## uemu JIT (Just-In-Time) Compiler Implementation Plan

This plan outlines a dynamic translation (JIT) system for `uemu` to enhance performance by translating "hot" RISC-V guest code into host machine code. It incorporates an asynchronous compilation model with robust handling for interrupts, memory management, and self-modifying code (SMC).

### License
Copyright 2025 Nuo Shen, Nanjing University  
Copyright 2025 UOTAN  

Licensed under the Apache License, Version 2.0 (the "License");  
you may not use this file except in compliance with the License.  
You may obtain a copy of the License at  

    http://www.apache.org/licenses/LICENSE-2.0  

Unless required by applicable law or agreed to in writing, software  
distributed under the License is distributed on an "AS IS" BASIS,  
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  
See the License for the specific language governing permissions and  
limitations under the License.

### I. Core Translator & Execution Model

The foundation is a Basic Block (BB) interpreter that identifies hot code, dispatches to a separate compilation thread, and executes translated code.

1.  **Basic Block (BB) as Translation Unit:** The JIT will translate **Basic Blocks**. A BB is a sequence of guest instructions starting at a specific Physical Address (PA) and ending with the first control-flow-changing instruction (e.g., branch, `jalr`, `ecall`, `mret`) or a potential exception.
2.  **Refactored CPU Thread ("Slow Path"):** The main CPU loop (`cpu_exec_once`) will be replaced by a BB-based interpreter loop:
    * Get current guest Virtual Address (VA) from `rv.PC`.
    * Perform **MMU translation** (`mmu_translate(VA, EXEC_PERM)`) to get the guest Physical Address (PA) and check execution permissions. If this fails, raise a guest page fault exception.
    * Look up this `PA` in the JIT Translation Cache (TCG).
    * **Cache Hit:**
        * Jump to the compiled host code.
        * The host code executes and returns the next guest PA.
        * The loop continues from this new PA (after an interrupt check).
    * **Cache Miss:**
        * Increment the "hotness" counter for this `PA`.
        * If the counter exceeds a threshold (e.g., 64), **submit this `PA`** to the Compilation Thread's work queue.
        * **Interpret** (decode/execute) the BB instruction by instruction until it terminates.
        * Update `rv.PC` to the next guest VA.
    * **Interrupt Check:** After every BB execution (JIT or interpreted), check for and handle pending interrupts (see Sec III).
3.  **Guest State Access:** A pointer to the global `rv` (CPU state) struct will be pinned to a dedicated host register (e.g., `r15` on x86-64) for high-speed access to GPRs and CSRs from JIT-compiled code.
4.  **Helper Functions:** Complex or infrequent operations (CSR R/W, MMU page table walks, `ecall`, `mret`, `sret`, FPU exceptions) will be implemented as C/C++ helper functions. JIT code will emit `call` instructions to these helpers.

### II. Translation Cache & Memory Management (SMC)

This section details the cache and the critical mechanism for handling self-modifying code (SMC) safely with an asynchronous compiler.

1.  **Cache Storage (TCG):** A hash map will store translations: `Key: Guest_PA -> Value: Host_Code_Address`. A fixed-size buffer (e.g., 128MB) will store the generated host code.
2.  **Cache Eviction:** An **LRU (Least Recently Used)** policy will evict old blocks when the cache is full.
3.  **Page Versioning for SMC:** To handle SMC and the compiler race condition:
    * A global `uint64_t *page_versions` array (one entry per guest physical page, initialized to 0) will be maintained.
    * A single global mutex (`tcg_cache_lock`) will protect the TCG, the `page_versions` array, and the reverse mapping.
4.  **SMC Handling (CPU Thread):** When a write is detected on `PA`:
    * Acquire the `tcg_cache_lock`.
    * Perform the write to guest RAM.
    * **Increment the page version:** `page_versions[PFN(PA)]++`.
    * **Invalidate:** Use the reverse-map to find and remove all JIT blocks that originate from this `PFN`.
    * **Reset hotness counters:** Set all hotness counters for invalidated blocks to 0.
    * Release the `tcg_cache_lock`.
5.  **Reverse Mapping:** A `PFN -> [list of JIT blocks]` table is required for efficient invalidation (as described in II.4).

### III. Precision: Interrupts, MMU, and RVC

1.  **Precise Interrupt Checks:** Pending interrupts will be checked **after every Basic Block execution** (in the interpreter loop) to ensure low latency.
2.  **Intra-Block Safepoints:** The JIT translator *must* inject **safepoints** into large BBs (e.g., loops). Every X (e.g., 1024) guest instructions, the JIT code will:
    * Emit a call to a `helper_check_interrupts` function.
    * This helper will check for pending interrupts (e.g., `rv.MIP & rv.MIE`) and, if any are found, **exit** back to the interpreter to handle the trap.
3.  **MMU Permission Checks:** As described in I.2, execute permissions are checked by the interpreter *before* every BB lookup/execution using the **guest VA**.
4.  **RVC Cross-Page Handling:** The translator must be page-aware. If an instruction at `PA` is 32-bits and `PA+2` is on a different physical page, the translator must snapshot version numbers for *both* pages (see IV.4) and the BB must terminate if `PA+2` is not executable.

### IV. Threading Model: Asynchronous Hotspot Compilation

This model prevents compilation from stalling the CPU.

1.  **Main Thread (Devices/UI):** Manages I/O, devices, and the UI.
2.  **CPU Emulation Thread:** Runs the core interpreter loop (Sec I.2), executes all code (cold and hot), and identifies hot blocks.
3.  **Compilation Thread (JIT):** A new child thread dedicated to compilation. It sleeps until work is available in a thread-safe **compile queue**.
4.  **Asynchronous Compilation & Race-Condition Handling:**
    * **CPU Thread:** When `PA` hotness > 64, pushes `PA` to the compile queue.
    * **JIT Thread:**
        1.  Wakes up, pops `PA`.
        2.  **Snapshot:** Acquires `tcg_cache_lock`. Reads and stores the current `page_versions` for all pages the BB will touch. Reads the *entire* BB from guest RAM into a local buffer. Releases `tcg_cache_lock`.
        3.  **Compile (Slow):** Compiles the BB from its **local buffer** into a temporary host code buffer. This happens *without* holding any locks.
        4.  **Commit/Verify (Fast):**
            * Acquires `tcg_cache_lock`.
            * **Verification 1:** Re-checks the `page_versions` against the snapshot. If any have changed, **abort compilation** (a write occurred).
            * **Verification 2:** Check if another thread compiled this block in the meantime (`tcg_cache.lookup(PA)`). If so, **abort**.
            * **Install:** If verified, move the compiled code to its final location, update the TCG hash map, and update the reverse-map.
            * Releases `tcg_cache_lock`.

### V. CPU Thread Execution Flow & Branch Handling

This section details the interpreter's operational model, control flow handling, and critical edge cases.

#### A. Interpreter Execution Model

1.  **Instruction-by-Instruction Interpretation:**
    * When a BB cache miss occurs, the interpreter enters **interpretation mode** for that BB.
    * It decodes and executes instructions **one at a time**, starting from the current `PA`.
    * After each instruction execution, `rv.PC` is updated to point to the next instruction's VA.
    * The BB terminates when:
        * A control-flow instruction is executed (branch, jump, `ecall`, `mret`, etc.).
        * An exception occurs (illegal instruction, misaligned access, page fault).
        * The instruction sequence crosses to a new physical page (requires new MMU translation and permission check).
2.  **BB Boundary Detection:**
    * After executing each instruction, check if:
        * The next `PA` would cross a physical page boundary (compare `PFN(current_PA)` with `PFN(next_PA)`).
        * A control-flow change occurred.
    * If either condition is true, **terminate the current BB** and return to the main interpreter loop (which will perform MMU translation, permission checks, and cache lookup for the next BB).

#### B. Control Flow Handling

**1. Direct/Static Branches (beq, bne, blt, etc.):**

* **Interpreted Execution:**
    * Evaluate the branch condition.
    * Update `rv.PC` to either the branch target (if taken) or `PC + 4` or `PC + 2`.
    * Terminate the BB and return control to the interpreter loop.
* **JIT Compilation:**
    * The JIT must generate code for **both paths** (taken and not-taken).
    * For tight loops (e.g., `loop: ... beq x1, x2, loop`), special handling is required:
        * **Backward branches within the same page:** The JIT can emit an internal loop in host code. However, it **must** inject a safepoint check (call to `helper_check_interrupts`) inside the loop to prevent infinite loops from blocking interrupts.
        * **Example:** For `while(1);` or similar infinite loops, the JIT-compiled code must call `helper_check_interrupts` periodically (e.g., every iteration or every N iterations). This helper checks for pending interrupts and returns to the interpreter if any are found.
    * **Branch chaining optimization:** If the branch target is already compiled (exists in TCG) and **is in the same page**, the JIT can emit a direct jump to the translated target. Otherwise, it returns the target PA to the interpreter.
    * **Cross-page branches:** Always terminate the BB and return the target PA to the interpreter (which will perform permission checks and cache lookup).

**2. Indirect/Dynamic Jumps (jalr, ret):**

* **Interpreted Execution:**
    * Compute the target address from the register value.
    * Update `rv.PC` to the computed target VA.
    * Terminate the BB and return to the interpreter loop.
* **JIT Compilation:**
    * The target cannot be determined at compile time.
    * JIT code must:
        1.  Compute the target VA (read from register, add offset).
        2.  **Return the target VA to the interpreter** (cannot chain directly).
        3.  The interpreter will perform MMU translation, permission check, and TCG lookup for the new target.

#### C. Self-Modifying Code (SMC) Edge Cases

**1. Same-Page Self-Modification:**

* **Problem:** A BB writes to memory on the same physical page it's executing from (e.g., unpacking/decryption code, JIT compilers in the guest).
* **Detection:**
    * Maintain a per-page **execution state** flag in the `page_versions` array or a separate `page_state` array:
        * `STATE_NORMAL`: Normal execution (JIT-eligible).
        * `STATE_SMC_DETECTED`: SMC detected on this page; must interpret only.
    * When a store instruction writes to a page with `PFN(store_PA) == PFN(current_PC_PA)`:
        * Set `page_state[PFN] = STATE_SMC_DETECTED`.
        * Invalidate all JIT cache entries for this PFN (as in II.4).
        * Set all hotness counters for this page to 0.
* **Interpreter Behavior:**
    * In the main loop, after MMU translation, check `page_state[PFN(PA)]`.
    * If `STATE_SMC_DETECTED`, **force interpretation** (skip TCG lookup, do not accumulate hotness, do not submit for compilation).
    * Continue interpreting instruction-by-instruction.
* **State Reset:**
    * Reset `page_state[PFN]` to `STATE_NORMAL` when:
        * Execution leaves the page (detected when the next instruction's PFN differs).
    * This allows JIT compilation to resume once the SMC region is no longer being executed.

**2. Compilation Cancellation on Concurrent Write:**

* **Problem:** The CPU thread writes to a page while the JIT thread is compiling a BB from that page.
* **Solution:**
    * The JIT thread registers the PFNs it's compiling in `active_compilations`.
    * The CPU thread's SMC handler (II.4) checks `active_compilations` before invalidating:
        * If the PFN is active, set a cancellation flag for that task and **skip the write to `page_versions`** temporarily (or use a separate `pending_invalidation` flag).
        * The JIT thread checks its cancellation flag periodically (or at commit time) and aborts if set.
        * After abort, the JIT thread clears its entry from `active_compilations`.
        * The CPU thread can then proceed with invalidation on its next SMC event (or via a deferred invalidation queue).

#### D. Hotness Counter Management

1.  **Counter Increment:** Increment `hotness[PA]` only when interpreting (cache miss). Do not increment on JIT cache hits.
2.  **Counter Reset on Invalidation:** When invalidating JIT blocks for a PFN (due to SMC), reset `hotness[PA]` to 0 for all PAs in that PFN. This prevents stale hot blocks from being immediately recompiled if the code has changed.
3.  **Threshold Tuning:** The threshold (e.g., 64) should be tuned based on profiling. Too low: wastes compilation time on cold code. Too high: misses optimization opportunities.

#### E. JIT Code Generation for Branches

1.  **Forward Branches (within BB):**
    * Generate conditional host code that mirrors the guest branch logic.
    * Both paths continue execution within the JIT-compiled block (if they remain in the same BB).
2.  **Backward Branches (loops):**
    * Emit a host-level loop for same-page backward branches.
    * **Critical:** Inject `call helper_check_interrupts` at the loop header or every N iterations.
    * If the helper detects a pending interrupt, it must:
        * Update `rv.PC` to the current instruction's VA.
        * Return to the interpreter (by returning a special sentinel value or setting a flag).
3.  **Block Chaining:**
    * For direct branches to already-compiled targets:
        * Look up the target PA in the TCG.
        * If found, emit a direct `jmp` to the host code.
        * If not found, return the target PA to the interpreter.
    * For indirect jumps (`jalr`), always return to the interpreter.
4.  **Exit Points:**
    * Every JIT block must have well-defined exit points:
        * **Return PA:** The next guest PA to execute (used by the interpreter).
        * **Exception flag:** Indicate if an exception occurred (page fault, illegal instruction, etc.).
