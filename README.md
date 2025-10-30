# Draft for JIT Plan

Designed for https://github.com/Uotan-Dev/uotan_riscv_emu

## uemu JIT (Just-In-Time) Compiler Implementation Plan

This plan outlines a dynamic translation (JIT) system to enhance `uemu`'s performance by translating "hot" RISC-V guest code into host machine code, while maintaining strict simulation accuracy for interrupts, memory management, and exceptions.

### I. Core Translator & Execution Model

The foundation of the JIT is a fast interpreter that identifies hot code and dispatches to translated blocks.

1.  **Basic Block (BB) as Translation Unit:** The JIT will not translate entire pages, but rather **Basic Blocks (BBs)**. A BB is a sequence of guest instructions starting at a known address and ending with the first control-flow-changing instruction (e.g., branch, `jal`, `jalr`, `ret`, `ecall`, `mret`, `sret`) or a potential exception.
2.  **Refactored Interpreter ("Slow Path"):** The main CPU loop (`cpu_exec_once`) will be replaced by a BB-based interpreter. Its core loop will:
    * Translate the current guest Virtual Address (`rv.PC`) to a Physical Address (PA).
    * Look up this PA in the JIT Translation Cache (TCG).
    * **Cache Hit:** Jump to the compiled host code. The host code will execute and return the next guest VA.
    * **Cache Miss:**
        * Increment the "hotness" counter for this PA.
        * If the counter exceeds a threshold (e.g., 64), submit this PA and the VA to the compilation thread (see Sec IV).
        * Interpret (decode and execute) instructions one by one until a BB-terminating instruction is hit.
        * Update `rv.PC` to the next block's address.
3.  **Guest State Access:** A pointer to the global `rv` (CPU state) struct will be passed to JIT code. For maximum performance, this pointer should be pinned to a dedicated host register (e.g., `r15` on x86-64), allowing JIT code to access guest registers (`rv.GPR`) and CSRs via fixed-offset addressing.
4.  **Helper Functions:** Complex or infrequent operations (CSR R/W, MMU lookups, `ecall`, `mret`, FPU exceptions) will not be translated directly. Instead, the JIT will emit a `call` to a C/C++ helper function.

### II. Translation Cache & Memory Management

This section details the cache that stores translated blocks and handles memory consistency.

1.  **Cache Key (Physical Address):** The translation cache (a hash map) will use the **Guest Physical Address (PA)** as its key. This is a robust design that naturally handles ASID/context switches. The map will store `Guest_PA -> Host_Code_Address`.
2.  **Cache Eviction:** The cache must have a fixed maximum size (e.g., 128MB). An **LRU (Least Recently Used)** policy will be used to evict old/cold blocks when the cache is full.
3.  **Self-Modifying Code (SMC) Handling:** This is critical for correctness.
    * A **reverse-mapping table** must be maintained, linking guest physical page frame numbers (PFNs) to a list of all JIT blocks that *start* in that page.
    * Any write to guest memory must be detected by the MMU/memory system.
    * Upon detecting a write to a physical page, the emulator must use the reverse map to find all JIT blocks in that page and **invalidate** them (i.e., remove them from the TCG hash map and free the host code memory).

### III. Precision: Interrupts, MMU, and RVC

This section ensures the JIT remains accurate.

1.  **Between-Block Interrupt Checks:** The main interpreter loop will check for pending interrupts *after* executing several BBs (whether interpreted or JIT-compiled) and *before* dispatching to the next BB.
2.  **Intra-Block Safepoints:** To handle long-running loops or large BBs, the JIT translator must inject **safepoints**. Every X (e.g., 65536) guest instructions, the JIT will emit host code to:
    * Increment a guest instruction counter (stored in the `rv` struct).
    * Check if interrupts are pending (e.g., `rv.MIP & rv.MIE`).
    * If an interrupt is pending, the JIT code will **exit** back to the interpreter, which will then handle the trap.
3.  **MMU Permission Checks:**
    * The interpreter must check the **execute permission** of a guest PA or VA *before* jumping to its compiled JIT code.
    * **RVC Cross-Page Handling:** The translator must be aware of instructions spanning page boundaries.

### IV. Threading Model: Hotspot Compilation

To avoid compilation pauses, translation will occur on a separate thread.

1.  **Main Thread:** This existing thread simulates devices and manages UI for the emulator.
2.  **Compilation Thread:** A new child thread will be created. It will sleep while waiting for compilation requests.
3.  **CPU emulation Thread:** This existing thread simulates guest instructions. It will also maintains a "hotness" counter for each guest PA. When a counter exceeds a threshold (e.g., 64), it pushes a "compile" job (containing the Guest PA) onto a thread-safe work queue.
4.  **Asynchronous Compilation:** The compilation thread wakes up, takes a job from the queue, translates the BB, and atomically updates the main translation cache (hash map) with the `Host_Code_Address`. This ensures the main thread sees the new code on its next lookup.

