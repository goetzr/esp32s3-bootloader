Here is the architectural hierarchy of ESP-IDF components, arranged from the highest application layers down to the physical silicon and ROM.

### Layered Architecture Overview

```
+-------------------------------------------------------------------------+
| Level 6: Application & User Logic (main, app_main, C++ domain code)     |
+-------------------------------------------------------------------------+
| Level 5: High-Level Protocols & Middleware (HTTP, MQTT, BLE, TLS, VFS)  |
+-------------------------------------------------------------------------+
| Level 4: OS Services, Memory & System Orchestration (FreeRTOS, esp_system)
+-------------------------------------------------------------------------+
| Level 3: Peripheral Driver Framework (driver, spi_flash, esp_timer)     |
+-------------------------------------------------------------------------+
| Level 2: Hardware Abstraction & Low-Level (hal, esp_libc, esp_rom)      |
+-------------------------------------------------------------------------+
| Level 1: Hardware Definition & Register Maps (soc)                      |
+-------------------------------------------------------------------------+
| Level 0: Physical Hardware & Mask ROM (Silicon registers, ROM bootloader)
+-------------------------------------------------------------------------+
```

### Component Hierarchy (Top to Bottom)

#### Level 6: Application Layer

The top level where user firmware and application logic reside.

- `main`**main**: The entry point component defining `app_main()` or custom C++ application runtime logic.
    

#### Level 5: High-Level Middleware, Networking & Protocol Stacks

Rich abstractions that sit on top of OS primitives, filesystems, and device drivers.

- **Networking & Cloud:** `lwip`, `esp_netif`, `esp_wifi`, `esp_eth`, `mqtt`, `esp_http_client`, `esp_http_server`, `esp_https_ota`
    
- **Security & Cryptography:** `mbedtls`, `esp_crypto_shared`
    
- **Storage & File Systems:** `vfs` (Virtual File System), `fatfs`, `spiffs`, `nvs_flash` (Non-Volatile Storage)
    
- **Bluetooth / Connectivity:** `bt` (Bluedroid / NimBLE stacks), `esp_event` (event dispatching loop)
    
- **Boot Management:** `bootloader_support`, `app_update` (OTA partition switching, image verification)
    

#### Level 4: Core System Orchestration & OS Services

The central hub managing task scheduling, system lifecycles, and memory allocation across cores.

- `freertos`**freertos**: FreeRTOS kernel (ESP-IDF SMP dual-core variant), task scheduler, semaphores, queues.
    
- `esp_system`**esp_system**: Core runtime initialization, startup sequence, panic/crash handlers, reset reason tracking, brownout monitoring, hardware RNG.
    
- `heap`**heap**: Dynamic multi-heap allocator (`heap_caps_malloc`) handling internal SRAM, IRAM, and external PSRAM pools.
    
- `esp_timer`**esp_timer**: High-resolution software timer daemon using hardware timer backends.
    
- `log`**log**: Tagged logging framework (`ESP_LOGI`, `ESP_LOGE`) routing messages to console devices.
    
- `esp_pm`**esp_pm / esp_sleep**`esp_sleep`: Power management, dynamic frequency scaling, light sleep, and deep sleep controls.
    

#### Level 3: Peripheral Driver Framework

Thread-safe, interrupt-driven, FreeRTOS-aware drivers providing high-level peripheral control.

- `driver`**driver**: Drivers for standard digital and analog peripherals:
    
    - `gpio`, `uart`, `i2c`, `spi_master`/`spi_slave`, `gptimer`, `ledc` (PWM), `i2s`, `adc`, `dac`, `usb`
        
- `spi_flash`**spi_flash**: Flash chip drivers, cache/MMU page-mapping abstraction, partition table readers, and wear levelling.
    
- `efuse`**efuse**: High-level eFuse burning and configuration reading routines.
    

#### Level 2: Hardware Abstraction Layer (HAL) & Glue Primitives

Stateless, RTOS-agnostic register helpers and standard library glue layers.

- `hal`**hal / LL (Low-Level)**: Stateless C functions implementing discrete peripheral operations directly against memory-mapped registers.
    
- `esp_libc`**esp_libc**: Toolchain C-library integration layer (Newlib/Picolibc syscall bindings like `_write`, `_read`, reentrancy structs, thread-safe mutexes).
    
- `esp_rom`**esp_rom**: Target-specific ROM wrappers, linker scripts, and hardware jump tables exposing built-in mask-ROM functions.
    
- `esp_hw_support`**esp_hw_support**: Low-level hardware assist routines (CPU frequency clock switching, interrupt allocators, low-level MAC derivation, cache maintenance).
    

#### Level 1: Hardware Definition Layer

Pure hardware definitions with zero runtime dependencies.

- `soc`**soc**:
    
    - Peripheral base memory addresses and memory-map boundaries (`soc.h`).
        
    - Register bitfields and packed struct layouts (`*_reg.h`, `*_struct.h`).
        
    - Target capability matrices (`soc_caps.h`).
        

#### Level 0: Silicon & Mask ROM

The immutable hardware layer underneath software control.

- **Hardware Peripherals & CPU**: Xtensa / RISC-V cores, bus matrix, DMA controllers, MMU/Cache controllers, peripheral FIFOs, eFuse array.
    
- **Internal Mask ROM**: 1st-stage ROM bootloader, factory calibration data, and base `libc`/ETS hardware routines.