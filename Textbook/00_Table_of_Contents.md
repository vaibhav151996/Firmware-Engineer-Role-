# Firmware Engineering: From Beginner to Expert
## A Complete Textbook Using STM32 Nucleo Boards

**Learning Timeline: 6–12 Months**

---

```
    ┌─────────────────────────────────────────────────────────────┐
    │          FIRMWARE ENGINEERING LEARNING ROADMAP               │
    │                                                             │
    │   BEGINNER (Months 1-3)                                     │
    │   ├── C Fundamentals for Embedded                           │
    │   ├── STM32 Architecture & Toolchain                        │
    │   ├── GPIO, Clocks, Interrupts                              │
    │   └── ★ Milestone: Bare-Metal LED Blink                     │
    │                                                             │
    │   INTERMEDIATE (Months 4-7)                                 │
    │   ├── UART, SPI, I2C, Timers, ADC, DMA                     │
    │   ├── Communication Protocols                               │
    │   ├── Build Systems & Debugging                             │
    │   └── ★ Milestone: UART Driver from Datasheet               │
    │                                                             │
    │   ADVANCED (Months 8-12)                                    │
    │   ├── RTOS (FreeRTOS), Bootloaders                          │
    │   ├── CAN, USB, Embedded C++, ARM Assembly                  │
    │   ├── Python Automation, Other Platforms                    │
    │   └── ★ Milestone: Capstone Product Project                 │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘
```

---

## Table of Contents

### PART I — FOUNDATIONS (Beginner · Months 1–3)

| Chapter | Title | Key Milestone |
|---------|-------|---------------|
| 1 | [Introduction to Microcontrollers & Bare-Metal LED Blink](./Chapter_01_Introduction_and_LED_Blink.md) | ★ Register-level LED Blink |
| 2 | [Embedded C Deep Dive](./Chapter_02_Embedded_C_Deep_Dive.md) | Memory, pointers, volatile, bitwise |
| 3 | [Development Environment & Toolchain](./Chapter_03_Development_Environment.md) | STM32CubeIDE, GCC, flashing |
| 4 | [GPIO — Register-Level Mastery](./Chapter_04_GPIO_Mastery.md) | Input/output, alternate functions |
| 5 | [Clock System & Power Management](./Chapter_05_Clock_System.md) | HSI, HSE, PLL, prescalers |
| 6 | [Interrupts & the NVIC](./Chapter_06_Interrupts_NVIC.md) | EXTI, priorities, ISR design |

### PART II — CORE PERIPHERALS (Beginner–Intermediate · Months 4–6)

| Chapter | Title | Key Milestone |
|---------|-------|---------------|
| 7 | [UART Communication](./Chapter_07_UART.md) | ★ UART Driver from Datasheet |
| 8 | [Timers & PWM](./Chapter_08_Timers_PWM.md) | Hardware timers, PWM generation |
| 9 | [ADC — Analog to Digital Conversion](./Chapter_09_ADC.md) | Sensor reading, conversion |
| 10 | [SPI Protocol](./Chapter_10_SPI.md) | Full-duplex, master/slave |
| 11 | [I2C Protocol](./Chapter_11_I2C.md) | Multi-device bus |
| 12 | [DMA — Direct Memory Access](./Chapter_12_DMA.md) | Offloading CPU transfers |

### PART III — ADVANCED COMMUNICATION (Intermediate · Months 6–7)

| Chapter | Title | Key Milestone |
|---------|-------|---------------|
| 13 | [CAN Bus](./Chapter_13_CAN_Bus.md) | Automotive/industrial protocol |
| 14 | [USB Communication](./Chapter_14_USB.md) | CDC, HID on STM32 |

### PART IV — SOFTWARE ARCHITECTURE (Intermediate–Advanced · Months 7–9)

| Chapter | Title | Key Milestone |
|---------|-------|---------------|
| 15 | [Build Systems — GCC, Make, CMake](./Chapter_15_Build_Systems.md) | Professional build workflows |
| 16 | [Memory Maps & Linker Scripts](./Chapter_16_Memory_Linker.md) | STM32 flash/RAM layout |
| 17 | [FreeRTOS on STM32](./Chapter_17_FreeRTOS.md) | ★ Multi-task RTOS Application |
| 18 | [Bootloader Design](./Chapter_18_Bootloader.md) | ★ Custom STM32 Bootloader |

### PART V — PROFESSIONAL SKILLS (Advanced · Months 9–11)

| Chapter | Title | Key Milestone |
|---------|-------|---------------|
| 19 | [Debugging & Test Equipment](./Chapter_19_Debugging.md) | ST-Link, oscilloscope, logic analyzer |
| 20 | [Embedded C++ for Firmware](./Chapter_20_Embedded_Cpp.md) | Safe subsets, templates, constexpr |
| 21 | [ARM Assembly on Cortex-M](./Chapter_21_ARM_Assembly.md) | Instruction set, startup code |
| 22 | [Python for Firmware Engineers](./Chapter_22_Python_Automation.md) | Scripting, flashing, log parsing |
| 23 | [Other Platforms — AVR, ESP32, PIC, NXP](./Chapter_23_Other_Platforms.md) | Comparative architecture study |

### PART VI — CAPSTONE (Advanced · Month 12)

| Chapter | Title | Key Milestone |
|---------|-------|---------------|
| 24 | [End-to-End Product Project](./Chapter_24_Capstone_Project.md) | ★ Temperature-Controlled Fan System |

### APPENDICES

| Appendix | Title |
|----------|-------|
| A | [STM32 Register Quick Reference](./Appendix_A_Register_Reference.md) |
| B | [Recommended Datasheets & Resources](./Appendix_B_Resources.md) |
| C | [Glossary of Firmware Terms](./Appendix_C_Glossary.md) |

---

## Hardware Requirements

| Item | Purpose | Required? |
|------|---------|-----------|
| STM32 Nucleo-F401RE (or F446RE) | Primary learning board | **YES** |
| USB Micro-B cable | Power + programming via ST-Link | **YES** |
| Breadboard + jumper wires | External circuits | Recommended |
| LEDs, resistors (330Ω), push buttons | Basic experiments | Recommended |
| Logic analyzer (e.g., Saleae clone) | Protocol debugging | Recommended |
| Oscilloscope (any entry-level) | Signal analysis | Optional |
| Temperature sensor (e.g., LM35, TMP36) | ADC projects | For Ch. 9, 24 |
| Small DC fan + MOSFET (e.g., IRF520) | PWM project | For Ch. 24 |
| SPI device (e.g., SPI flash W25Q64) | SPI practice | For Ch. 10 |
| I2C device (e.g., OLED SSD1306) | I2C practice | For Ch. 11 |
| CAN transceiver (e.g., MCP2551) | CAN bus practice | For Ch. 13 |

---

## Software Requirements

| Tool | Purpose |
|------|---------|
| STM32CubeIDE | Primary IDE (includes compiler, debugger) |
| STM32CubeMX | Peripheral configuration (integrated in CubeIDE) |
| ARM GCC Toolchain | Compiler (bundled with CubeIDE) |
| Git | Version control |
| PuTTY / Tera Term / minicom | Serial terminal |
| Python 3.x | Automation scripts |
| Make / CMake | Build system chapters |

---

## How to Use This Textbook

```
    ┌──────────────────────────────────────────────┐
    │              STUDY WORKFLOW                   │
    │                                              │
    │   1. READ the concept overview               │
    │          ↓                                   │
    │   2. STUDY the STM32 hardware internals      │
    │          ↓                                   │
    │   3. TYPE the code (never copy-paste)         │
    │          ↓                                   │
    │   4. FLASH to your Nucleo board              │
    │          ↓                                   │
    │   5. DEBUG when things go wrong              │
    │          ↓                                   │
    │   6. COMPLETE the exercises                  │
    │          ↓                                   │
    │   7. MOVE to the next chapter                │
    └──────────────────────────────────────────────┘
```

> **Golden Rule:** Every single code example in this book is designed to run on a real STM32 Nucleo board. Type it. Flash it. Debug it. That is how firmware engineers are made.

---

*Begin with [Chapter 1 →](./Chapter_01_Introduction_and_LED_Blink.md)*
