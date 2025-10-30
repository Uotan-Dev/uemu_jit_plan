# Draft for JIT Plan

Designed for https://github.com/Uotan-Dev/uotan_riscv_emu

## uemu JIT (Just-In-Time) Compiler Implementation Plan

This plan outlines a dynamic translation (JIT) system for `uemu` to enhance performance by translating "hot" RISC-V guest code into host machine code. It incorporates an asynchronous compilation model with robust handling for interrupts, memory management, and self-modifying code (SMC).

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
