# soc
- project_include.cmake
	- Compiler Options:
		- -mlongcalls
		- -fno-builtin-memcpy
		- -fno-builtin-memset
		- -fno-builtin-bzero
	
	- Linker Options:
		- -nostartfiles

# xtensa
- Windowed ABI (default)
	- Include window overflow/underflow vector assembly routines in your startup image (`.iram0.vectors`), or your CPU will crash as soon as function call depth exceeds the register window.
	- **Strictly Required Files:** `xtensa_vectors.S` / `window_vectors.S`
	- **What you need:** The raw assembly routines that handle:
	    - `WindowOverflow4`, `WindowOverflow8`, `WindowOverflow12`
	    - `WindowUnderflow4`, `WindowUnderflow8`, `WindowUnderflow12`
	- _Alternative:_ If you compile with `-mabi=call0`, the compiler uses flat register conventions without register windowing, allowing you to bypass window exception handlers entirely.
		- If you add `-mabi=call0` to your compiler flags, GCC disables register windowing entirely and uses standard flat caller/callee-saved registers (like ARM Cortex-M or RISC-V), using normal `call0` and stack pushes.
		- ou can **completely omit** the complex window overflow/underflow vector assembly.
- CPU Exception Vectors & Registers
	- When your bare-metal code triggers a memory fault, bad pointer dereference, or illegal instruction, the hardware jumps to the **User Exception Vector** (`VECBASE + 0x50`).
	- The `xtensa` headers define the hardware exception registers you need to read inside your exception stub:
		- `EXCCAUSE`**EXCCAUSE**: Numeric reason for the crash (e.g., `28` = LoadProhibited, `29` = StoreProhibited, `0` = IllegalInstruction, `9` = LoadStoreAlignmentCause).
		- `EXCVADDR`**EXCVADDR**: The exact memory address that caused the fault.
		- `EPC1`**EPC1**: The Program Counter (instruction address) where the crash happened.
		- `xtensa/specreg.h`**xtensa/specreg.h**: Defines the symbolic names and register numbers for these special registers.
	- **Implement:** Minimal `.iram0.vectors` entry point to log `EXCCAUSE` and `EPC1` to UART upon crashes.

# esp_rom

