# Is there any code in the first stage boot loader of the esp32-s3 that the second stage boot loader, application code, or indirectly through the C/C++ standard library calls?

**Yes, absolutely.** The first-stage bootloader is not just a transient sequence of startup instructions—it is part of the **ESP32-S3 ROM** (0x40000000–0x4005FFFF), which remains permanently mapped and executable at runtime.

Both the second-stage bootloader and application binaries (including the C/C++ runtime) frequently call routines residing inside the ROM to save SRAM/Flash space and access low-level hardware routines.

Here is a breakdown of the ROM code and functions commonly called by downstream code:

### 1. Standard C Runtime & Utility Routines (`libc`)

The ESP32-S3 ROM contains a pre-compiled version of standard C library functions (`newlib` subset). Toolchains provide **ROM linker scripts** (e.g., `esp32s3.rom.ld` / `esp32s3.rom.libgloss.ld`) that alias standard C functions directly to ROM addresses rather than compiling them into your binary:

- **Memory Functions:** `memcpy`, `memset`, `memmove`, `memcmp`, `bzero`
    
- **String Operations:** `strlen`, `strcpy`, `strncpy`, `strcmp`, `strncmp`, `strstr`, `strchr`
    
- **Formatted I/O:** `printf`, `sprintf`, `snprintf`, `puts` (routed to the ROM console UART)
    
- **Standard Math/Conversion:** `atoi`, `strtol`, `rand`, `srand`, basic float arithmetic helpers
    

### 2. Low-Level Hardware & "ETS" Functions

The ROM exposes essential hardware drivers prefixed with `ets_` (Espressif Tool System) or `rom_`:

- `ets_printf()`**ets_printf():** A lightweight, unbuffered console print function that outputs directly to the default UART without needing full stdio buffering setup.
    
- `ets_delay_us()`**ets_delay_us():** Accurate hardware-cycle delay loops calibrated to CPU clock frequencies.
    
- `ets_install_uart_printf()`**ets_install_uart_printf() / ets_install_putc1():**`ets_install_putc1()` Hooks to redirect early console output to UART0, UART1, or USB-Serial/JTAG.
    
- **Interrupt Matrix Functions:** Basic interrupt configuration and CPU interrupt vector routing.
    

### 3. SPI Flash & Cache / MMU Initialization

When writing a custom 2nd-stage bootloader, you rarely write raw SPI command bit-banging from scratch. Instead, the bootloader calls ROM SPI Flash routines:

- `esp_rom_spiflash_read()`**esp_rom_spiflash_read() / esp_rom_spiflash_write() / esp_rom_spiflash_erase_sector():**`esp_rom_spiflash_write()``esp_rom_spiflash_erase_sector()` Raw block driver routines to read/write external flash sectors.
    
- `esp_rom_spiflash_config_param()`**esp_rom_spiflash_config_param():** Sets clock dividers, dummy cycles, and bus widths (DIO/QIO/OPI).
    
- `Cache_Allocate_NDrama_Addr()`**Cache_Allocate_NDrama_Addr() / Cache_Suspend_DCache() / Cache_Resume_DCache():**`Cache_Suspend_DCache()``Cache_Resume_DCache()` Low-level Cache and MMU controller configuration routines to map external flash addresses into the Xtensa memory space.
    

### 4. Cryptographic & Hardware Accelerators

The ROM includes routines to interact directly with hardware cryptographic units:

- **SHA Hardware Interface:** Functions to feed blocks into the hardware SHA accelerator (`ets_sha_enable`, `ets_sha_process`, etc.).
    
- **RSA & AES Helpers:** Basic hardware-accelerated crypto calls used during image verification and decryption.
    
- **CRC Computations:** `crc32_le()`, `crc16_be()`, and other CRC calculation helpers used for payload verification.
    

### How These Are Linked in Bare-Metal Development

When writing bare-metal C or C++ without ESP-IDF:

1. You include the official `esp32s3.rom.ld`**esp32s3.rom.ld** (or a stripped-down custom version) in your linker command line (`-T esp32s3.rom.ld`).
    
2. This maps symbol names like `memcpy = 0x4000...;` directly to their fixed ROM addresses.
    
3. **Advantage:** You save substantial internal SRAM and flash space by reusing the factory ROM code rather than statically linking redundant copies from your compiler's `libc.a`.