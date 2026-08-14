# First Stage (ROM) Bootloader
- Loads second stage bootloader image to RAM (IRAM and DRAM) from flash offset 0x1000

# Second Stage (Flash) Bootloader
- Located at address 0x1000 in flash
## Capabilities / Responsiblities
- Flash encryption
- Secure boot
- Factory reset
- Boot test firmware
- App rollback
- Use watchdog timer to reset chip if application doesn't start
- Fast boot from deep sleep
- Select application partition to boot (from partition table)
- Load boot partition to RAM and run it
# Startup Code
- Initialize DRAM:
	- Copy .data section into DRAM
	- Zero-initialize .bss section in DRAM