# Chapter 18: Bootloader Design
## ★ MILESTONE: Custom UART Bootloader for STM32

**Difficulty Level: ★★★★★ Expert**  
**Estimated Time: 3–4 weeks**

---

## 18.1 Concept Overview

### What Is a Bootloader?

A bootloader is a small program that runs **before** your main application. Its job is to:
1. Check if a firmware update is available
2. If yes: receive the new firmware, write it to Flash, and then jump to it
3. If no: jump directly to the existing application

```
  Boot Process:
  
  Power On / Reset
       │
       ↓
  ┌──────────────┐
  │  BOOTLOADER  │  Runs first (sits at Flash start: 0x08000000)
  │              │
  │  Check for   │
  │  update?     │
  └──────┬───────┘
         │
    ┌────┴────┐
    │         │
   YES        NO
    │         │
    ↓         ↓
  ┌──────┐  ┌──────────┐
  │Update│  │ Jump to  │
  │FW via│  │Application│
  │UART  │  │ at       │
  └──┬───┘  │0x08010000│
     │      └──────────┘
     ↓
  ┌──────────┐
  │Write new │
  │FW to     │
  │Flash     │
  └──┬───────┘
     │
     ↓
  ┌──────────┐
  │ Jump to  │
  │ new App  │
  └──────────┘
```

### Memory Layout for Bootloader + Application

```
  Flash Memory Layout:
  
  0x0800_0000  ┌──────────────────────┐
               │     BOOTLOADER       │  Sector 0-3 (64 KB)
               │                      │  Linker script: 0x08000000, 64K
               │  Vector Table (BL)   │
               │  Bootloader Code     │
               │  UART receive        │
               │  Flash programming   │
               │  Jump logic          │
  0x0801_0000  ├──────────────────────┤
               │     APPLICATION      │  Sector 4-7 (448 KB)
               │                      │  Linker script: 0x08010000, 448K
               │  Vector Table (APP)  │  ← VTOR must point here!
               │  Application Code    │
               │  (Your FreeRTOS      │
               │   project, etc.)     │
               │                      │
  0x0808_0000  └──────────────────────┘
```

---

## 18.2 STM32 Hardware Internals

### Flash Programming

```
  STM32F401 Flash Programming Rules:
  
  1. Unlock Flash (write keys to FLASH_KEYR):
     Key 1: 0x45670123
     Key 2: 0xCDEF89AB
  
  2. Erase by SECTOR (not individual bytes!):
     Sector 4 (64 KB) = smallest sector for app start
     Erasing takes time (~1-2 seconds for large sectors)
  
  3. Program in 8/16/32-bit words (set PSIZE):
     Must be aligned
     Can only change 1→0 (not 0→1, that's what erase does)
  
  4. Lock Flash when done (set LOCK bit)
  
  FLASH Register Map:
  Base: 0x40023C00
  FLASH_ACR    (0x00): Access control (wait states)
  FLASH_KEYR   (0x04): Key register (unlock)
  FLASH_SR     (0x0C): Status (busy, errors)
  FLASH_CR     (0x10): Control (erase, program, lock)
```

### Jumping to Application — How It Works

```
  The "jump to app" process:
  
  1. Read the app's initial stack pointer:
     uint32_t app_sp = *(uint32_t *)0x08010000;
  
  2. Read the app's reset handler address:
     uint32_t app_reset = *(uint32_t *)0x08010004;
  
  3. Relocate the vector table:
     SCB->VTOR = 0x08010000;
     (So interrupts go to the APP's handlers, not the bootloader's)
  
  4. Set the stack pointer:
     __set_MSP(app_sp);
  
  5. Jump!
     ((void (*)(void))app_reset)();
  
  ┌──────────────────────────────────────────────────┐
  │  Application's Vector Table at 0x08010000:        │
  │                                                  │
  │  0x08010000:  0x20017FFF  (Initial SP)           │
  │  0x08010004:  0x080101A1  (Reset Handler)        │
  │  0x08010008:  0x08010245  (NMI Handler)          │
  │  0x0801000C:  0x08010249  (HardFault Handler)    │
  │  ...                                             │
  └──────────────────────────────────────────────────┘
```

---

## 18.3 Step-by-Step Implementation

### Bootloader Code

```c
/******************************************************************************
 * File:    bootloader.c
 * Brief:   UART bootloader for STM32F401 — bare-metal
 *
 * ★ MILESTONE: Bootloader Implementation
 *
 * Memory layout:
 *   Bootloader: 0x08000000 — 0x0800FFFF (64 KB, Sectors 0-3)
 *   Application: 0x08010000 — 0x0807FFFF (448 KB, Sectors 4-7)
 *
 * Boot protocol:
 *   1. Wait 3 seconds for 'U' character on UART
 *   2. If received: enter update mode (receive binary, write to Flash)
 *   3. If timeout: jump to application
 *
 * Update protocol (simple):
 *   1. Bootloader sends "READY\n"
 *   2. Host sends 4-byte file size (little-endian)  
 *   3. Bootloader sends "OK\n"
 *   4. Host sends firmware binary data
 *   5. Bootloader writes data to Flash in chunks
 *   6. Bootloader sends "DONE\n" and jumps to app
 *****************************************************************************/

#include <stdint.h>

/* ========================================================================= */
/*                           CONSTANTS                                       */
/* ========================================================================= */

#define APP_START_ADDRESS    0x08010000UL
#define FLASH_KEY1           0x45670123UL
#define FLASH_KEY2           0xCDEF89ABUL
#define UART_TIMEOUT_MS      3000UL      /* 3 second boot delay */

/* ========================================================================= */
/*                       REGISTER DEFINITIONS                                */
/* ========================================================================= */

/* RCC */
#define RCC_AHB1ENR     (*(volatile uint32_t *)0x40023830UL)
#define RCC_APB1ENR     (*(volatile uint32_t *)0x40023840UL)

/* GPIOA (PA2 = TX, PA3 = RX) */
#define GPIOA_MODER     (*(volatile uint32_t *)0x40020000UL)
#define GPIOA_AFRL      (*(volatile uint32_t *)0x40020020UL)

/* USART2 */
#define USART2_SR       (*(volatile uint32_t *)0x40004400UL)
#define USART2_DR       (*(volatile uint32_t *)0x40004404UL)
#define USART2_BRR      (*(volatile uint32_t *)0x40004408UL)
#define USART2_CR1      (*(volatile uint32_t *)0x4000440CUL)

/* Flash */
#define FLASH_ACR       (*(volatile uint32_t *)0x40023C00UL)
#define FLASH_KEYR      (*(volatile uint32_t *)0x40023C04UL)
#define FLASH_SR        (*(volatile uint32_t *)0x40023C0CUL)
#define FLASH_CR        (*(volatile uint32_t *)0x40023C10UL)

/* SCB (System Control Block) */
#define SCB_VTOR        (*(volatile uint32_t *)0xE000ED08UL)

/* SysTick for timeout */
#define SYST_CSR        (*(volatile uint32_t *)0xE000E010UL)
#define SYST_RVR        (*(volatile uint32_t *)0xE000E014UL)
#define SYST_CVR        (*(volatile uint32_t *)0xE000E018UL)

/* ========================================================================= */
/*                      LOW-LEVEL FUNCTIONS                                  */
/* ========================================================================= */

static volatile uint32_t tick_count = 0;

void SysTick_Handler(void)
{
    tick_count++;
}

static void systick_init(void)
{
    SYST_RVR = 16000UL - 1UL;   /* 1 ms at 16 MHz HSI */
    SYST_CVR = 0UL;
    SYST_CSR = 7UL;              /* Enable + interrupt + processor clock */
}

static void uart_init_bl(void)
{
    RCC_AHB1ENR |= (1UL << 0);   /* GPIOA */
    RCC_APB1ENR |= (1UL << 17);  /* USART2 */

    /* PA2 = AF7 (TX), PA3 = AF7 (RX) */
    GPIOA_MODER &= ~((3UL << 4) | (3UL << 6));
    GPIOA_MODER |=  ((2UL << 4) | (2UL << 6));
    GPIOA_AFRL  &= ~((0xFUL << 8) | (0xFUL << 12));
    GPIOA_AFRL  |=  ((7UL << 8) | (7UL << 12));

    USART2_BRR = 0x008BUL;  /* 115200 baud at 16 MHz */
    USART2_CR1 = (1UL << 13) | (1UL << 3) | (1UL << 2);  /* UE, TE, RE */
}

static void uart_send(char c)
{
    while (!(USART2_SR & (1UL << 7)));
    USART2_DR = (uint32_t)c;
}

static void uart_print(const char *s)
{
    while (*s) { uart_send(*s++); }
}

static int uart_receive_timeout(uint8_t *c, uint32_t timeout_ms)
{
    uint32_t start = tick_count;
    while ((tick_count - start) < timeout_ms)
    {
        if (USART2_SR & (1UL << 5))
        {
            *c = (uint8_t)(USART2_DR & 0xFFUL);
            return 1;
        }
    }
    return 0;  /* Timeout */
}

static uint8_t uart_receive_blocking(void)
{
    while (!(USART2_SR & (1UL << 5)));
    return (uint8_t)(USART2_DR & 0xFFUL);
}

/* ========================================================================= */
/*                       FLASH PROGRAMMING                                   */
/* ========================================================================= */

static void flash_unlock(void)
{
    if (FLASH_CR & (1UL << 31))  /* LOCK bit */
    {
        FLASH_KEYR = FLASH_KEY1;
        FLASH_KEYR = FLASH_KEY2;
    }
}

static void flash_lock(void)
{
    FLASH_CR |= (1UL << 31);  /* Set LOCK bit */
}

static void flash_wait_busy(void)
{
    while (FLASH_SR & (1UL << 16));  /* Wait for BSY clear */
}

/*
 * Erase a Flash sector
 * sector_num: 0-7 for STM32F401
 */
static void flash_erase_sector(uint8_t sector_num)
{
    flash_wait_busy();

    FLASH_CR &= ~(0xFUL << 3);              /* Clear SNB bits */
    FLASH_CR |= ((uint32_t)sector_num << 3); /* Set sector number */
    FLASH_CR |= (1UL << 1);                 /* SER: Sector erase */
    FLASH_CR |= (1UL << 16);                /* STRT: Start erase */

    flash_wait_busy();

    FLASH_CR &= ~(1UL << 1);                /* Clear SER */
}

/*
 * Program a byte to Flash
 */
static void flash_program_byte(uint32_t address, uint8_t data)
{
    flash_wait_busy();

    FLASH_CR &= ~(3UL << 8);    /* PSIZE = 00 (byte) */
    FLASH_CR |= (1UL << 0);     /* PG: Programming mode */

    *(volatile uint8_t *)address = data;

    flash_wait_busy();

    FLASH_CR &= ~(1UL << 0);    /* Clear PG */
}

/* ========================================================================= */
/*                       JUMP TO APPLICATION                                 */
/* ========================================================================= */

static void jump_to_app(void)
{
    /* Read app's initial stack pointer and reset handler */
    uint32_t app_sp    = *(volatile uint32_t *)(APP_START_ADDRESS);
    uint32_t app_reset = *(volatile uint32_t *)(APP_START_ADDRESS + 4UL);

    /* Sanity check: is there a valid application?
     * The stack pointer should point somewhere in RAM (0x20000000-0x20017FFF)
     */
    if ((app_sp & 0xFFF00000UL) != 0x20000000UL)
    {
        uart_print("No valid app found!\r\n");
        return;  /* No valid application — stay in bootloader */
    }

    uart_print("Jumping to application...\r\n");

    /* Wait for UART to finish sending */
    while (!(USART2_SR & (1UL << 6)));

    /* Disable SysTick */
    SYST_CSR = 0UL;

    /* Disable all interrupts */
    /* (In a full implementation, disable all NVIC IRQs) */

    /* Relocate vector table to application */
    SCB_VTOR = APP_START_ADDRESS;

    /* Set stack pointer to app's initial SP */
    __asm volatile ("MSR MSP, %0" : : "r" (app_sp));

    /* Jump to app's Reset Handler */
    void (*app_entry)(void) = (void (*)(void))app_reset;
    app_entry();

    /* Should never reach here */
    while (1);
}

/* ========================================================================= */
/*                       UPDATE FIRMWARE                                     */
/* ========================================================================= */

static void update_firmware(void)
{
    uint32_t fw_size;
    uint32_t addr;

    uart_print("READY\r\n");

    /* Receive firmware size (4 bytes, little-endian) */
    fw_size  = (uint32_t)uart_receive_blocking();
    fw_size |= (uint32_t)uart_receive_blocking() << 8;
    fw_size |= (uint32_t)uart_receive_blocking() << 16;
    fw_size |= (uint32_t)uart_receive_blocking() << 24;

    if (fw_size == 0UL || fw_size > (448UL * 1024UL))
    {
        uart_print("ERROR: Invalid size\r\n");
        return;
    }

    uart_print("OK\r\n");

    /* Erase application sectors (4-7) */
    uart_print("Erasing...\r\n");
    flash_unlock();
    flash_erase_sector(4);  /* 64 KB */
    flash_erase_sector(5);  /* 128 KB */
    flash_erase_sector(6);  /* 128 KB */
    flash_erase_sector(7);  /* 128 KB */

    /* Receive and program firmware */
    uart_print("Programming...\r\n");
    addr = APP_START_ADDRESS;
    for (uint32_t i = 0; i < fw_size; i++)
    {
        uint8_t byte = uart_receive_blocking();
        flash_program_byte(addr, byte);
        addr++;

        /* Progress indicator every 1 KB */
        if ((i & 0x3FFUL) == 0UL)
        {
            uart_send('.');
        }
    }

    flash_lock();
    uart_print("\r\nDONE\r\n");
}

/* ========================================================================= */
/*                              MAIN                                         */
/* ========================================================================= */

int main(void)
{
    uint8_t c;

    systick_init();
    uart_init_bl();

    uart_print("\r\n[BOOTLOADER] v1.0\r\n");
    uart_print("Send 'U' within 3 seconds to update firmware...\r\n");

    /* Wait for update command */
    if (uart_receive_timeout(&c, UART_TIMEOUT_MS) && c == 'U')
    {
        uart_print("Entering update mode...\r\n");
        update_firmware();
    }
    else
    {
        uart_print("No update command. Booting application...\r\n");
    }

    /* Jump to application */
    jump_to_app();

    /* Should never reach here */
    while (1)
    {
        uart_print("ERROR: App jump failed!\r\n");
        for (volatile uint32_t i = 0; i < 1600000UL; i++);
    }

    return 0;
}
```

### Application Linker Script Changes

The application must be linked for a different start address:

```ld
/* Application linker script: STM32F401_APP.ld */
MEMORY
{
    RAM   (xrw)  : ORIGIN = 0x20000000, LENGTH = 96K
    FLASH (rx)   : ORIGIN = 0x08010000, LENGTH = 448K  /* Start at 64K offset! */
}
```

The application must also set `SCB->VTOR = 0x08010000;` early in its startup.

---

## 18.4 Python Host Script for Firmware Upload

```python
#!/usr/bin/env python3
"""
upload_firmware.py — Send firmware binary to STM32 bootloader via UART

Usage: python upload_firmware.py COM3 firmware.bin
"""

import serial
import sys
import struct
import time

def upload(port, firmware_path):
    # Open firmware binary
    with open(firmware_path, 'rb') as f:
        firmware = f.read()
    
    print(f"Firmware size: {len(firmware)} bytes")
    
    # Open serial port
    ser = serial.Serial(port, 115200, timeout=5)
    time.sleep(0.1)
    
    # Send update command
    ser.write(b'U')
    
    # Wait for READY
    line = ser.readline().decode().strip()
    print(f"Bootloader: {line}")
    if 'READY' not in line:
        print("ERROR: Bootloader not ready")
        return
    
    # Send firmware size (4 bytes, little-endian)
    ser.write(struct.pack('<I', len(firmware)))
    
    # Wait for OK
    line = ser.readline().decode().strip()
    print(f"Bootloader: {line}")
    if 'OK' not in line:
        print("ERROR: Size rejected")
        return
    
    # Send firmware data
    print("Uploading", end='', flush=True)
    for i in range(0, len(firmware), 256):
        chunk = firmware[i:i+256]
        ser.write(chunk)
        print('.', end='', flush=True)
    
    # Wait for DONE
    time.sleep(1)
    while ser.in_waiting:
        line = ser.readline().decode().strip()
        print(f"\nBootloader: {line}")
    
    ser.close()
    print("Upload complete!")

if __name__ == '__main__':
    if len(sys.argv) != 3:
        print("Usage: python upload_firmware.py <COM_PORT> <firmware.bin>")
        sys.exit(1)
    upload(sys.argv[1], sys.argv[2])
```

---

## 18.5 Common Mistakes & Debugging

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Forgetting to set VTOR in app | App interrupts crash | Set SCB->VTOR = 0x08010000 in app startup |
| Wrong app linker script | App doesn't start | FLASH ORIGIN must match APP_START_ADDRESS |
| Not unlocking Flash | Flash writes silently fail | Write KEY1, KEY2 to FLASH_KEYR |
| Erasing wrong sector | Bootloader erased! | Double-check sector numbers (app starts at sector 4) |
| Jumping without disabling interrupts | Hard fault after jump | Disable all peripherals and NVIC before jumping |

---

## 18.6 Exercises

**Exercise 18.1:** Add CRC-32 verification: calculate CRC of the received firmware, compare with a CRC sent by the host script.

**Exercise 18.2:** Add a "golden image" fallback: if the app fails to boot (detected via watchdog timeout), revert to a known-good firmware.

**Exercise 18.3:** Implement A/B partitioning: two app slots, bootloader selects the active one based on a metadata header.

**Exercise 18.4:** Add encryption: the host encrypts the firmware, the bootloader decrypts before writing to Flash (AES-128).

---

## 18.7 Industry & Career Notes

- Writing a bootloader is a **premier interview project** — it demonstrates deep understanding of Flash, memory maps, and system design
- Production bootloaders include: CRC verification, encryption, rollback, version checking, and fail-safe mechanisms
- Many companies use the bootloader for **over-the-air (OTA) updates** in IoT products
- Standards like **AUTOSAR** and **SBOM** (Software Bill of Materials) require traceability of bootloader + application versions

---

## 18.8 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │         ★ MILESTONE: BOOTLOADER COMPLETE ★                 │
  │                                                            │
  │  ✓ Bootloader sits at Flash start (0x08000000)             │
  │  ✓ Application starts at offset (0x08010000)               │
  │  ✓ Jump sequence: set VTOR → set MSP → call Reset_Handler │
  │  ✓ Flash programming: Unlock → Erase → Program → Lock     │
  │  ✓ UART protocol for receiving firmware from host          │
  │  ✓ Python script for host-side upload                      │
  │  ✓ Safety: validate app SP before jumping                  │
  │                                                            │
  │  SYSTEMS INTEGRATED:                                       │
  │    GPIO, UART, Flash, Memory Map, Linker Scripts,         │
  │    Vector Table, Clock System, SysTick                     │
  │                                                            │
  │  This is a PORTFOLIO PROJECT — put it on your resume!      │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 19 — Debugging & Test Equipment →](./Chapter_19_Debugging.md)*
