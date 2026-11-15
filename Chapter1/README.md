# Chapter 1: Memory Management Fundamentals

## Core Fundamentals & Word Size
* **Word:** A `word` is a fixed-size basic unit of data used in a particular computer architecture. It represents the CPU's native chunk size—the natural width of data the processor was built to handle in a single operation.
  * In a 32-bit architecture: Word size is **32 bits (4 bytes)**.
  * In a 64-bit architecture: Word size is **64 bits (8 bytes)**.
  * The word size determines register width, pointer size (`IntPtr` / `nint`), memory bus transfer chunks, and data alignment boundaries.
* **Static Memory Allocation & Its Limitations:** The earliest programming languages (early FORTRAN, ALGOL) supported only static memory allocation, where memory sizes and locations had to be hardcoded at compile time. This led to severe drawbacks:
  * **The Over/Under-Allocation Dilemma:** Over-allocating wastes memory when workloads are small; under-allocating causes crashes when input sizes exceed compile-time limits.
  * **No Dynamic Data Structures:** Linked lists, trees, graphs, and resizable arrays (`List<T>`) cannot exist without dynamic allocation.
  * **No Recursion:** Functions sharing fixed static memory locations overwrite their own local variables and return addresses when calling themselves.

---

## 1. Stack Pointer (SP) and Program Counter (PC / IP) Workflow During Execution

### Execution Model and Downward Stack Growth
In modern architectures, memory addresses increase upwards, but the **call stack grows downward** (from high memory addresses to low memory addresses):
* **Allocating / Pushing onto the stack:** Decrements the Stack Pointer (`SP = SP - size`).
* **Discarding / Popping from the stack:** Increments the Stack Pointer (`SP = SP + size`).

```text
[ High Address: e.g., 0x1000 ]  ──► Initial SP
         │
         │  Stack grows DOWNWARD (allocation decrements SP)
         ▼
[ Low Address:  e.g., 0x0E00 ]  ──► Current SP
```

### The Function Call Workflow (`CALL` and `RET`)
1. **Before Function Call (`main` calls `fun1`):**
   * The Program Counter (`PC` / `RIP`) holds address `A1`, pointing to the `call fun1` instruction inside `main`.
   * `SP` points to the current boundary of `main`'s stack frame.
2. **Executing the Call (`CALL` instruction):**
   * The hardware `CALL` instruction performs two critical tasks atomically:
     1. **Pushes the Return Address (`A1 + 1`):** It calculates the address of the next instruction right after the call in `main` and pushes it onto the stack.
     2. **Updates the Program Counter:** It sets `PC = AddressOf(fun1)`.
   * The new function (`fun1`) establishes its **Activation Frame** (or Stack Frame) by decrementing `SP` to reserve space for:
     * Function arguments (if passed via stack).
     * Saved return address (`A1 + 1`).
     * Saved base pointer / frame pointer (if used).
     * Local variables.
3. **Nested Function Call (`fun1` calls `fun2`):**
   * Inside `fun1` at instruction `A2`, `fun1` calls `fun2`.
   * The CPU pushes return address `A2 + 1` onto the stack and updates `PC = AddressOf(fun2)`.
   * A new activation frame for `fun2` is created below `fun1`'s frame.
4. **Returning from Function (`RET` instruction):**
   * When `fun2` completes:
     1. Its activation frame is discarded simply by incrementing `SP` by the frame size.
     2. The hardware `RET` instruction pops the stored return address (`A2 + 1`) directly into the Program Counter: `PC = A2 + 1`.
     3. Execution instantly resumes at the next line of `fun1`.
   * When `fun1` completes, its `RET` pops `A1 + 1` into `PC`, returning execution seamlessly to `main`.

### Why the Return Address MUST Be Stored in the Activation Frame
* If the return address were stored in a single fixed CPU register or a static memory slot, a function could never call another function without overwriting its own return path.
* Specifically, **nested calls** (`main` $\to$ `fun1` $\to$ `fun2`) and **recursive calls** (`fun` $\to$ `fun` $\to$ `fun`) require a Last-In, First-Out (LIFO) structure.
* Storing the return address inside the per-call activation frame ensures every invocation preserves its own private return ticket, which is naturally unwound in reverse order.

---

## 2. Stack Machines vs. Register-Based Machines

### What is the Difference?
* **Stack Machine (e.g., .NET IL, Java Bytecode, UCSD Pascal p-code):**
  * Has no general-purpose registers. Operations implicitly pop their operands from an evaluation stack and push the result back onto the stack.
  * Example: `a + b` is executed as:
    ```text
    push a
    push b
    add      ; Pops a and b, adds them, pushes result
    pop c
    ```
* **Register Machine (e.g., x86/x64, ARM, RISC-V):**
  * Operates directly on a fixed set of fast, hardware-addressed registers.
  * Example: `a + b` is executed as:
    ```assembly
    mov eax, [a]
    add eax, [b]
    mov [c], eax
    ```

### Why Hardware Computers Do Not Operate on Stack Machines Today
While early hardware experiments used stack-based CPUs (e.g., Burroughs B5000), all modern general-purpose processors are **register-based**. The reasons are rooted in physics and microarchitecture:

1. **The "Top-of-Stack" Bottleneck:**
   * In a stack machine, every operation must read from and write to the very top of the stack. Even if the top elements are cached in hardware registers, all instructions serialize through that single bottleneck.
2. **Instruction-Level Parallelism (ILP) and Out-of-Order Execution:**
   * Modern CPUs have multiple ALUs and execute instructions concurrently or out of order.
   * In a register machine, independent operations can execute in parallel during the same clock cycle:
     ```assembly
     add rax, rbx   ; ALU 1 executes this
     mul rcx, rdx   ; ALU 2 executes this at the exact same time!
     ```
   * In a stack machine, because all operations mutate the same stack top, instructions have strict serial dependencies and cannot be easily parallelized by superscalar hardware.
3. **Register Reuse vs. Memory Shuffling:**
   * In a register machine, once a variable is loaded into `RAX`, it can be reused across 10 subsequent operations without touching RAM.
   * In a stack machine, reusing an operand requires constant `dup`, `swap`, and `rot` instructions, or spilling values to memory.

> **Why .NET IL is Still a Stack Machine:**
> Intermediate languages (.NET CIL, Java bytecode, WebAssembly) use a virtual stack machine because stack bytecode is **compact**, simple to generate by compilers (like Roslyn), and completely agnostic to how many physical registers a CPU actually possesses. At runtime, the .NET **JIT compiler** translates that virtual stack bytecode into native **x64/ARM register machine code**.

---

## 3. Why the "Mutator" Abstraction is Needed

In formal memory management literature (introduced by Dijkstra et al.), the program is conceptually split into two distinct entities:

```text
 ┌───────────────────────────────────────┐
 │               MUTATOR                 │  (Your application threads)
 │ • Executes business logic             │
 │ • Alters the object graph via:        │
 │     Mutator.New()                     │
 │     Mutator.Write()                   │
 └──────────────────┬────────────────────┘
                    │  (Concurrent modifications)
                    ▼
 ┌───────────────────────────────────────┐
 │              COLLECTOR                │  (The Garbage Collector)
 │ • Traces reachable objects            │
 │ • Reclaims unreferenced memory        │
 └───────────────────────────────────────┘
```

### Purposes of the Mutator Abstraction:
1. **Formal Separation of Concerns:**
   * From the GC’s perspective, application threads do only one thing: they continuously mutate the directed graph of memory references. Abstracting the application as a "Mutator" allows GC designers to prove the correctness and safety of collection algorithms independently of any specific programming language.
2. **Concurrency and Race-Condition Modeling:**
   * In modern concurrent/background GCs, the Collector is tracing the object graph *while* the Mutator is concurrently running and modifying pointers.
   * If the Mutator moves a reference from an unscanned object to an already-scanned object behind the Collector's back, the Collector might miss it and mistakenly collect live memory. The Mutator abstraction allows precise formalization of synchronization barriers (**Write Barriers** and **Read Barriers**) to intercept these mutations.
3. **Standardizing Memory Operations:**
   * It reduces application behavior to two canonical primitives:
     * `Mutator.New`: Allocates a node on the graph.
     * `Mutator.Write`: Modifies an edge on the graph (`a.field = b`).

---

## 4. How Conservative GC Works & The Ambiguous Pointer Dilemma

A **Conservative Garbage Collector** (e.g., the Boehm-Demers-Weiser Collector) operates in environments without compiler or runtime type metadata (such as unmanaged C/C++). 

Because it lacks type information, it cannot know whether an 8-byte chunk on the stack or in a register is a pointer or an integer. It must **guess**, erring on the side of caution (hence "conservative").

### Allocator Support: Segments of Equal-Size Objects
To make address decoding possible without type info:
1. The allocator divides the heap into pages/segments where **every object in a given segment has the exact same size** (e.g., all 64-byte objects in Segment A, all 128-byte objects in Segment B).
2. Given any candidate address $W$ landing in a segment starting at $\text{Base}$, the GC computes the exact object boundary via integer division and multiplication:
   $$\text{SlotIndex} = \frac{W - \text{Base}}{\text{ObjectSize}}$$
   $$\text{ObjectBaseAddress} = \text{Base} + (\text{SlotIndex} \times \text{ObjectSize})$$
This resolves interior pointers (pointers into the middle of an object) back to the object's start in $O(1)$ time.

### The 5-Step Address Filter Pipeline
When the Conservative GC scans every word $W$ on the stack and in registers:
1. **Alignment Check:** Is $W$ aligned to a word boundary (`W & 0x7 == 0`)? If not $\to$ Discard.
2. **Range Envelope Check:** Does $W$ fall between the global heap bounds ($\text{MinHeap} \le W < \text{MaxHeap}$)? If not $\to$ Discard.
3. **Page Table Lookup:** Does the page containing $W$ belong to this GC's allocated pool? If not $\to$ Discard.
4. **Allocation Bitmap Check:** Does the slot's allocation bit equal `1` (actively allocated vs. free space)? If free $\to$ Discard.
5. **Mark Bitmap Decision:** If active, mark the object as alive and push its base address to the mark stack.

---

### Practical Example: False Retention (Phantom Memory Leak)

Consider a 64-bit program running a Conservative GC with heap range `[0x00007FFF10000000 - 0x00007FFF20000000]`.

```c
// 1. Allocate a large 100 MB video buffer on the heap
//    The allocator places it at address: 0x00007FFF14A00000
byte* videoBuffer = (byte*)GC_malloc(100 * 1024 * 1024);

// 2. Program finishes processing video and discards reference
videoBuffer = NULL; 

// 3. Elsewhere on the stack, an innocent 64-bit integer calculation occurs:
uint64_t checkSum = 140733543153664; // In Hex: 0x00007FFF14A00000
```

#### Memory State at Time of Collection:
```text
========================================================================================
                      THREAD STACK (Starts at high memory: 0x00A0...)
========================================================================================
Stack Address          Variable Name      Type in Code    Raw 64-bit Value Stored Inside
────────────────────────────────────────────────────────────────────────────────────────
0x000000A01234F010     videoBuffer        byte*           0x0000000000000000 (NULL)
0x000000A01234F018     checkSum           uint64_t        0x00007FFF14A00000
                                                                  │
                                                                  │ (Coincidentally matches!)
                                                                  ▼
========================================================================================
                      MANAGED HEAP (Range: 0x00007FFF10000000 – 0x00007FFF20000000)
========================================================================================
Heap Address           Object Content
────────────────────────────────────────────────────────────────────────────────────────
0x00007FFF14A00000 ──► [ 100 MB Video Buffer Data ]  <-- SHOULD BE RECLAIMED AND FREED!
```

#### The GC's Decoding Process:
1. The GC scans stack slot `0x000000A01234F018` and reads the value $W = \mathbf{0x00007FFF14A00000}$.
2. **Filter 1 (Alignment):** `0x00007FFF14A00000 & 0x7 == 0` $\implies$ **Passed**.
3. **Filter 2 (Heap Range):** Falls inside `[0x00007FFF10000000 - 0x00007FFF20000000]` $\implies$ **Passed**.
4. **Filter 3 (Page Table):** Page descriptor confirms an allocated block exists here $\implies$ **Passed**.
5. **Filter 4 (Allocation Bitmap):** Slot is actively in use $\implies$ **Passed**.
6. **The Blind Choice:** The GC cannot tell whether the bits inside `checkSum` represent an integer or an address. Because discarding an active pointer causes catastrophic segmentation faults, the GC **conservatively assumes it is a pointer**.
7. **Result:** The GC sets the mark bit for the 100 MB buffer to `1`. The buffer is **retained in memory indefinitely**, causing a phantom memory leak.

> **Why .NET Avoids This:** .NET uses a **Precise (Exact) GC**. The JIT emits compiler stack maps for every instruction. When .NET inspects `0x000000A01234F018`, the stack map explicitly reports that the slot holds a scalar integer, so the GC completely ignores its bits.

---

## 5. Smart Pointers: `unique_ptr` vs. `shared_ptr`

Smart pointers implement automatic lifetime management in C++ at the library level using RAII (Resource Acquisition Is Initialization), constructors, destructors, and operator overloading.

### 1. `std::unique_ptr<T>` (Exclusive Ownership)
* **Semantics:** Realizes strict, exclusive ownership (1 object $\leftrightarrow$ 1 owner).
* **Copying is Forbidden:** Copy constructor and assignment operators are deleted (`= delete`). It cannot be copied:
  ```cpp
  std::unique_ptr<Dog> p1 = std::make_unique<Dog>("Rex");
  std::unique_ptr<Dog> p2 = p1;            // ❌ COMPILE ERROR!
  std::unique_ptr<Dog> p2 = std::move(p1); // ✅ Allowed: Ownership transferred; p1 becomes null.
  ```
* **Destruction Trigger:** The underlying object is deleted immediately when:
  1. The `unique_ptr` variable goes out of scope (at closing brace `}`).
  2. Another object is assigned to it (`p2 = std::make_unique<Dog>("Buddy");` deletes "Rex" first).
* **Overhead:** **Zero memory and zero CPU overhead.** A `unique_ptr` compiles down to the exact size and speed of a raw C pointer.

### 2. `std::shared_ptr<T>` (Reference Counting Ownership)
* **Semantics:** Realizes shared ownership where multiple pointers can own the same heap resource.
* **Mechanism:** Maintains a separate **Control Block** holding an atomic `ReferenceCounter`:
  * Every assignment or copy (`p2 = p1`) increments `ReferenceCounter`.
  * Every destruction or reassignment decrements `ReferenceCounter`.
  * When `ReferenceCounter == 0`, the owned object and the control block are immediately deleted.
* **Drawbacks:**
  * Memory overhead (control block allocation).
  * CPU overhead: Updates must use thread-safe atomic CPU instructions (`LOCK XADD` / `InterlockedIncrement`), which causes cache-coherency traffic across CPU cores.
  * Cyclic references (`A -> B` and `B -> A`) cause permanent memory leaks (requiring `std::weak_ptr` to break cycles).

---

### Why Library-Level Support is Advantageous Compared to Runtime Support

Kokosa emphasizes that Reference Counting and Smart Pointers can be implemented entirely in user-space libraries (like C++ `std::`) without runtime/VM support. This provides significant architectural benefits:

1. **No Virtual Machine or Heavy Runtime Engine Required:**
   * A Precise Tracing GC (like .NET CLR or JVM) requires an invasive, multimillion-line runtime engine capable of freezing threads, walking stacks, reading JIT tables, and handling GC safepoints.
   * Library-level smart pointers run on bare-metal architectures, microcontrollers, bootloaders, and OS kernels using standard C++ templates.
2. **Deterministic Destruction (Instant Cleanup):**
   * In a tracing GC, memory is freed at an arbitrary, non-deterministic time in the future during a collection cycle.
   * With library-level smart pointers, destruction happens **at the exact microsecond the variable exits scope**. This guarantees immediate release not just of RAM, but of OS handles (file descriptors, sockets, GPU buffers, database locks).
3. **Pay-As-You-Go Efficiency:**
   * You only pay the cost of reference counting when you explicitly choose `shared_ptr`.
   * If exclusive ownership suffices, you use `unique_ptr` with zero runtime overhead.
   * In contrast, a managed runtime imposes an immutable metadata tax (16-byte object headers, JIT stack maps, safepoints) on *every single object* created in the system.
