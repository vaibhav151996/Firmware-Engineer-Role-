# Firmware Engineer Role — Complete Textbook

A comprehensive, hands-on textbook for becoming a professional Firmware Engineer. Covers bare-metal embedded development from first LED blink to a complete IoT system, targeting the **STM32F401RE Nucleo** board with register-level C code.

## Target Platform
- **MCU:** STM32F401RE (ARM Cortex-M4, 84 MHz, 512 KB Flash, 96 KB SRAM)
- **Board:** NUCLEO-F401RE
- **Toolchain:** arm-none-eabi-gcc, OpenOCD, GDB
- **Style:** Bare-metal, register-level (no HAL library)

---

## Table of Contents

### Part I: Foundations
| # | Chapter | Difficulty | Key Topics |
|---|---------|-----------|------------|
| 1 | [First Bare-Metal LED Blink](Textbook/Chapter_01_LED_Blink.md) | ★☆☆☆☆ | GPIO output, RCC clock enable, memory-mapped I/O |
| 2 | [Embedded C Deep Dive](Textbook/Chapter_02_Embedded_C_Deep_Dive.md) | ★★☆☆☆ | volatile, fixed-width types, bitwise ops, structs, MISRA |
| 3 | [Development Environment](Textbook/Chapter_03_Development_Environment.md) | ★★☆☆☆ | Toolchain, STM32CubeIDE, compiler flags, debugging |
| 4 | [GPIO Mastery](Textbook/Chapter_04_GPIO_Mastery.md) | ★★☆☆☆ | Complete GPIO driver library, input/output/AF modes |

### Part II: Core Peripherals
| # | Chapter | Difficulty | Key Topics |
|---|---------|-----------|------------|
| 5 | [Clock System & SysTick](Textbook/Chapter_05_Clock_System.md) | ★★★☆☆ | PLL configuration to 84 MHz, SysTick timer, delay_ms |
| 6 | [Interrupts & NVIC](Textbook/Chapter_06_Interrupts_NVIC.md) | ★★★☆☆ | EXTI, NVIC priorities, ISR design patterns |
| 7 | [UART Communication](Textbook/Chapter_07_UART.md) | ★★★☆☆ | ★ MILESTONE — Complete UART driver, interrupt RX, ring buffer |
| 8 | [Timers & PWM](Textbook/Chapter_08_Timers_PWM.md) | ★★★☆☆ | TIM2 periodic interrupt, PWM LED dimming |
| 9 | [ADC (Analog-to-Digital)](Textbook/Chapter_09_ADC.md) | ★★★☆☆ | Single-channel ADC, voltage conversion |
| 10 | [SPI Communication](Textbook/Chapter_10_SPI.md) | ★★★☆☆ | SPI master driver, register read/write |
| 11 | [I2C Communication](Textbook/Chapter_11_I2C.md) | ★★★☆☆ | I2C master driver, repeated start |
| 12 | [DMA (Direct Memory Access)](Textbook/Chapter_12_DMA.md) | ★★★★☆ | DMA UART TX, normal & circular modes |

### Part III: Advanced Protocols
| # | Chapter | Difficulty | Key Topics |
|---|---------|-----------|------------|
| 13 | [CAN Bus](Textbook/Chapter_13_CAN_Bus.md) | ★★★★☆ | bxCAN loopback, TX/RX with filters |
| 14 | [USB (CDC Virtual COM)](Textbook/Chapter_14_USB.md) | ★★★★☆ | USB CDC overview, descriptors, ST library |

### Part IV: System Design
| # | Chapter | Difficulty | Key Topics |
|---|---------|-----------|------------|
| 15 | [Build Systems (Make & CMake)](Textbook/Chapter_15_Build_Systems.md) | ★★★☆☆ | Complete Makefile & CMakeLists.txt |
| 16 | [Memory Layout & Linker Scripts](Textbook/Chapter_16_Memory_Linker.md) | ★★★★☆ | Annotated linker script, sections, startup |
| 17 | [FreeRTOS](Textbook/Chapter_17_FreeRTOS.md) | ★★★★☆ | ★ MILESTONE — Multi-task app, queues, semaphores |
| 18 | [Bootloader Design](Textbook/Chapter_18_Bootloader.md) | ★★★★★ | ★ MILESTONE — UART bootloader, Flash programming, jump-to-app |

### Part V: Professional Skills
| # | Chapter | Difficulty | Key Topics |
|---|---------|-----------|------------|
| 19 | [Debugging & Test Equipment](Textbook/Chapter_19_Debugging.md) | ★★★☆☆ | GDB, Hard Fault handler, oscilloscope, logic analyzer |
| 20 | [Embedded C++](Textbook/Chapter_20_Embedded_Cpp.md) | ★★★★☆ | Classes, constexpr, templates, RAII for firmware |
| 21 | [ARM Assembly Essentials](Textbook/Chapter_21_ARM_Assembly.md) | ★★★★☆ | Thumb-2 instructions, reading disassembly, startup code |
| 22 | [Python for Firmware Engineering](Textbook/Chapter_22_Python_FW.md) | ★★★☆☆ | Serial tools, HIL testing, plotting, build automation |
| 23 | [Other Platforms](Textbook/Chapter_23_Other_Platforms.md) | ★★★☆☆ | AVR, ESP32, PIC, NXP comparison and porting |

### Part VI: Capstone
| # | Chapter | Difficulty | Key Topics |
|---|---------|-----------|------------|
| 24 | [Capstone: IoT Environmental Monitor](Textbook/Chapter_24_Capstone.md) | ★★★★★ | Complete IoT system integrating all chapters |

### Reference
| Section | Description |
|---------|-------------|
| [Appendices](Textbook/Appendices.md) | Pin map, register addresses, clock tree, GCC flags, glossary, interview prep |

---

## How to Use This Textbook

1. **Follow in order** — each chapter builds on the previous
2. **Type every code example** — don't copy-paste
3. **Do the exercises** — they reinforce concepts
4. **Build the capstone** — it's your portfolio project

## Hardware Required

- NUCLEO-F401RE board (~$15)
- USB Micro-B cable
- Breadboard + jumper wires
- Optional: logic analyzer, oscilloscope, sensors (BME280, LDR)
