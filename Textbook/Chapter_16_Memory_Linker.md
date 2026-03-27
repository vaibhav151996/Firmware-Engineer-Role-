# Chapter 16: Memory Maps & Linker Scripts
## STM32 Flash/RAM Layout and Controlling Code Placement

**Difficulty Level: ★★★★☆ Advanced**  
**Estimated Time: 1–2 weeks**

---

## 16.1 Concept Overview

### What Is a Linker Script?

A linker script (.ld file) tells the linker **where to place code and data** in the microcontroller's memory. Without it, the linker wouldn't know that Flash starts at `0x08000000` or that RAM is at `0x20000000`.

```
  The linker script answers critical questions:
  
  ┌─────────────────────────────────────────────────────┐
  │  Q: Where does my code go?                          │
  │  A: .text section → Flash at 0x08000000             │
  │                                                     │
  │  Q: Where do initialized variables go?              │
  │  A: .data init values → Flash (read at startup)     │
  │     .data runtime copy → RAM at 0x20000000          │
  │                                                     │
  │  Q: Where does the stack start?                     │
  │  A: Top of RAM: 0x20000000 + RAM_SIZE               │
  │                                                     │
  │  Q: How big is Flash/RAM?                           │
  │  A: Defined in the MEMORY command of the script     │
  └─────────────────────────────────────────────────────┘
```

---

## 16.2 STM32F401 Memory Map

```
  Complete STM32F401 Address Space:
  
  0x0000_0000  ┌──────────────────────┐
               │ Aliased to Flash or  │  Boot mode dependent
               │ System Memory        │
  0x0800_0000  ├──────────────────────┤
               │   FLASH (512 KB)     │  Your program (.text)
               │   Sectors 0-7        │  Constants (.rodata)
               │                      │  Init data (.data source)
  0x0808_0000  ├──────────────────────┤
               │   (Reserved)         │
  0x1FFF_0000  ├──────────────────────┤
               │  System Memory       │  ST bootloader (ROM)
               │  (30 KB)             │  DFU, UART boot
  0x1FFF_7800  ├──────────────────────┤
               │  OTP Area (528 B)    │  One-Time Programmable
  0x1FFF_C000  ├──────────────────────┤
               │  Option Bytes (16 B) │  Read/write protection
  0x2000_0000  ├──────────────────────┤
               │   SRAM (96 KB)       │  Variables (.data, .bss)
               │                      │  Stack, Heap
  0x2001_8000  ├──────────────────────┤
               │   (Reserved)         │
  0x4000_0000  ├──────────────────────┤
               │   Peripherals        │  GPIO, UART, SPI, etc.
  0xE000_0000  ├──────────────────────┤
               │  Cortex-M Core       │  NVIC, SysTick, SCB
  0xFFFF_FFFF  └──────────────────────┘
  
  
  Flash Sector Layout (STM32F401, 512 KB):
  
  Sector  Address Range              Size
  ──────  ─────────────────────────  ──────
  0       0x0800_0000 - 0x0800_3FFF  16 KB
  1       0x0800_4000 - 0x0800_7FFF  16 KB
  2       0x0800_8000 - 0x0800_BFFF  16 KB
  3       0x0800_C000 - 0x0800_FFFF  16 KB
  4       0x0801_0000 - 0x0801_FFFF  64 KB
  5       0x0802_0000 - 0x0803_FFFF  128 KB
  6       0x0804_0000 - 0x0805_FFFF  128 KB
  7       0x0806_0000 - 0x0807_FFFF  128 KB
  
  This layout is critical for bootloader design (Chapter 18):
  Bootloader → Sectors 0-3 (64 KB)
  Application → Sectors 4-7 (448 KB)
```

---

## 16.3 Anatomy of a Linker Script

```ld
/* =========================================================================
 * STM32F401RETX_FLASH.ld — Annotated Linker Script
 * ========================================================================= */

/* Entry point — first function to execute */
ENTRY(Reset_Handler)

/* Stack size */
_estack = ORIGIN(RAM) + LENGTH(RAM);  /* Top of RAM */
_Min_Heap_Size = 0x200;   /* 512 bytes for heap (if used) */
_Min_Stack_Size = 0x400;  /* 1024 bytes minimum for stack */

/* =========================================================================
 * MEMORY: Define available memory regions
 * ========================================================================= */
MEMORY
{
    RAM   (xrw)  : ORIGIN = 0x20000000, LENGTH = 96K
    FLASH (rx)   : ORIGIN = 0x08000000, LENGTH = 512K
}
/* 
 * RAM:   x=executable, r=readable, w=writable
 * FLASH: r=readable, x=executable (not writable at runtime!)
 */

/* =========================================================================
 * SECTIONS: Define where each type of data goes
 * ========================================================================= */
SECTIONS
{
    /* --- Vector Table (MUST be at flash start!) --- */
    .isr_vector :
    {
        . = ALIGN(4);
        KEEP(*(.isr_vector))   /* KEEP prevents garbage collection */
        . = ALIGN(4);
    } >FLASH

    /* --- Code (.text) --- */
    .text :
    {
        . = ALIGN(4);
        *(.text)           /* All code */
        *(.text*)          /* All code subsections */
        *(.glue_7)         /* ARM/Thumb interworking */
        *(.glue_7t)
        KEEP(*(.init))     /* C runtime init */
        KEEP(*(.fini))     /* C runtime fini */
        . = ALIGN(4);
        _etext = .;        /* Symbol marking end of code */
    } >FLASH

    /* --- Read-Only Data (.rodata) — const variables, string literals --- */
    .rodata :
    {
        . = ALIGN(4);
        *(.rodata)
        *(.rodata*)
        . = ALIGN(4);
    } >FLASH

    /* --- Initialized Data (.data) --- */
    /* Source is in Flash (for initial values), 
       destination is in RAM (runtime location) */
    _sidata = LOADADDR(.data);  /* Flash address of init values */
    
    .data :
    {
        . = ALIGN(4);
        _sdata = .;        /* Start of .data in RAM */
        *(.data)
        *(.data*)
        . = ALIGN(4);
        _edata = .;        /* End of .data in RAM */
    } >RAM AT> FLASH
    /* >RAM: VMA (Virtual Memory Address) = where it runs */
    /* AT> FLASH: LMA (Load Memory Address) = where it's stored */

    /* --- Zero-Initialized Data (.bss) --- */
    .bss :
    {
        . = ALIGN(4);
        _sbss = .;         /* Start of .bss */
        __bss_start__ = _sbss;
        *(.bss)
        *(.bss*)
        *(COMMON)
        . = ALIGN(4);
        _ebss = .;         /* End of .bss */
        __bss_end__ = _ebss;
    } >RAM

    /* --- Stack and Heap check --- */
    ._user_heap_stack :
    {
        . = ALIGN(8);
        PROVIDE(end = .);
        . = . + _Min_Heap_Size;
        . = . + _Min_Stack_Size;
        . = ALIGN(8);
    } >RAM
}
```

### How Startup Code Uses Linker Symbols

```c
/*
 * The startup code (startup_stm32f401retx.s) uses the linker symbols
 * _sidata, _sdata, _edata, _sbss, _ebss to initialize RAM:
 */

/* Pseudocode of what the startup assembly does: */
void Reset_Handler(void)
{
    /* 1. Copy .data from Flash to RAM */
    uint32_t *src = &_sidata;  /* Source: Flash */
    uint32_t *dst = &_sdata;   /* Destination: RAM */
    while (dst < &_edata)
    {
        *dst++ = *src++;
    }

    /* 2. Zero out .bss */
    dst = &_sbss;
    while (dst < &_ebss)
    {
        *dst++ = 0;
    }

    /* 3. Call main() */
    main();
}
```

---

## 16.4 Placing Data in Specific Sections

```c
/* Place a variable in a custom section */
__attribute__((section(".my_config")))
const uint32_t device_config[4] = {0xDEADBEEF, 0x01, 0x02, 0x03};

/* Place a function in RAM (for speed or Flash programming) */
__attribute__((section(".RamFunc")))
void fast_function(void)
{
    /* This function runs from RAM — faster than Flash at high speeds */
}

/* Place data at an absolute address (e.g., for bootloader shared data) */
__attribute__((section(".shared_data")))
volatile uint32_t bootloader_magic __attribute__((used));
```

---

## 16.5 Exercises

**Exercise 16.1:** Modify the linker script to reserve 16 KB of Flash for a bootloader (Sectors 0-3) and start the application at 0x08010000.

**Exercise 16.2:** Use `arm-none-eabi-nm` to list all symbols and their addresses. Find where your `main()`, global variables, and stack are located.

**Exercise 16.3:** Create a custom section for calibration data and place it in the last Flash sector. Write code that can read these values at runtime.

**Exercise 16.4:** Analyze the `.map` file from a build. Create a table showing how much Flash and RAM each source file uses.

---

## 16.6 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 16 SUMMARY                        │
  │                                                            │
  │  ✓ The linker script defines Flash and RAM regions          │
  │  ✓ .text (code) and .rodata (constants) go to Flash        │
  │  ✓ .data (initialized vars) stored in Flash, copied to RAM │
  │  ✓ .bss (zero-init vars) goes to RAM, zeroed at startup    │
  │  ✓ Stack grows downward from top of RAM                    │
  │  ✓ Startup code uses linker symbols (_sidata, _sdata, etc) │
  │  ✓ Custom sections allow precise placement of data         │
  │  ✓ Understanding the memory map is essential for           │
  │    bootloader design and memory optimization               │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 17 — FreeRTOS on STM32 →](./Chapter_17_FreeRTOS.md)*
