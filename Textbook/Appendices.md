# Appendices

---

## Appendix A: STM32F401RE Quick Reference

### Pin Map (Nucleo Board)

```
  Nucleo-F401RE Key Pins:
  ┌───────────┬──────────┬─────────────────────────────┐
  │ Pin       │ Function │ Notes                       │
  ├───────────┼──────────┼─────────────────────────────┤
  │ PA0       │ ADC1_IN0 │ ADC input, User analog      │
  │ PA1       │ ADC1_IN1 │ ADC input                   │
  │ PA2       │ USART2_TX│ ST-LINK Virtual COM port    │
  │ PA3       │ USART2_RX│ ST-LINK Virtual COM port    │
  │ PA4       │ SPI1_NSS │ SPI chip select             │
  │ PA5       │ SPI1_SCK │ ★ User LED (LD2)           │
  │ PA6       │ SPI1_MISO│ SPI data in                 │
  │ PA7       │ SPI1_MOSI│ SPI data out                │
  │ PA8       │ TIM1_CH1 │ Timer/PWM                   │
  │ PA9       │ USART1_TX│ Alternative UART            │
  │ PA10      │ USART1_RX│ Alternative UART            │
  │ PA11      │ USB_DM   │ USB Data Minus              │
  │ PA12      │ USB_DP   │ USB Data Plus               │
  │ PA13      │ SWDIO    │ Debug (do not use as GPIO!) │
  │ PA14      │ SWCLK    │ Debug (do not use as GPIO!) │
  │ PB3       │ SWO      │ Trace output                │
  │ PB6       │ I2C1_SCL │ I2C clock (open-drain!)     │
  │ PB7       │ I2C1_SDA │ I2C data (open-drain!)      │
  │ PB8       │ TIM4_CH3 │ Timer/PWM                   │
  │ PB9       │ TIM4_CH4 │ Timer/PWM                   │
  │ PB10      │ I2C2_SCL │ Alternative I2C             │
  │ PC13      │ USER_BTN │ ★ User button (active LOW) │
  └───────────┴──────────┴─────────────────────────────┘
```

### Peripheral Base Addresses

```
  Core Peripherals:
  ┌───────────────────┬──────────────┐
  │ Peripheral        │ Base Address │
  ├───────────────────┼──────────────┤
  │ SysTick           │ 0xE000E010   │
  │ NVIC              │ 0xE000E100   │
  │ SCB               │ 0xE000ED00   │
  │ FPU               │ 0xE000EF30   │
  │ DWT               │ 0xE0001000   │
  │ ITM               │ 0xE0000000   │
  └───────────────────┴──────────────┘
  
  AHB1 Peripherals:
  ┌───────────────────┬──────────────┐
  │ GPIOA             │ 0x40020000   │
  │ GPIOB             │ 0x40020400   │
  │ GPIOC             │ 0x40020800   │
  │ GPIOD             │ 0x40020C00   │
  │ GPIOE             │ 0x40021000   │
  │ GPIOH             │ 0x40021C00   │
  │ RCC               │ 0x40023800   │
  │ FLASH             │ 0x40023C00   │
  │ DMA1              │ 0x40026000   │
  │ DMA2              │ 0x40026400   │
  └───────────────────┴──────────────┘
  
  APB1 Peripherals:
  ┌───────────────────┬──────────────┐
  │ TIM2              │ 0x40000000   │
  │ TIM3              │ 0x40000400   │
  │ TIM4              │ 0x40000800   │
  │ TIM5              │ 0x40000C00   │
  │ WWDG              │ 0x40002C00   │
  │ IWDG              │ 0x40003000   │
  │ SPI2 / I2S2       │ 0x40003800   │
  │ SPI3 / I2S3       │ 0x40003C00   │
  │ USART2            │ 0x40004400   │
  │ I2C1              │ 0x40005400   │
  │ I2C2              │ 0x40005800   │
  │ I2C3              │ 0x40005C00   │
  │ PWR               │ 0x40007000   │
  └───────────────────┴──────────────┘
  
  APB2 Peripherals:
  ┌───────────────────┬──────────────┐
  │ TIM1              │ 0x40010000   │
  │ USART1            │ 0x40011000   │
  │ USART6            │ 0x40011400   │
  │ ADC1              │ 0x40012000   │
  │ SDIO              │ 0x40012C00   │
  │ SPI1 / I2S1       │ 0x40013000   │
  │ SPI4              │ 0x40013400   │
  │ SYSCFG            │ 0x40013800   │
  │ EXTI              │ 0x40013C00   │
  │ TIM9              │ 0x40014000   │
  │ TIM10             │ 0x40014400   │
  │ TIM11             │ 0x40014800   │
  └───────────────────┴──────────────┘
```

### GPIO Register Offsets

```
  Offset  Register  Width   Description
  0x00    MODER     32-bit  Mode (00=Input, 01=Output, 10=AF, 11=Analog)
  0x04    OTYPER    16-bit  Output type (0=Push-Pull, 1=Open-Drain)
  0x08    OSPEEDR   32-bit  Output speed (00=Low, 01=Med, 10=Fast, 11=High)
  0x0C    PUPDR     32-bit  Pull-up/pull-down (00=None, 01=Up, 10=Down)
  0x10    IDR       16-bit  Input data (read-only)
  0x14    ODR       16-bit  Output data
  0x18    BSRR      32-bit  Bit set/reset (lower 16=set, upper 16=reset)
  0x1C    LCKR      32-bit  Lock register
  0x20    AFRL      32-bit  Alternate function low (pins 0-7)
  0x24    AFRH      32-bit  Alternate function high (pins 8-15)
```

### Clock Frequencies (After 84 MHz Configuration)

```
  Clock Tree (Chapter 5 Configuration):
  
  HSI = 16 MHz (internal)
    └── PLL Input (/M=8) = 2 MHz
         └── PLL VCO (*N=168) = 336 MHz
              ├── PLLCLK (/P=4) = 84 MHz  ← SYSCLK
              └── USB CLK (/Q=7) = 48 MHz ← USB OTG FS
  
  SYSCLK = 84 MHz
    ├── AHB  (HCLK)  = 84 MHz (/1)
    │    ├── Cortex System Timer = 84 MHz
    │    ├── FCLK = 84 MHz
    │    └── DMA, GPIO, etc.
    ├── APB1 (PCLK1) = 42 MHz (/2)
    │    ├── Timer clock = 84 MHz (×2 when prescaler > 1)
    │    ├── USART2, I2C1-3, SPI2-3
    │    └── TIM2-5, WWDG, IWDG
    └── APB2 (PCLK2) = 84 MHz (/1)
         ├── Timer clock = 84 MHz
         ├── USART1, USART6, SPI1, SPI4
         ├── ADC1 (max 36 MHz with prescaler)
         └── TIM1, TIM9-11, SYSCFG, EXTI
```

### Flash Wait States

```
  ┌──────────────────────┬─────────────────┐
  │ HCLK Frequency       │ Wait States     │
  ├──────────────────────┼─────────────────┤
  │ 0  — 30 MHz          │ 0 WS            │
  │ 30 — 64 MHz          │ 1 WS            │
  │ 64 — 84 MHz          │ 2 WS            │
  └──────────────────────┴─────────────────┘
  
  Set in FLASH_ACR before increasing clock speed!
  Also enable: Instruction Cache + Data Cache + Prefetch
```

---

## Appendix B: ARM Cortex-M4 Exception Numbers

```
  ┌──────┬───────────────────┬──────────┬──────────────────────────┐
  │ IRQ# │ Exception         │ Priority │ Notes                    │
  ├──────┼───────────────────┼──────────┼──────────────────────────┤
  │ -    │ Reset             │ -3 (max) │ Power on / reset button  │
  │ -14  │ NMI               │ -2       │ Non-Maskable Interrupt   │
  │ -13  │ HardFault         │ -1       │ All faults if not config │
  │ -12  │ MemManage         │ Config   │ MPU violations           │
  │ -11  │ BusFault          │ Config   │ Bus errors               │
  │ -10  │ UsageFault        │ Config   │ Undefined instr, etc.    │
  │ -5   │ SVCall            │ Config   │ SVC instruction (RTOS)   │
  │ -4   │ Debug Monitor     │ Config   │ Debug events             │
  │ -2   │ PendSV            │ Config   │ Context switch (RTOS)    │
  │ -1   │ SysTick           │ Config   │ System timer             │
  │ 0+   │ External IRQs     │ Config   │ Peripheral interrupts    │
  └──────┴───────────────────┴──────────┴──────────────────────────┘
  
  STM32F401 External IRQs (partial):
  ┌──────┬───────────────────────────┐
  │ IRQ  │ Source                    │
  ├──────┼───────────────────────────┤
  │ 6    │ EXTI0                     │
  │ 7    │ EXTI1                     │
  │ 8    │ EXTI2                     │
  │ 9    │ EXTI3                     │
  │ 10   │ EXTI4                     │
  │ 18   │ ADC (global)              │
  │ 23   │ EXTI9_5                   │
  │ 28   │ TIM2                      │
  │ 29   │ TIM3                      │
  │ 30   │ TIM4                      │
  │ 31   │ I2C1_EV                   │
  │ 32   │ I2C1_ER                   │
  │ 35   │ SPI1                      │
  │ 36   │ SPI2                      │
  │ 38   │ USART2                    │
  │ 40   │ EXTI15_10                 │
  │ 47   │ DMA1_Stream0              │
  │ 48   │ DMA1_Stream1              │
  │ ...  │ ...                       │
  │ 52   │ DMA1_Stream5              │
  │ 53   │ DMA1_Stream6              │
  │ 67   │ OTG_FS                    │
  └──────┴───────────────────────────┘
```

---

## Appendix C: GCC Compiler Flags Reference

```
  Essential Flags for ARM Cortex-M4:
  
  Architecture:
    -mcpu=cortex-m4         Target CPU
    -mthumb                 Thumb-2 instruction set
    -mfloat-abi=hard        Hardware FPU
    -mfpu=fpv4-sp-d16       Single-precision FPU
  
  C Standard:
    -std=c11                C11 standard (recommended)
    -std=gnu11              C11 with GNU extensions
  
  Warnings (use ALL of these):
    -Wall                   Basic warnings
    -Wextra                 Extra warnings
    -Werror                 Warnings → errors
    -Wshadow                Variable shadowing
    -Wdouble-promotion      Float → double implicit promotion
    -Wformat=2              printf format checking
    -Wundef                 Undefined macro in #if
    -Wconversion            Type conversion warnings
    -Wno-unused-parameter   Allow unused params (for ISRs)
  
  Optimization:
    -O0                     No optimization (debug)
    -O1                     Basic optimization
    -O2                     Full optimization
    -Os                     Optimize for size (★ recommended)
    -Og                     Optimize for debug (good breakpoints)
    -flto                   Link-time optimization
  
  Debug:
    -g                      Debug symbols (DWARF)
    -g3                     Maximum debug info (includes macros)
    -gdwarf-4               DWARF version 4
  
  Code Generation:
    -ffunction-sections     Each function in its own section
    -fdata-sections         Each variable in its own section
    -fno-common             Don't merge uninitialized globals
    -fno-builtin            Don't use built-in functions
    -fstack-usage           Generate .su files (stack usage)
  
  Linker:
    -Wl,--gc-sections       Remove unused sections (needs -f*-sections)
    -Wl,-Map=output.map     Generate map file
    -Wl,--print-memory-usage  Show Flash/RAM usage
    -T linker_script.ld     Use custom linker script
    --specs=nosys.specs     No system calls (bare-metal)
    --specs=nano.specs      Use newlib-nano (smaller printf)
```

---

## Appendix D: Glossary

```
  ADC     Analog-to-Digital Converter
  AF      Alternate Function (GPIO mode for peripherals)
  AHB     Advanced High-performance Bus
  APB     Advanced Peripheral Bus
  ARM     Advanced RISC Machine
  BLE     Bluetooth Low Energy
  BRR     Baud Rate Register
  BSP     Board Support Package
  BSRR    Bit Set/Reset Register
  CAN     Controller Area Network
  CMSIS   Cortex Microcontroller Software Interface Standard
  CRC     Cyclic Redundancy Check
  DAC     Digital-to-Analog Converter
  DMA     Direct Memory Access
  DSP     Digital Signal Processing
  DWT     Data Watchpoint and Trace
  EEPROM  Electrically Erasable Programmable Read-Only Memory
  ELF     Executable and Linkable Format
  EXTI    External Interrupt
  FIFO    First In, First Out
  FPU     Floating Point Unit
  GPIO    General Purpose Input/Output
  HAL     Hardware Abstraction Layer
  HEX     Intel HEX format (firmware file)
  HIL     Hardware-in-the-Loop (testing)
  HSE     High-Speed External (oscillator)
  HSI     High-Speed Internal (oscillator)
  I2C     Inter-Integrated Circuit
  IDE     Integrated Development Environment
  IDR     Input Data Register
  IRQ     Interrupt Request
  ISR     Interrupt Service Routine
  ITM     Instrumentation Trace Macrocell
  IWDG    Independent Watchdog
  JTAG    Joint Test Action Group (debug interface)
  LDR     Load Register (ARM instruction)
  LED     Light Emitting Diode
  LMA     Load Memory Address
  LSE     Low-Speed External (oscillator, 32.768 kHz)
  LSI     Low-Speed Internal (oscillator)
  MCU     Microcontroller Unit
  MISRA   Motor Industry Software Reliability Association
  MISO    Master In Slave Out (SPI)
  MODER   Mode Register (GPIO)
  MOSI    Master Out Slave In (SPI)
  MPU     Memory Protection Unit
  MSP     Main Stack Pointer
  NVIC    Nested Vectored Interrupt Controller
  ODR     Output Data Register
  OTA     Over-The-Air (firmware update)
  PLL     Phase-Locked Loop
  PSP     Process Stack Pointer
  PUPDR   Pull-Up/Pull-Down Register
  PWM     Pulse Width Modulation
  RAII    Resource Acquisition Is Initialization (C++)
  RAM     Random Access Memory
  RCC     Reset and Clock Control
  ROM     Read-Only Memory
  RTOS    Real-Time Operating System
  SBOM    Software Bill of Materials
  SCB     System Control Block
  SCL     Serial Clock (I2C)
  SDA     Serial Data (I2C)
  SPI     Serial Peripheral Interface
  SRAM    Static Random Access Memory
  STR     Store Register (ARM instruction)
  SVD     System View Description (XML register map)
  SWD     Serial Wire Debug
  SWO     Serial Wire Output (trace)
  TCB     Task Control Block
  TIM     Timer peripheral
  UART    Universal Asynchronous Receiver/Transmitter
  USB     Universal Serial Bus
  VMA     Virtual Memory Address
  VTOR    Vector Table Offset Register
  WDG     Watchdog
  WFI     Wait For Interrupt (ARM instruction)
  WWDG    Window Watchdog
```

---

## Appendix E: Recommended Resources

### Books

```
  ★ Essential Reading:
  1. "The Definitive Guide to ARM Cortex-M3 and Cortex-M4 Processors"
     — Joseph Yiu (THE reference for ARM Cortex-M)
  
  2. "Making Embedded Systems" — Elecia White
     (Practical design patterns for embedded)
  
  3. "Mastering STM32" — Carmine Noviello
     (STM32-specific, very detailed)
  
  4. "Test Driven Development for Embedded C" — James Grenning
     (Professional testing practices)
  
  Optional but Valuable:
  5. "Embedded Systems: Real-Time Operating Systems for ARM Cortex-M"
     — Jonathan Valvano
  
  6. "Computer Organization and Design: ARM Edition" — Patterson & Hennessy
  
  7. "Writing Solid Code" — Steve Maguire
```

### Online Resources

```
  Documentation:
  • ST Reference Manual (RM0368) — THE source of truth for STM32F401
  • ARM Architecture Reference Manual — CPU instruction details
  • FreeRTOS.org — Official RTOS documentation
  • CMSIS Documentation — ARM standard APIs
  
  Communities:
  • STM32 Community Forum — forum.st.com
  • r/embedded (Reddit) — embedded engineering community
  • EEVblog Forum — electronics engineering
  • Stack Overflow [embedded] tag
  
  YouTube Channels:
  • ControllersTech — STM32 tutorials
  • Fastbit Embedded Brain Academy — Cortex-M deep dive
  • Phil's Lab — PCB design + firmware
  • Ben Eater — Computer architecture fundamentals
  
  Tools:
  • Godbolt Compiler Explorer — See assembly output online
  • STM32CubeMX — Visual peripheral configurator
  • KiCad — Free PCB design software
  • PulseView / sigrok — Open-source logic analyzer
```

---

## Appendix F: Interview Preparation

### Common Firmware Interview Questions

```
  Fundamentals:
  Q: What is volatile and when do you use it?
  A: Tells compiler not to optimize away reads/writes. Use for:
     memory-mapped registers, ISR-shared variables, DMA buffers.
  
  Q: Explain the difference between a mutex and a semaphore.
  A: Mutex = binary lock with ownership (only owner can unlock).
     Semaphore = counting, no ownership (any task can signal).
     Mutex has priority inheritance; semaphore does not.
  
  Q: What happens when the MCU resets?
  A: 1) Reads initial SP from 0x00000000 (or aliased Flash)
     2) Reads Reset_Handler address from 0x00000004
     3) Begins executing Reset_Handler
     4) Startup code: copy .data, zero .bss, call main()
  
  Q: How do you debug a Hard Fault?
  A: 1) Implement HardFault_Handler that reads stacked PC and LR
     2) Read CFSR register for fault reason
     3) Use addr2line to find source file and line from PC
     4) Check for: null pointer, stack overflow, alignment, undefined instr
  
  Q: What is DMA and when would you use it?
  A: Direct Memory Access — hardware copies data without CPU.
     Use for: high-throughput UART/SPI, ADC continuous conversion,
     memory-to-memory copy. Frees CPU for computation.
  
  Q: Explain I2C protocol.
  A: Two-wire (SCL + SDA), open-drain, master-slave.
     Start → Address (7-bit) + R/W → ACK → Data → ACK → Stop
     Multiple slaves on same bus. Clock stretching for slow slaves.
  
  Intermediate:
  Q: How would you design a bootloader?
  A: (See Chapter 18 — the entire chapter is your answer!)
  
  Q: How do you handle priority inversion in an RTOS?
  A: Use mutex with priority inheritance. When high-priority task
     blocks on mutex held by low-priority task, the low-priority
     task temporarily inherits the high priority.
  
  Q: What is the difference between polling and interrupt-driven I/O?
  A: Polling: CPU continuously checks status register (wastes cycles).
     Interrupt: CPU does other work, hardware signals when ready.
     Use interrupts for production; polling for simple prototypes.
  
  Advanced:
  Q: How would you measure interrupt latency?
  A: Toggle GPIO in ISR, measure time from event to GPIO change
     with oscilloscope. Typical: 12 cycles (Cortex-M4) + NVIC.
  
  Q: Describe memory-mapped I/O.
  A: Peripherals appear as addresses. Writing to 0x40020018
     (GPIOA->BSRR) sets/clears pins. No special instructions —
     just LDR/STR to those addresses.
  
  Q: How do you prevent race conditions in firmware?
  A: 1) Disable interrupts (critical section)
     2) Use RTOS mutex/semaphore
     3) Use atomic operations (LDREX/STREX on Cortex-M)
     4) Design ISRs to only set flags, process in main loop
```

### Whiteboard Coding Questions

```
  1. "Write a function to set bit N of a register"
     void set_bit(volatile uint32_t *reg, uint8_t n) {
         *reg |= (1UL << n);
     }
  
  2. "Implement a circular buffer"
     (See Chapter 7 — ring buffer for UART)
  
  3. "Write a debounce algorithm for a button"
     (See Chapter 6 — ISR + timer-based debounce)
  
  4. "Implement a simple UART driver"
     (See Chapter 7 — your complete uart_driver.c)
  
  5. "Convert a 12-bit ADC reading to voltage"
     float adc_to_volts(uint16_t adc, float vref) {
         return (float)adc * vref / 4095.0f;
     }
```

---

## Appendix G: Quick Setup Checklist

### New STM32 Project From Scratch

```
  [ ] 1. Install toolchain: arm-none-eabi-gcc
  [ ] 2. Install OpenOCD for flashing
  [ ] 3. Create project directory structure:
         project/
         ├── src/main.c
         ├── include/
         ├── startup/startup_stm32f401.s
         ├── STM32F401RE.ld
         └── Makefile
  
  [ ] 4. Write minimal main.c (LED blink from Chapter 1)
  [ ] 5. Copy startup file from STM32CubeF4 (or write your own)
  [ ] 6. Create linker script (Chapter 16)
  [ ] 7. Create Makefile (Chapter 15)
  [ ] 8. Build: make
  [ ] 9. Flash: make flash (via OpenOCD)
  [ ] 10. Verify LED blinks
  [ ] 11. Set up GDB debugging (Chapter 19)
  [ ] 12. Add UART (Chapter 7) for printf
  [ ] 13. Commit to git, push to GitHub
```

---

# Textbook Complete

```
  ╔═══════════════════════════════════════════════════════════════╗
  ║                                                               ║
  ║   THE COMPLETE FIRMWARE ENGINEERING TEXTBOOK                  ║
  ║                                                               ║
  ║   24 Chapters • Appendices • 500+ Code Examples              ║
  ║                                                               ║
  ║   Target: STM32F401RE Nucleo Board                            ║
  ║   Style:  Bare-Metal, Register-Level                          ║
  ║   Tools:  arm-none-eabi-gcc, OpenOCD, GDB                    ║
  ║                                                               ║
  ║   From "Hello LED" to "IoT Sensor System"                    ║
  ║   From beginner to interview-ready                            ║
  ║                                                               ║
  ║   Build things. Break things. Understand everything.          ║
  ║                                                               ║
  ╚═══════════════════════════════════════════════════════════════╝
```

---

*[← Back to Table of Contents](../README.md)*
