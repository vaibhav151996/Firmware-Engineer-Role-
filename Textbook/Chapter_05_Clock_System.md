# Chapter 5: Clock System & Power Management
## HSI, HSE, PLL, Bus Prescalers, and SysTick Timer

**Difficulty Level: ★★☆☆☆ Beginner–Intermediate**
**Estimated Time: 2 weeks**

---

## 5.1 Concept Overview

### Why Clocks Matter

Every operation inside a microcontroller is driven by a **clock signal** — a regular oscillating signal that synchronizes all logic. The clock determines:
- **Speed:** How fast the CPU executes instructions
- **Peripheral timing:** UART baud rate, timer resolution, ADC sample rate
- **Power consumption:** Higher clock = more power

```
  Clock Signal:
  
       ┌──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐
       │  │  │  │  │  │  │  │  │  │  │  │
  ─────┘  └──┘  └──┘  └──┘  └──┘  └──┘  └──
  
       |←────→|
       1 clock cycle
       
  At 16 MHz: 1 cycle = 62.5 nanoseconds
  At 84 MHz: 1 cycle = 11.9 nanoseconds
```

### STM32F401 Clock Sources

```
  ┌─────────────────────────────────────────────────────────┐
  │              STM32F401 CLOCK SOURCES                      │
  │                                                          │
  │  ┌──────────────────┐                                    │
  │  │  HSI (16 MHz)    │  Internal RC oscillator             │
  │  │  ± 1% accuracy   │  Available immediately after reset │
  │  │  Default source!  │  No external components needed    │
  │  └────────┬─────────┘                                    │
  │           │                                               │
  │  ┌────────┴─────────┐                                    │
  │  │  HSE (4-26 MHz)  │  External crystal/oscillator       │
  │  │  Typ: 8 MHz      │  High accuracy (20 ppm)           │
  │  │  Nucleo: 8 MHz   │  Needs startup time (~2ms)        │
  │  │  from ST-Link    │                                    │
  │  └────────┬─────────┘                                    │
  │           │                                               │
  │  ┌────────┴─────────┐                                    │
  │  │  PLL             │  Phase-Locked Loop                  │
  │  │  Multiplies HSI  │  Can reach up to 84 MHz            │
  │  │  or HSE to get   │  Configurable M, N, P dividers     │
  │  │  high frequency  │                                    │
  │  └────────┬─────────┘                                    │
  │           │                                               │
  │  ┌────────┴─────────┐                                    │
  │  │  LSI (32 kHz)    │  Low-speed internal                │
  │  │                  │  For watchdog, RTC                  │
  │  └─────────────────┘                                    │
  │                                                          │
  │  ┌─────────────────┐                                     │
  │  │  LSE (32.768kHz)│  Low-speed external crystal         │
  │  │                  │  For precise RTC                   │
  │  └─────────────────┘                                     │
  └──────────────────────────────────────────────────────────┘
```

---

## 5.2 STM32 Hardware Internals

### The Clock Tree

The clock tree distributes the main clock to different parts of the system through prescalers (dividers):

```
  STM32F401 Clock Tree (Simplified):
  
  HSI(16MHz)──┐
              ├──→ [System Clock Switch] ──→ SYSCLK (up to 84 MHz)
  HSE(8MHz) ──┤           │                      │
              │           │                      │
  PLL ────────┘           │              ┌───────┴────────┐
   ↑                      │              │    AHB         │
   │                      │              │  Prescaler     │
  HSI or HSE ─→ PLLM ─→ PLLN ─→ PLLP   │  (/1,2,4...512)│
               (/2-63) (*50-432) (/2,4,  │                │
                         6,8)   └───────┬────────┘
                                         │
                                    HCLK (AHB bus clock)
                                    (= CPU clock, up to 84 MHz)
                                         │
                              ┌──────────┼──────────┐
                              │          │          │
                         ┌────┴────┐┌────┴────┐┌────┴────┐
                         │ APB1    ││ APB2    ││ AHB     │
                         │Prescaler││Prescaler││ Periphs │
                         │(/1,2,4, ││(/1,2,4, ││(GPIO,   │
                         │ 8,16)   ││ 8,16)   ││ DMA)    │
                         └────┬────┘└────┬────┘└─────────┘
                              │          │
                         APB1 Clock  APB2 Clock
                         (≤42 MHz)   (≤84 MHz)
                              │          │
                         ┌────┴────┐┌────┴────┐
                         │UART2-5  ││UART1,6  │
                         │TIM2-5   ││TIM1,9-11│
                         │SPI2,3   ││SPI1,4   │
                         │I2C1-3   ││ADC1     │
                         │PWR      ││SYSCFG   │
                         └─────────┘└─────────┘
```

### PLL Configuration

The PLL takes an input clock and produces a higher frequency:

```
  PLL Formula:
  
  PLL_output = (PLL_input / PLLM) × PLLN / PLLP
  
  Example: HSI → 84 MHz SYSCLK
  
  PLL_input = HSI = 16 MHz
  PLLM = 16  → 16 MHz / 16 = 1 MHz (VCO input)
  PLLN = 336 → 1 MHz × 336 = 336 MHz (VCO output)
  PLLP = 4   → 336 MHz / 4 = 84 MHz (SYSCLK!)
  
  Constraints:
  • VCO input (after PLLM): must be 1-2 MHz (recommend 2 MHz)
  • VCO output (after PLLN): must be 100-432 MHz
  • PLLP output: must not exceed 84 MHz (for F401)
```

### Flash Wait States

When the CPU clock is fast, Flash memory can't keep up. You must configure **wait states**:

```
  Flash Wait States for STM32F401 (at 3.3V):
  
  SYSCLK Range          Wait States (LATENCY)
  ──────────────────    ─────────────────────
  0-30 MHz              0 WS
  30-64 MHz             1 WS
  64-84 MHz             2 WS
  
  IMPORTANT: Set wait states BEFORE increasing the clock!
  If you forget, the CPU tries to read Flash too fast → crash!
```

---

## 5.3 Step-by-Step Implementation

### Configuring 84 MHz from HSI via PLL

```c
/******************************************************************************
 * File:    clock_config.c
 * Brief:   Configure STM32F401 to run at 84 MHz using HSI + PLL
 * 
 * Clock path: HSI(16MHz) → PLL → SYSCLK(84MHz)
 * PLL: 16MHz / 16 × 336 / 4 = 84 MHz
 *****************************************************************************/

#include <stdint.h>

/* RCC Register Definitions */
#define RCC_BASE        0x40023800UL
#define RCC_CR          (*(volatile uint32_t *)(RCC_BASE + 0x00UL))
#define RCC_PLLCFGR     (*(volatile uint32_t *)(RCC_BASE + 0x04UL))
#define RCC_CFGR        (*(volatile uint32_t *)(RCC_BASE + 0x08UL))

/* Flash Register */
#define FLASH_BASE      0x40023C00UL
#define FLASH_ACR       (*(volatile uint32_t *)(FLASH_BASE + 0x00UL))

/* RCC_CR bits */
#define RCC_CR_HSION        (1UL << 0)
#define RCC_CR_HSIRDY       (1UL << 1)
#define RCC_CR_PLLON        (1UL << 24)
#define RCC_CR_PLLRDY       (1UL << 25)

/* RCC_CFGR bits */
#define RCC_CFGR_SW_PLL     (2UL << 0)     /* Select PLL as system clock */
#define RCC_CFGR_SWS_PLL    (2UL << 2)     /* PLL is system clock (status) */
#define RCC_CFGR_PPRE1_DIV2 (4UL << 10)    /* APB1 prescaler = /2 */

/* FLASH_ACR bits */
#define FLASH_ACR_LATENCY_2WS  (2UL << 0)  /* 2 wait states */
#define FLASH_ACR_ICEN          (1UL << 9)  /* Instruction cache enable */
#define FLASH_ACR_DCEN          (1UL << 10) /* Data cache enable */
#define FLASH_ACR_PRFTEN        (1UL << 8)  /* Prefetch enable */

void clock_init_84mhz(void)
{
    /*
     * STEP 1: Ensure HSI is running (it's the default after reset)
     */
    RCC_CR |= RCC_CR_HSION;
    while (!(RCC_CR & RCC_CR_HSIRDY))
    {
        /* Wait for HSI to stabilize */
    }

    /*
     * STEP 2: Configure Flash wait states BEFORE increasing clock
     * At 84 MHz and 3.3V, we need 2 wait states
     * Also enable instruction cache, data cache, and prefetch
     */
    FLASH_ACR = FLASH_ACR_LATENCY_2WS | FLASH_ACR_ICEN 
              | FLASH_ACR_DCEN | FLASH_ACR_PRFTEN;

    /*
     * STEP 3: Configure PLL
     * Source: HSI (16 MHz)
     * PLLM = 16  → VCO input = 16/16 = 1 MHz
     * PLLN = 336 → VCO output = 1 × 336 = 336 MHz
     * PLLP = 4   → PLL output = 336/4 = 84 MHz
     * PLLQ = 7   → USB OTG = 336/7 = 48 MHz (for USB)
     */
    RCC_PLLCFGR = (16UL << 0)    /* PLLM = 16 */
                | (336UL << 6)    /* PLLN = 336 */
                | (1UL << 16)     /* PLLP = 4 (encoding: 01 = /4) */
                | (0UL << 22)     /* PLL source = HSI */
                | (7UL << 24);    /* PLLQ = 7 */

    /*
     * STEP 4: Enable PLL and wait for lock
     */
    RCC_CR |= RCC_CR_PLLON;
    while (!(RCC_CR & RCC_CR_PLLRDY))
    {
        /* Wait for PLL to lock (typically < 1 ms) */
    }

    /*
     * STEP 5: Configure bus prescalers
     * AHB  = SYSCLK / 1 = 84 MHz (GPIO, DMA run at full speed)
     * APB1 = SYSCLK / 2 = 42 MHz (max for APB1 peripherals)
     * APB2 = SYSCLK / 1 = 84 MHz (max for APB2 peripherals)
     */
    RCC_CFGR &= ~(0xFUL << 4);    /* AHB prescaler = /1 (clear HPRE) */
    RCC_CFGR |= RCC_CFGR_PPRE1_DIV2;  /* APB1 = /2 */
    /* APB2 defaults to /1 */

    /*
     * STEP 6: Switch system clock to PLL
     */
    RCC_CFGR &= ~(3UL << 0);      /* Clear SW bits */
    RCC_CFGR |= RCC_CFGR_SW_PLL;  /* Select PLL */

    /* Wait until PLL is used as system clock */
    while ((RCC_CFGR & (3UL << 2)) != RCC_CFGR_SWS_PLL)
    {
        /* Wait for clock switch to complete */
    }

    /* Now running at 84 MHz! */
}
```

### SysTick Timer — Precise Millisecond Delay

With a known clock frequency, we can use the Cortex-M SysTick timer for precise delays:

```c
/******************************************************************************
 * File:    systick.c
 * Brief:   SysTick millisecond delay for STM32F401 at 84 MHz
 *****************************************************************************/

#include <stdint.h>

/* SysTick registers (part of ARM Cortex-M core, not STM32 peripheral) */
#define SYSTICK_BASE    0xE000E010UL
#define SYST_CSR        (*(volatile uint32_t *)(SYSTICK_BASE + 0x00UL))
#define SYST_RVR        (*(volatile uint32_t *)(SYSTICK_BASE + 0x04UL))
#define SYST_CVR        (*(volatile uint32_t *)(SYSTICK_BASE + 0x08UL))

/* SysTick control bits */
#define SYST_CSR_ENABLE     (1UL << 0)
#define SYST_CSR_TICKINT    (1UL << 1)
#define SYST_CSR_CLKSOURCE  (1UL << 2)
#define SYST_CSR_COUNTFLAG  (1UL << 16)

static volatile uint32_t systick_ms = 0;

/*
 * SysTick_Handler — Called every 1 ms by the SysTick interrupt
 * This function name must match exactly what's in the vector table
 */
void SysTick_Handler(void)
{
    systick_ms++;
}

/*
 * systick_init() — Configure SysTick for 1ms interrupts
 *
 * SysTick counts DOWN from the reload value to 0.
 * When it reaches 0, it reloads and (if enabled) triggers an interrupt.
 *
 * For 1ms at 84 MHz:
 *   Reload = 84,000,000 / 1,000 - 1 = 83,999
 *   (Subtract 1 because counting includes 0)
 */
void systick_init(uint32_t sys_clock_hz)
{
    SYST_RVR = (sys_clock_hz / 1000UL) - 1UL;  /* 1ms reload value */
    SYST_CVR = 0UL;                             /* Clear current value */
    SYST_CSR = SYST_CSR_ENABLE                  /* Enable SysTick */
             | SYST_CSR_TICKINT                 /* Enable interrupt */
             | SYST_CSR_CLKSOURCE;              /* Use processor clock */
}

/*
 * delay_ms() — Block for the specified number of milliseconds
 */
void delay_ms(uint32_t ms)
{
    uint32_t start = systick_ms;
    while ((systick_ms - start) < ms)
    {
        /* Wait — the SysTick interrupt increments systick_ms */
    }
}

/*
 * get_tick() — Get current millisecond counter
 */
uint32_t get_tick(void)
{
    return systick_ms;
}
```

---

## 5.4 Practical STM32 Nucleo Example

### LED Blink at 84 MHz with Precise 500ms Timing

```c
#include <stdint.h>
#include "gpio_driver.h"    /* From Chapter 4 */

extern void clock_init_84mhz(void);
extern void systick_init(uint32_t sys_clock_hz);
extern void delay_ms(uint32_t ms);

int main(void)
{
    /* Initialize system clock to 84 MHz */
    clock_init_84mhz();
    
    /* Initialize SysTick for precise delays */
    systick_init(84000000UL);
    
    /* Initialize LED */
    gpio_clock_enable(GPIOA);
    GPIO_Config_t led = {
        .pin = 5, .mode = GPIO_MODE_OUTPUT,
        .otype = GPIO_OTYPE_PUSHPULL,
        .speed = GPIO_SPEED_LOW, .pupd = GPIO_PUPD_NONE, .af = 0
    };
    gpio_init(GPIOA, &led);
    
    while (1)
    {
        gpio_toggle(GPIOA, 5);
        delay_ms(500);    /* Precise 500ms delay! */
    }
}
```

---

## 5.5 Diagrams

### Clock Configuration Sequence

```
  Clock initialization order:
  
  ┌──────────────────┐
  │ 1. HSI is running│  (default after reset)
  │    at 16 MHz     │
  └────────┬─────────┘
           ↓
  ┌──────────────────┐
  │ 2. Set Flash     │  MUST be done before increasing clock!
  │    wait states   │  Otherwise: CPU reads corrupt data
  └────────┬─────────┘
           ↓
  ┌──────────────────┐
  │ 3. Configure PLL │  Set M, N, P, Q dividers
  │    parameters    │  Don't enable yet!
  └────────┬─────────┘
           ↓
  ┌──────────────────┐
  │ 4. Enable PLL    │  Takes ~microseconds to lock
  │    Wait for RDY  │  Poll PLLRDY bit
  └────────┬─────────┘
           ↓
  ┌──────────────────┐
  │ 5. Set bus        │  APB1 must not exceed 42 MHz
  │    prescalers    │  APB2 can run at 84 MHz
  └────────┬─────────┘
           ↓
  ┌──────────────────┐
  │ 6. Switch SYSCLK │  Set SW bits in RCC_CFGR
  │    to PLL        │  Wait for SWS confirmation
  └────────┬─────────┘
           ↓
  ┌──────────────────┐
  │ ✓ Running at     │
  │   84 MHz!        │
  └──────────────────┘
```

---

## 5.6 Common Mistakes & Debugging

| Mistake | Result | Fix |
|---------|--------|-----|
| Forgetting Flash wait states | Hard fault or data corruption | Always set wait states BEFORE increasing clock |
| APB1 exceeding 42 MHz | Peripheral malfunction | Use APB1 prescaler ≥ 2 when SYSCLK > 42 MHz |
| Wrong PLL divider values | VCO out of range, PLL won't lock | Check VCO input (1-2 MHz) and output (100-432 MHz) |
| Switching to PLL before it locks | Unpredictable behavior | Always wait for PLLRDY |
| Not updating UART baud after clock change | Wrong baud rate | Recalculate BRR after changing SYSCLK |

---

## 5.7 Exercises

**Exercise 5.1:** Configure the STM32F401 to run at 48 MHz using HSI + PLL. Calculate the appropriate M, N, P values.

**Exercise 5.2:** Measure the LED blink frequency with an oscilloscope at 16 MHz (default) and 84 MHz (PLL). How much faster is the software delay at 84 MHz?

**Exercise 5.3:** Use the SysTick timer to measure how long a GPIO configuration function takes to execute (in microseconds).

**Exercise 5.4:** Write a function that switches between 16 MHz (HSI direct) and 84 MHz (PLL) at runtime while maintaining correct Flash wait states.

---

## 5.8 Industry & Career Notes

- Clock configuration is one of the first things initialized in any firmware project — get it wrong and **nothing works correctly**
- Low-power products dynamically scale the clock frequency to save battery
- The choice between HSI and HSE depends on cost (HSE requires a crystal) vs accuracy (HSI is ±1%, HSE is ~20ppm)
- When calculating peripheral timing (e.g., UART baud rate), you must know which bus the peripheral is on (APB1 vs APB2) and its prescaler

---

## 5.9 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 5 SUMMARY                         │
  │                                                            │
  │  ✓ STM32F401 has multiple clock sources: HSI, HSE, PLL    │
  │                                                            │
  │  ✓ The PLL multiplies a low-frequency source to get       │
  │    high SYSCLK (up to 84 MHz on F401)                     │
  │                                                            │
  │  ✓ Flash wait states MUST be set before increasing clock   │
  │                                                            │
  │  ✓ Bus prescalers divide SYSCLK for different buses:       │
  │    AHB (GPIO, DMA), APB1 (≤42MHz), APB2 (≤84MHz)         │
  │                                                            │
  │  ✓ SysTick provides precise millisecond timing using       │
  │    Cortex-M core timer with interrupt                      │
  │                                                            │
  │  ✓ Peripheral clocks depend on which bus they're on —     │
  │    critical for baud rate and timer calculations           │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 6 — Interrupts & the NVIC →](./Chapter_06_Interrupts_NVIC.md)*
