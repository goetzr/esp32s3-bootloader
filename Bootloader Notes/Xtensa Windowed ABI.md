The **Xtensa Windowed Register ABI** is a hardware-assisted calling convention designed by Tensilica to drastically reduce the execution overhead of function calls.

In traditional architectures (like ARM Cortex-M or x86), entering and exiting functions requires explicit instructions to push registers to the stack (saving state) and pop them back on return (restoring state). The Xtensa Windowed ABI eliminates most of this software overhead by using a **large physical register file** and a **sliding logical window** managed by hardware.

## 1. The Core Concept: Logical Window vs. Physical File

The key distinction in the Xtensa architecture is between what the compiler/instructions see and what physically exists in the silicon:

- **Logical Registers (a0 – a15):**`a0``a15` At any given moment, the instruction set and compiler can only name 16 general-purpose registers: `a0` through `a15`. This is the **Register Window**.
    
- **Physical Register File (AR0 – AR63):**`AR0``AR63` The ESP32 / ESP32-S3 implementations of the Xtensa LX6/LX7 cores feature a circular buffer of **64 physical registers** (Address Registers, or `AR`).
    

When a function is called, the hardware physically shifts the base of the logical window into the physical register file rather than pushing registers onto the stack in SRAM.

```
Logical Registers:   [ a0 | a1 | a2 | a3 | a4 | a5 | a6 | a7 | ... | a15 ]
                           \    \    \    \
                            \    \    \    \   (Sliding Window)
                             ▼    ▼    ▼    ▼
Physical Registers: [ AR0 | AR1 | AR2 | AR3 | AR4 | AR5 | ... | AR63 ] (Wraps circularly)
```

## 2. How the Window Slides: Special Registers

The CPU tracks the location of active windows using three dedicated Special Registers:

1. `WINDOWBASE`**WINDOWBASE:** An internal pointer/index into the physical register file indicating where logical register `a0` currently maps.
    
2. `WINDOWSTART`**WINDOWSTART:** A 64-bit (or 16/32-bit depending on core config) bitmask where each bit corresponds to a physical register slot. A bit is set to `1` if that physical slot is the start of an active function's window (`WINDOWBASE`).
    
3. `PS`**PS (Processor State):** Contains control bits, including user privilege and interrupt masks.
    

## 3. The Function Call Mechanism: `ENTRY`, `CALL`, and `RETW`

Calls are parameterized by **how many registers the caller rotates the window forward**: 4, 8, or 12.

### Step 1: The Caller (`CALL4`, `CALL8`, or `CALL12`)

The caller places arguments into its own output registers and executes a windowed call instruction:

- `CALL4 target` \rightarrow Rotates the window forward by 4 physical registers.
    
- `CALL8 target` \rightarrow Rotates the window forward by 8 physical registers (the standard default in GCC).
    
- `CALL12 target` \rightarrow Rotates the window forward by 12 physical registers (used for functions with many parameters).
    

When `CALL8` executes:

- The caller's `a8` becomes the callee's `a0`.
    
- The caller's `a9`–`a11` become the callee's `a1`–`a3`.
    
- The caller's `a12`–`a15` become the callee's `a4`–`a7` (**Function arguments arg0 through arg3**`arg0``arg3`).
    

### Step 2: The Callee (`ENTRY`)

Every windowed function must begin with an `ENTRY` instruction:

```assembly
my_function:
    entry a1, 32    # Reserves 32 bytes of local stack frame and commits the window
```

- The `ENTRY` instruction tells the hardware to update `WINDOWBASE` and allocate the requested stack space (`a1` is always the Stack Pointer).
    
- If moving the window forward would overwrite older, un-saved caller windows in the physical register ring, the `ENTRY` instruction triggers a **Hardware Window Overflow Exception**.
    

### Step 3: Returning (`RETW`)

When the function finishes:

```assembly
    retw            # Windowed return
```

- `RETW` reads the return address stored in `a0` (which includes encoded information about how large the caller's call rotation was).
    
- The hardware automatically rotates `WINDOWBASE` backward by 4, 8, or 12 slots, restoring the caller's exact register context.
    
- If the caller’s context was previously spilled to memory, `RETW` triggers a **Hardware Window Underflow Exception**.
    

## 4. Parameter & Return Passing Layout (Standard `CALL8`)

Under the standard `-mabi=windowed` compiler configuration using `CALL8`:

|Logical Register|Role from Callee's Perspective|Role from Caller's Perspective|
|---|---|---|
|`a0`**a0**|**Return PC** (and call width encoding)|Caller's `a8`|
|`a1`**a1**|**Stack Pointer (SP)**|Caller's `a9`|
|`a2`**a2**|**Argument 0 / Return Value**|Caller's `a10` (receives return value)|
|`a3`**a3**|**Argument 1**|Caller's `a11`|
|`a4`**a4**|**Argument 2**|Caller's `a12`|
|`a5`**a5**|**Argument 3**|Caller's `a13`|
|`a6`**a6**|**Argument 4**|Caller's `a14`|
|`a7`**a7**|**Argument 5**|Caller's `a15`|
|`a8`**a8 – a15**`a15`|Temporary / Scratch / Outgoing args|N/A|

Because `a2` in the callee maps directly to `a10` in the caller, returning a value requires zero memory operations—the callee places the result in `a2`, and upon `RETW`, it is already sitting in the caller's `a10`.

## 5. Spilling and Restoring: Overflows & Underflows

Because the physical register ring is finite (64 registers), deeply nested function calls or recursion will eventually wrap around:

```
[ Active Frames: Frame 1 -> Frame 2 -> Frame 3 -> Frame 4 ] -> Wraps to Frame 1!
```

When an `ENTRY` instruction detects that advancing `WINDOWBASE` would overwrite a register slot whose `WINDOWSTART` bit is still set:

1. **Window Overflow Exception:** The CPU pauses execution and hardware branches directly to the **Window Overflow Vector** (e.g., `WindowOverflow8` at fixed offset `VECBASE + 0x00`).
    
2. **Spill Handler:** A fast, hand-tuned assembly routine (typically 4–8 instructions) writes the oldest frame's physical registers out into its allocated stack space in RAM.
    
3. The hardware clears that slot's `WINDOWSTART` bit and resumes the `ENTRY` instruction.
    

When a function executes `RETW` and discovers the caller’s registers were spilled to RAM:

1. **Window Underflow Exception:** The CPU branches to the **Window Underflow Vector** (`WindowUnderflow8`).
    
2. **Restore Handler:** Reloads the caller’s saved registers from the stack back into physical AR slots.
    
3. Resumes execution in the caller.
    

## 6. Windowed ABI vs. Call0 ABI

Xtensa toolchains allow choosing between the Windowed ABI and the flat **Call0 ABI**:

|Feature|Windowed ABI (`-mabi=windowed`)|Call0 ABI (`-mabi=call0`)|
|---|---|---|
|**Call Instruction**|`call4`, `call8`, `call12`|`call0`|
|**Register Mechanism**|Sliding window over 64 physical AR registers|Flat 16 registers (`a0`–`a15`), standard caller/callee-saved|
|**Hardware Vectors**|**Requires** `WindowOverflow` & `WindowUnderflow` handlers|**No window vectors needed**|
|**Call/Return Speed**|Extremely fast (zero memory traffic for shallow call trees)|Standard stack push/pop overhead|
|**Code Size**|Smaller per function (no prologue/epilogue register pushes)|Slightly larger function prologues|
|**Bare-Metal Complexity**|Higher (must provide low-level window vector assembly)|Minimal (works like ARM Cortex-M / RISC-V)|