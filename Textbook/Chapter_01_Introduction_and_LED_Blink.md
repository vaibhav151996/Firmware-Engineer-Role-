# Chapter 1: Introduction to Microcontrollers and Bare-Metal C
## Writing a Register-Level LED Blink on STM32 Nucleo

**Difficulty Level: ★☆☆☆☆ Beginner**
**Estimated Time: 1–2 weeks**

---

## 1.1 Concept Overview

### What Is a Microcontroller?

A microcontroller (MCU) is a **tiny computer on a single chip**. Unlike your desktop PC, which has a separate CPU, RAM, storage, and I/O devices, a microcontroller integrates all of these into one package:

```
┌─────────────────────────────────────────────────────┐
│                  MICROCONTROLLER (MCU)               │
│                                                     │
│   ┌─────────┐   ┌─────────┐   ┌──────────────┐     │
│   │  CPU    │   │  Flash  │   │    SRAM      │     │
│   │(Cortex-M│   │(Program │   │ (Variables   │     │
│   │  Core)  │   │ Memory) │   │  & Stack)    │     │
│   └────┬────┘   └────┬────┘   └──────┬───────┘     │
│        │              │               │             │
│   ─────┴──────────────┴───────────────┴─────────    │
│                  INTERNAL BUS (AHB/APB)             │
│   ─────┬──────────────┬───────────────┬─────────    │
│        │              │               │             │
│   ┌────┴────┐   ┌────┴────┐   ┌──────┴───────┐     │
│   │  GPIO   │   │  UART   │   │   Timers     │     │
│   │ (Pins)  │   │ (Serial)│   │  (PWM/Count) │     │
│   └────┬────┘   └────┬────┘   └──────┬───────┘     │
│        │              │               │             │
└────────┼──────────────┼───────────────┼─────────────┘
         │              │               │
    Physical Pins   TX/RX Pins     PWM Output Pins
```

### Why Microcontrollers Matter

Microcontrollers are **everywhere**:
- Your washing machine uses an MCU to control motor speed and water temperature
- Your car has 50–100 MCUs controlling the engine, brakes, airbags, seat heaters
- Medical devices like insulin pumps and heart monitors rely on embedded firmware
- Industrial robots, drones, and IoT sensors are all driven by firmware

**Firmware** is the software that runs directly on a microcontroller. Unlike application software (like a web browser), firmware interacts directly and intimately with hardware — toggling pins, reading sensors, controlling motors, and managing communication protocols.

### Why Bare-Metal?

"Bare-metal" means programming the microcontroller **without an operating system**. Your code runs directly on the hardware. There's no Linux, no Windows, no scheduler — just your code and the CPU.

```
Desktop Software Stack:          Bare-Metal Firmware Stack:

┌──────────────────┐             ┌──────────────────┐
│   Application    │             │  YOUR FIRMWARE   │
├──────────────────┤             │   (That's it!)   │
│   Libraries      │             │                  │
├──────────────────┤             │   Directly       │
│ Operating System │             │   controls       │
├──────────────────┤             │   hardware       │
│    Drivers       │             │   registers      │
├──────────────────┤             └────────┬─────────┘
│    Hardware      │                      │
└──────────────────┘             ┌────────┴─────────┐
                                 │    HARDWARE       │
                                 └──────────────────┘
```

This is both terrifying and empowering. You have **complete control** — but you're also responsible for **everything**.

### Real Industrial Examples

| Industry | Application | MCU Used |
|----------|-------------|----------|
| Automotive | Engine Control Unit (ECU) | STM32, Infineon Aurix |
| Medical | Blood glucose monitor | STM32L (low-power) |
| Consumer | Smart thermostat | STM32, ESP32 |
| Industrial | Motor controller | STM32F4, NXP i.MX RT |
| Aerospace | Satellite subsystem | Radiation-hardened ARM |

---

## 1.2 STM32 Hardware Internals

### Meet Your STM32 Nucleo Board

The STM32 Nucleo-F401RE is our primary learning board. Let's understand its anatomy:

```
    ┌──────────────────────────────────────────────┐
    │              STM32 NUCLEO-F401RE             │
    │                                              │
    │   ┌──────────────────────────────────────┐   │
    │   │          ST-LINK DEBUGGER             │   │
    │   │    (Top section of the board)         │   │
    │   │  ┌─────┐                             │   │
    │   │  │ USB │  ← USB Micro-B connector    │   │
    │   │  └─────┘    (Power + Programming)    │   │
    │   │                                      │   │
    │   │  [LD1] COM LED (ST-Link activity)    │   │
    │   └──────────────────────────────────────┘   │
    │               ↕ SWD Connection               │
    │   ┌──────────────────────────────────────┐   │
    │   │         TARGET MCU SECTION            │   │
    │   │                                      │   │
    │   │  STM32F401RET6                       │   │
    │   │  ┌────────────┐                      │   │
    │   │  │ ARM        │  84 MHz max          │   │
    │   │  │ Cortex-M4  │  512 KB Flash        │   │
    │   │  │ + FPU      │  96 KB SRAM          │   │
    │   │  └────────────┘                      │   │
    │   │                                      │   │
    │   │  [B1] USER BUTTON ← PA0 (Blue)       │   │
    │   │  [LD2] USER LED   ← PA5 (Green)      │   │
    │   │                                      │   │
    │   │  Arduino-compatible headers:         │   │
    │   │  ┌─┬─┬─┬─┬─┬─┬─┬─┐                 │   │
    │   │  │D│D│D│D│D│D│D│D│ CN5 (Digital)     │   │
    │   │  └─┴─┴─┴─┴─┴─┴─┴─┘                 │   │
    │   │  ┌─┬─┬─┬─┬─┬─┐                     │   │
    │   │  │A│A│A│A│A│A│ CN8 (Analog)          │   │
    │   │  └─┴─┴─┴─┴─┴─┘                     │   │
    │   │                                      │   │
    │   │  Morpho headers (all MCU pins):      │   │
    │   │  ║║║║║║║║║║║║║║║║║║║║  CN7 (Left)    │   │
    │   │  ║║║║║║║║║║║║║║║║║║║║  CN10 (Right)  │   │
    │   └──────────────────────────────────────┘   │
    └──────────────────────────────────────────────┘
```

### Key Hardware Facts for Nucleo-F401RE

| Feature | Detail |
|---------|--------|
| MCU | STM32F401RET6 |
| Core | ARM Cortex-M4 with FPU |
| Max Clock | 84 MHz |
| Flash | 512 KB |
| SRAM | 96 KB |
| GPIO Ports | A, B, C, D, H |
| User LED (LD2) | Connected to **PA5** (Port A, Pin 5) |
| User Button (B1) | Connected to **PC13** (active LOW) |
| ST-Link UART | USART2 (PA2=TX, PA3=RX) |
| Debug Interface | SWD via integrated ST-Link V2-1 |

### What Is a Register?

A **register** is a specific memory address that controls hardware behavior. When you write a value to a register, the hardware physically changes — a pin goes high, a timer starts counting, a UART begins transmitting.

Think of registers as **control knobs** on the hardware, but instead of turning them physically, you write numbers to specific memory addresses.

```
  Memory Map (Simplified STM32F401):

  0x0000_0000  ┌─────────────────┐
               │    Flash        │  Your program lives here
               │   (512 KB)      │
  0x0800_0000  ├─────────────────┤
               │    ...          │
  0x2000_0000  ├─────────────────┤
               │    SRAM         │  Variables, stack, heap
               │   (96 KB)       │
  0x2001_8000  ├─────────────────┤
               │    ...          │
  0x4000_0000  ├─────────────────┤
               │  Peripherals    │  GPIO, UART, SPI, etc.
               │   (APB1)        │  ← Writing here controls
  0x4001_0000  ├─────────────────┤     real hardware!
               │  Peripherals    │
               │   (APB2)        │
  0x4002_0000  ├─────────────────┤
               │  Peripherals    │
               │   (AHB1)        │  GPIO ports live here
  0x4008_0000  ├─────────────────┤
               │  Peripherals    │
               │   (AHB2)        │
               └─────────────────┘
  0xE000_0000  ┌─────────────────┐
               │ Cortex-M Core   │  NVIC, SysTick, SCB
               │  Peripherals    │
               └─────────────────┘
```

### GPIO Registers We Need for LED Blink

To blink the LED on PA5, we need to control the **GPIOA** peripheral. Here are the essential registers:

```
  GPIOA Base Address: 0x4002_0000

  Offset  Register        Purpose
  ──────  ──────────────  ────────────────────────────────────────
  0x00    GPIOA_MODER     Mode Register: Input/Output/AF/Analog
  0x04    GPIOA_OTYPER    Output Type: Push-pull / Open-drain
  0x08    GPIOA_OSPEEDR   Output Speed: Low/Medium/High/Very High
  0x0C    GPIOA_PUPDR     Pull-up / Pull-down configuration
  0x10    GPIOA_IDR       Input Data Register (read pin state)
  0x14    GPIOA_ODR       Output Data Register (set pin state)
  0x18    GPIOA_BSRR      Bit Set/Reset Register (atomic pin control)
```

#### MODER Register (Mode Register) — 0x4002_0000 + 0x00

Each pin uses **2 bits** in the MODER register:

```
  GPIOA_MODER (32 bits):
  
  Bit:  31 30 29 28 27 26 ... 11 10  9  8  7  6  5  4  3  2  1  0
  Pin:  P15   P14   P13      P5     P4     P3     P2     P1     P0
        ─────┬─────┬─────    ──┬──  ──┬──  ──┬──  ──┬──  ──┬──  ──┬──
  
  Mode values (2 bits per pin):
    00 = Input mode (reset state)
    01 = General purpose output
    10 = Alternate function
    11 = Analog mode
  
  For PA5 as OUTPUT:
    Bits [11:10] = 01
```

#### ODR Register (Output Data Register) — 0x4002_0000 + 0x14

Each pin uses **1 bit**:

```
  GPIOA_ODR (lower 16 bits active):
  
  Bit:  15 14 13 12 11 10  9  8  7  6  5  4  3  2  1  0
  Pin:  P15 P14 P13 P12 P11 P10 P9 P8 P7 P6 P5 P4 P3 P2 P1 P0
                                            ↑
                                         PA5 = LD2
  
  Write 1 to bit 5 → LED ON
  Write 0 to bit 5 → LED OFF
```

### The Clock Gate — Don't Forget!

**Critical concept:** On STM32, peripherals are **disabled by default** to save power. Before you can use GPIOA, you must enable its clock in the **RCC** (Reset and Clock Control) peripheral.

```
  RCC Base Address: 0x4002_3800
  
  RCC_AHB1ENR (AHB1 peripheral clock enable register)
  Offset: 0x30
  
  Bit 0: GPIOAEN — GPIOA clock enable
    0 = GPIOA clock disabled (default after reset)
    1 = GPIOA clock enabled
  
  You MUST set this bit before touching any GPIOA register!
```

This is the **#1 beginner mistake** — trying to configure GPIO without enabling the clock first. Nothing will work, and there will be no error message. The hardware simply ignores your writes to unpowered peripherals.

```
  Sequence to blink LED on PA5:
  
  ┌─────────────────────────┐
  │ 1. Enable GPIOA clock   │  Write bit 0 of RCC_AHB1ENR
  │    (RCC_AHB1ENR)        │
  └───────────┬─────────────┘
              ↓
  ┌─────────────────────────┐
  │ 2. Set PA5 as OUTPUT    │  Write bits [11:10] = 01
  │    (GPIOA_MODER)        │  in GPIOA_MODER
  └───────────┬─────────────┘
              ↓
  ┌─────────────────────────┐
  │ 3. Toggle PA5           │  XOR bit 5 of GPIOA_ODR
  │    (GPIOA_ODR)          │  
  └───────────┬─────────────┘
              ↓
  ┌─────────────────────────┐
  │ 4. Delay                │  Simple busy loop
  └───────────┬─────────────┘
              ↓
         Go to step 3
```

---

## 1.3 Step-by-Step Implementation

### Understanding the Datasheet Approach

Before writing code, let's learn **how a professional firmware engineer thinks**:

1. **Identify the goal:** Toggle PA5 (LED)
2. **Open the Reference Manual:** RM0368 for STM32F401
3. **Find the peripheral:** GPIO chapter
4. **Identify required registers:** MODER, ODR (and RCC for clock)
5. **Calculate the bit positions:** PA5 → bits [11:10] for MODER, bit 5 for ODR
6. **Write the code:** Direct register manipulation

This is **the most important skill** in firmware engineering — reading datasheets and reference manuals to understand hardware, then writing code to control it.

### The Complete Bare-Metal LED Blink Program

Here is the full, production-style bare-metal LED blink. **Type every line yourself** — do not copy-paste.

```c
/******************************************************************************
 * File:    main.c
 * Author:  [Your Name]
 * Date:    [Date]
 * Brief:   Bare-metal LED blink on STM32F401RE Nucleo
 *          No HAL, no libraries — direct register access only
 *
 * Hardware: STM32 Nucleo-F401RE
 *           LD2 (Green LED) connected to PA5
 *
 * Reference: RM0368 - STM32F401 Reference Manual
 *            Section 8.4 - GPIO Registers
 *            Section 6.3 - RCC AHB1 peripheral clock enable register
 *****************************************************************************/

#include <stdint.h>   /* For uint32_t — standard integer types */

/* ========================================================================= */
/*                       REGISTER ADDRESS DEFINITIONS                        */
/* ========================================================================= */

/*
 * These addresses come directly from the STM32F401 Reference Manual (RM0368)
 * and the STM32F401 Datasheet.
 *
 * Memory map:
 *   AHB1 peripherals start at 0x40020000
 *   GPIOA is the first AHB1 peripheral → base = 0x40020000
 *   RCC base = 0x40023800
 */

/* RCC (Reset and Clock Control) registers */
#define RCC_BASE        0x40023800UL
#define RCC_AHB1ENR     (*(volatile uint32_t *)(RCC_BASE + 0x30UL))

/* GPIOA registers */
#define GPIOA_BASE      0x40020000UL
#define GPIOA_MODER     (*(volatile uint32_t *)(GPIOA_BASE + 0x00UL))
#define GPIOA_ODR       (*(volatile uint32_t *)(GPIOA_BASE + 0x14UL))

/* ========================================================================= */
/*                            BIT DEFINITIONS                                */
/* ========================================================================= */

/* RCC_AHB1ENR bit for GPIOA clock enable */
#define RCC_AHB1ENR_GPIOAEN     (1UL << 0)

/* PA5 configuration constants */
#define LED_PIN                 5U
#define MODER5_MASK             (3UL << (LED_PIN * 2))   /* Clear bits [11:10] */
#define MODER5_OUTPUT           (1UL << (LED_PIN * 2))   /* Set to output mode */
#define ODR5                    (1UL << LED_PIN)          /* ODR bit 5 */

/* ========================================================================= */
/*                          DELAY FUNCTION                                   */
/* ========================================================================= */

/*
 * simple_delay() — Software busy-wait delay
 *
 * This is NOT a precise delay function. It simply burns CPU cycles.
 * The actual delay depends on:
 *   - CPU clock frequency (default 16 MHz HSI for F401)
 *   - Compiler optimization level
 *   - Flash wait states
 *
 * In later chapters, we'll use hardware timers for precise delays.
 *
 * The 'volatile' keyword prevents the compiler from optimizing away
 * the loop — without it, the compiler might remove the entire loop
 * since it "does nothing useful" from the compiler's perspective.
 */
void simple_delay(uint32_t count)
{
    /* volatile prevents optimization of this loop */
    volatile uint32_t i;
    for (i = 0; i < count; i++)
    {
        /* Intentionally empty — just burning CPU cycles */
    }
}

/* ========================================================================= */
/*                             MAIN FUNCTION                                 */
/* ========================================================================= */

int main(void)
{
    /* ===================================================================
     * STEP 1: Enable the GPIOA peripheral clock
     * ===================================================================
     * Why: STM32 peripherals are clock-gated (disabled) by default to
     *      save power. We must enable the clock before accessing any
     *      GPIOA register.
     *
     * Register: RCC_AHB1ENR (RCC AHB1 peripheral clock enable register)
     * Ref Manual: RM0368, Section 6.3.10
     *
     * We use |= (OR-equals) to SET bit 0 without disturbing other bits.
     * This is called Read-Modify-Write (RMW):
     *   1. Read the current register value
     *   2. OR it with our bit mask (sets bit 0 to 1)
     *   3. Write the result back
     */
    RCC_AHB1ENR |= RCC_AHB1ENR_GPIOAEN;

    /* ===================================================================
     * STEP 2: Configure PA5 as General-Purpose Output
     * ===================================================================
     * Why: GPIO pins can be input, output, alternate function, or analog.
     *      We need output mode to drive the LED.
     *
     * Register: GPIOA_MODER (GPIO port mode register)
     * Ref Manual: RM0368, Section 8.4.1
     *
     * PA5 uses bits [11:10] of MODER:
     *   00 = Input (default after reset)
     *   01 = General-purpose output  ← We want this
     *   10 = Alternate function
     *   11 = Analog
     *
     * Technique: Clear first, then set
     *   &= ~MASK  → clears the 2 bits (sets them to 00)
     *   |= VALUE  → sets them to our desired value (01)
     *
     * Why not just OR? Because bits might not be 00 after reset
     * (some pins have alternate reset values). Always clear first.
     */
    GPIOA_MODER &= ~MODER5_MASK;    /* Clear bits [11:10] → 00 */
    GPIOA_MODER |=  MODER5_OUTPUT;   /* Set bits [11:10]   → 01 */

    /* ===================================================================
     * STEP 3: Infinite loop — Toggle LED forever
     * ===================================================================
     * In bare-metal firmware, main() must NEVER return.
     * There is no operating system to return to!
     * If main() returns, the CPU will execute random memory → crash.
     *
     * The XOR (^=) operation toggles a bit:
     *   If bit is 0, XOR with 1 makes it 1 (LED ON)
     *   If bit is 1, XOR with 1 makes it 0 (LED OFF)
     *
     * This is cleaner than separate "set" and "clear" operations.
     */
    while (1)   /* Infinite loop — firmware runs forever */
    {
        GPIOA_ODR ^= ODR5;           /* Toggle PA5 (LD2) */
        simple_delay(500000);         /* Wait ~0.5 seconds at 16 MHz */
    }

    /* This line is never reached, but some compilers warn without it */
    return 0;
}
```

### Line-by-Line Breakdown

Let's examine the critical parts in detail:

#### The `volatile` Keyword

```c
#define RCC_AHB1ENR     (*(volatile uint32_t *)(RCC_BASE + 0x30UL))
```

This line does A LOT. Let's break it apart:

```
  (*(volatile uint32_t *)(RCC_BASE + 0x30UL))
   │  │        │          │              │
   │  │        │          │              └─ UL = unsigned long literal
   │  │        │          └─ Address calculation: 0x40023800 + 0x30
   │  │        │                                = 0x40023830
   │  │        └─ Cast to pointer-to-uint32_t
   │  └─ volatile: tells compiler "this memory can change by hardware,
   │               don't optimize reads/writes away!"
   └─ Dereference: treat this address AS a variable we can read/write

  Result: RCC_AHB1ENR behaves like a uint32_t variable located at
          address 0x40023830 in memory — but it's actually a hardware
          register that controls peripheral clocks!
```

**Why `volatile`?** Without it, the compiler might:
- Cache the register value in a CPU register and never re-read it
- Remove "unnecessary" writes that the compiler thinks have no effect
- Reorder reads/writes for "efficiency"

All of these optimizations would **break** hardware interaction. `volatile` tells the compiler: "This memory location has side effects. Always read from and write to the actual address."

#### Bitwise Operations

```
  Understanding the three key bit operations:
  
  1. SET a bit (using OR):
     register |= (1 << bit_number)
     
     Example: Set bit 0 of RCC_AHB1ENR
     Before: 0000 0000 0000 0000 0000 0000 0000 0000
     Mask:   0000 0000 0000 0000 0000 0000 0000 0001  (1 << 0)
     OR:     0000 0000 0000 0000 0000 0000 0000 0001
     Result: Bit 0 is now 1, all other bits unchanged!

  2. CLEAR bits (using AND with NOT):
     register &= ~(mask)
     
     Example: Clear bits [11:10] of GPIOA_MODER
     Before: xxxx xxxx xxxx xxxx xxxx xx?? xxxx xxxx
     Mask:   0000 0000 0000 0000 0000 1100 0000 0000  (3 << 10)
     NOT:    1111 1111 1111 1111 1111 0011 1111 1111  (~mask)
     AND:    xxxx xxxx xxxx xxxx xxxx xx00 xxxx xxxx
     Result: Bits [11:10] are now 00, all others unchanged!

  3. TOGGLE a bit (using XOR):
     register ^= (1 << bit_number)
     
     Example: Toggle bit 5 of GPIOA_ODR
     If bit was 0: 0 XOR 1 = 1 (LED turns ON)
     If bit was 1: 1 XOR 1 = 0 (LED turns OFF)
```

### What Happens at Startup? (The Reset Sequence)

When you power on the Nucleo board or press the reset button, the Cortex-M4 CPU:

```
  Power On / Reset
       │
       ↓
  ┌─────────────────────────────────────────┐
  │ 1. CPU reads address 0x00000000         │
  │    → Gets initial Stack Pointer value   │
  │    (Top of SRAM = 0x20018000 for F401)  │
  └───────────────┬─────────────────────────┘
                  ↓
  ┌─────────────────────────────────────────┐
  │ 2. CPU reads address 0x00000004         │
  │    → Gets Reset Handler address         │
  │    (Points to startup code)             │
  └───────────────┬─────────────────────────┘
                  ↓
  ┌─────────────────────────────────────────┐
  │ 3. Startup code runs:                   │
  │    - Copies initialized data to SRAM    │
  │    - Zeros out BSS section              │
  │    - Calls SystemInit() (optional)      │
  │    - Calls main()                       │
  └───────────────┬─────────────────────────┘
                  ↓
  ┌─────────────────────────────────────────┐
  │ 4. YOUR main() function starts          │
  │    → LED blink code runs here!          │
  └─────────────────────────────────────────┘
```

The startup code and vector table are provided by the STM32CubeIDE project template. In Chapter 21 (ARM Assembly), we'll write our own startup code.

---

## 1.4 Practical STM32 Nucleo Example

### Creating the Project in STM32CubeIDE

Follow these steps exactly:

**Step 1: Create a New Project**
1. Open STM32CubeIDE
2. File → New → STM32 Project
3. In the "Target Selection" tab:
   - Click "Board Selector"
   - Type "NUCLEO-F401RE" in the search box
   - Select it and click "Next"
4. Enter a project name: `01_bare_metal_led_blink`
5. Select "Empty" project (NOT "STM32Cube" — we don't want HAL)
6. Click "Finish"

**Step 2: Understand the Generated Project**
```
  Project structure:
  
  01_bare_metal_led_blink/
  ├── Inc/                     ← Header files go here
  ├── Src/
  │   ├── main.c               ← YOUR CODE GOES HERE
  │   ├── syscalls.c            ← System call stubs
  │   └── sysmem.c              ← Memory management stubs
  ├── Startup/
  │   └── startup_stm32f401retx.s  ← Assembly startup code
  └── STM32F401RETX_FLASH.ld      ← Linker script
```

**Step 3: Replace main.c**
- Delete everything in `Src/main.c`
- Type in the bare-metal LED blink code from Section 1.3 above

**Step 4: Build**
- Click the hammer icon (🔨) or press Ctrl+B
- The console should show: `Build Finished. 0 errors, 0 warnings`

**Step 5: Flash and Run**
1. Connect your Nucleo board via USB
2. Click the green "Run" button (▶) or press F11 for debug mode
3. The green LED (LD2) should start blinking!

### What If It Doesn't Work?

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| LED doesn't blink | Forgot to enable GPIOA clock | Check `RCC_AHB1ENR |= (1 << 0)` |
| LED stays on or off | Wrong MODER bits | Verify bits [11:10] are set to 01 |
| Build errors | Missing `#include <stdint.h>` | Add it at the top |
| Can't flash | USB not connected properly | Reconnect, check device manager |
| Can't flash | ST-Link drivers not installed | Install from st.com |
| LED blinks too fast | Delay value too small | Increase `simple_delay(500000)` |

---

## 1.5 Diagrams — Signal Flow and Timing

### LED Toggle Timing Diagram

```
  Time →
  
  GPIOA_ODR bit 5:
  
        ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐
        │      │      │      │      │      │      │      │
  ──────┘      └──────┘      └──────┘      └──────┘      └──────
  
  LED:   ON     OFF    ON     OFF    ON     OFF    ON     OFF
  
        |←─────→|
         ~0.5 sec
         (delay)
```

### Register Write Sequence

```
  Time →
  
  Step 1: Enable GPIOA Clock
  ─────────────────────────────────────────────────────
  CPU writes RCC_AHB1ENR:
  Before: 0x00000000
  After:  0x00000001  (bit 0 set = GPIOA clock ON)
  
  Step 2: Configure PA5 as Output
  ─────────────────────────────────────────────────────
  CPU writes GPIOA_MODER:
  Before: 0x0C000000  (reset value — PA13,PA14 = SWD)
  After clear: 0x0C000000 & ~(0x00000C00) = 0x0C000000
  After set:   0x0C000000 | 0x00000400  = 0x0C000400
  Bits [11:10] = 01 → PA5 is now output
  
  Step 3 (loop): Toggle PA5
  ─────────────────────────────────────────────────────
  CPU XOR GPIOA_ODR with 0x00000020:
  Cycle 1: 0x00000000 ^ 0x00000020 = 0x00000020 (LED ON)
  Cycle 2: 0x00000020 ^ 0x00000020 = 0x00000000 (LED OFF)
  Cycle 3: 0x00000000 ^ 0x00000020 = 0x00000020 (LED ON)
  ...
```

### Signal Flow: From Code to Photon

```
  ┌────────────┐     ┌──────────────┐     ┌─────────────┐
  │  Your C    │     │   AHB1 Bus   │     │   GPIOA     │
  │  code in   │────→│  (Hardware   │────→│  Peripheral │
  │  main()    │     │   bus)       │     │  Block      │
  └────────────┘     └──────────────┘     └──────┬──────┘
                                                  │
                                                  ↓
                                           ┌──────┴──────┐
                                           │ PA5 Pin     │
                                           │ Output      │
                                           │ Driver      │
                                           └──────┬──────┘
                                                  │
                                           ┌──────┴──────┐
                                           │  330Ω       │
                                           │  Resistor   │
                                           │  (on board)  │
                                           └──────┬──────┘
                                                  │
                                           ┌──────┴──────┐
                                           │   LED       │
                                           │  (Green,    │
                                           │   LD2)      │
                                           └──────┬──────┘
                                                  │
                                                 GND
```

---

## 1.6 Common Mistakes & Debugging

### Mistake #1: Forgetting the Clock Enable

```c
/* WRONG — GPIOA clock is not enabled */
GPIOA_MODER &= ~MODER5_MASK;
GPIOA_MODER |=  MODER5_OUTPUT;
/* This write is SILENTLY IGNORED — the peripheral is unpowered! */

/* CORRECT — Always enable clock first */
RCC_AHB1ENR |= RCC_AHB1ENR_GPIOAEN;   /* Clock ON */
GPIOA_MODER &= ~MODER5_MASK;            /* Now this works */
GPIOA_MODER |=  MODER5_OUTPUT;
```

### Mistake #2: Wrong Bit Position Calculation

```c
/* WRONG — This sets PA2 as output, not PA5! */
GPIOA_MODER |= (1 << 5);
/* Bit 5 in MODER corresponds to MODER2[1], not PA5! */

/* CORRECT — PA5 uses bits [11:10], so shift by (5*2) = 10 */
GPIOA_MODER |= (1 << 10);
/* Or better, use self-documenting macros: */
GPIOA_MODER |= (1UL << (LED_PIN * 2));
```

### Mistake #3: Forgetting `volatile`

```c
/* WRONG — Compiler may optimize away repeated reads/writes */
#define GPIOA_ODR  (*(uint32_t *)(GPIOA_BASE + 0x14UL))

/* The compiler might reason:
 * "This loop writes to the same address repeatedly.
 *  I can optimize by writing only once."
 * Result: LED turns on but never blinks!
 */

/* CORRECT */
#define GPIOA_ODR  (*(volatile uint32_t *)(GPIOA_BASE + 0x14UL))
```

### Mistake #4: Using `int` Instead of `uint32_t`

```c
/* RISKY — 'int' size is platform-dependent */
int *ptr = (int *)(GPIOA_BASE + 0x14);  /* Could be 16-bit on some platforms! */

/* CORRECT — Always use fixed-width types for hardware registers */
volatile uint32_t *ptr = (volatile uint32_t *)(GPIOA_BASE + 0x14UL);
```

### Debugging with ST-Link

**Viewing Registers in STM32CubeIDE:**

1. Start a debug session (F11)
2. Window → Show View → SFR (Special Function Registers)
3. Expand "GPIOA" → Find "MODER" and "ODR"
4. Step through your code and watch the values change
5. Verify: After `GPIOA_MODER |= MODER5_OUTPUT`, bits [11:10] should be 01

**Using Breakpoints:**

1. Double-click the left margin of a line to set a breakpoint
2. When the CPU hits that line, execution pauses
3. Examine:
   - Variables: Hover over variable names
   - Registers: SFR view
   - Memory: Window → Show View → Memory Browser
   - Stack: Debug view shows call stack

**Memory Browser:**
1. Open Memory Browser
2. Type `0x40020000` (GPIOA base)
3. You can see the raw register values in hex
4. Verify that MODER at offset 0x00 shows the expected value

---

## 1.7 Exercises

### Conceptual Questions

1. **Why must we enable the GPIOA clock before configuring GPIO registers?** Explain what happens at the hardware level when the clock is disabled.

2. **If the STM32F401 has a 32-bit data bus, why does the MODER register use 2 bits per pin instead of 1?** What additional modes does this enable?

3. **Why is `volatile` essential for hardware register access?** Give an example of what could go wrong without it.

4. **What would happen if `main()` returned?** Why do firmware programs always have an infinite loop?

5. **The Nucleo-F401RE's default clock is 16 MHz HSI. If you change the delay to `simple_delay(1000000)`, approximately how long would the delay be?** Is this calculation reliable? Why or why not?

### Coding Exercises

**Exercise 1.1 — Change Blink Speed**
Modify the delay value to make the LED blink at these approximate rates:
- a) Very fast (~10 Hz)
- b) Heartbeat pattern (two quick blinks, then a pause)
- c) SOS in Morse code (... --- ...)

**Exercise 1.2 — Use BSRR Instead of ODR**
Rewrite the toggle using the BSRR (Bit Set/Reset Register) at offset 0x18:
- Writing to bits [15:0] SETS the corresponding pin
- Writing to bits [31:16] RESETS the corresponding pin
- Hint: `GPIOA_BSRR = (1 << 5);` to turn ON, `GPIOA_BSRR = (1 << 21);` to turn OFF

**Exercise 1.3 — Read the Button**
PA0 is connected to the user button (B1) on some Nucleo boards (PC13 on others — check your board).
- Configure the button pin as INPUT
- Read `GPIOA_IDR` or `GPIOC_IDR`
- Turn LED on only when button is pressed

**Exercise 1.4 — Second LED**
Connect an external LED to PB0:
- Enable GPIOB clock (bit 1 of RCC_AHB1ENR)
- GPIOB base address: 0x40020400
- Configure PB0 as output and blink it

### Hardware Experiments

**Experiment 1.1 — Measure the Blink**
Use a multimeter set to DC voltage to measure PA5:
- What voltage do you read when the LED is on? (~3.3V)
- What voltage when off? (~0V)
- Can you see the LED flicker on the multimeter?

**Experiment 1.2 — Observe with an Oscilloscope**
Connect an oscilloscope probe to PA5:
- Measure the actual blink frequency
- How does it compare to your calculation?
- What does the waveform look like? (Should be a clean square wave)

---

## 1.8 Industry & Career Notes

### How Professionals Do It

In industry, register-level LED blink isn't the end goal — it's the **foundation**. Professional firmware engineers:

1. **Use this knowledge daily** when debugging: "Why isn't this peripheral working? Let me check the clock gate and pin configuration in the register view."

2. **Choose appropriate abstraction levels**: Sometimes they use HAL (Hardware Abstraction Layer) for productivity, but they always understand what's happening underneath. The HAL is just calling registers for you.

3. **Write driver libraries**: Instead of using the raw register addresses in application code, they write clean driver functions:

```c
/* Professional driver approach (you'll build this in Chapter 4): */
void gpio_init(GPIO_TypeDef *port, uint8_t pin, uint8_t mode);
void gpio_write(GPIO_TypeDef *port, uint8_t pin, uint8_t value);
void gpio_toggle(GPIO_TypeDef *port, uint8_t pin);
```

4. **Follow coding standards**: MISRA C guidelines require, among other things:
   - Use of `UL` suffix on constants to prevent implicit type conversion
   - Explicit `(void)` for functions with no parameters
   - No magic numbers — always use named constants

### Trade-offs in Real Products

| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Bare-metal registers | Full control, smallest code | Verbose, not portable | Learning, ultra-constrained systems |
| CMSIS headers | Readable, still direct access | Vendor-specific | Most professional projects |
| HAL/LL libraries | Rapid development | Code bloat, harder to debug | Prototyping, complex peripherals |
| Arduino | Very fast prototyping | No control, hides everything | Proof-of-concept only |

### Your First Career Prep

After completing this chapter, you can honestly say in an interview:

> "I've written bare-metal firmware on ARM Cortex-M. I understand memory-mapped I/O, the importance of clock gating, why `volatile` is necessary, and how to read reference manuals to configure peripherals at the register level."

That statement alone places you ahead of most junior candidates.

---

## 1.9 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 1 SUMMARY                         │
  │                                                            │
  │  ✓ A microcontroller is a CPU + Memory + Peripherals       │
  │    on a single chip                                        │
  │                                                            │
  │  ✓ STM32 peripherals are memory-mapped — you control       │
  │    hardware by reading/writing specific memory addresses   │
  │                                                            │
  │  ✓ Always enable the peripheral clock (RCC) before         │
  │    accessing any peripheral registers                      │
  │                                                            │
  │  ✓ The volatile keyword is MANDATORY for hardware          │
  │    register access — prevents compiler optimization        │
  │                                                            │
  │  ✓ Bitwise operations (|=, &=~, ^=) are used to           │
  │    set, clear, and toggle individual bits                  │
  │                                                            │
  │  ✓ Bare-metal firmware runs in an infinite loop —          │
  │    there is no OS to return to                             │
  │                                                            │
  │  ✓ The STM32 Nucleo-F401RE has LD2 on PA5 and uses        │
  │    the integrated ST-Link for programming/debugging        │
  │                                                            │
  │  KEY REGISTERS LEARNED:                                    │
  │    RCC_AHB1ENR  — Peripheral clock enable                  │
  │    GPIOA_MODER  — Pin mode (input/output/AF/analog)        │
  │    GPIOA_ODR    — Output data (drive pins high/low)        │
  │    GPIOA_BSRR   — Atomic bit set/reset                    │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 2 — Embedded C Deep Dive →](./Chapter_02_Embedded_C_Deep_Dive.md)*
