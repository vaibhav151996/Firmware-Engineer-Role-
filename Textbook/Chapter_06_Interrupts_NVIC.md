# Chapter 6: Interrupts & the NVIC
## External Interrupts, Priorities, ISR Design Patterns

**Difficulty Level: ★★☆☆☆ Beginner–Intermediate**
**Estimated Time: 2 weeks**

---

## 6.1 Concept Overview

### What Are Interrupts?

An **interrupt** is a signal that tells the CPU to **immediately stop what it's doing** and execute a specific handler function (called an **ISR** — Interrupt Service Routine). When the ISR finishes, the CPU resumes exactly where it left off.

```
  Without Interrupts (Polling):        With Interrupts:
  
  main loop:                           main loop:
  ┌─────────────────────┐              ┌─────────────────────┐
  │ Check button?       │              │ Do useful work      │
  │ Check UART?         │              │ (sleep if idle)     │
  │ Check timer?        │              └─────────┬───────────┘
  │ Check ADC?          │                        │
  │ Do actual work      │              Button pressed? ──→ ISR runs!
  │ Repeat forever      │              UART byte?     ──→ ISR runs!  
  └─────────────────────┘              Timer expired?  ──→ ISR runs!
  
  Problem: Wastes CPU time             CPU only responds when needed.
  checking things that                 Much more efficient!
  haven't happened yet.
```

### The NVIC (Nested Vectored Interrupt Controller)

The NVIC is the ARM Cortex-M hardware that manages all interrupts:

```
  ┌──────────────────────────────────────────────────┐
  │                      NVIC                         │
  │                                                  │
  │  Features:                                       │
  │  • Up to 240 interrupt sources                   │
  │  • Configurable priorities (0-15 on STM32F4)     │
  │  • Nested: Higher priority can interrupt lower   │
  │  • Hardware stacking: Automatic register save    │
  │  • Tail-chaining: Efficient back-to-back ISRs    │
  │                                                  │
  │  ┌──────────┐    ┌──────────┐    ┌──────────┐   │
  │  │ IRQ #0   │    │ IRQ #1   │    │ IRQ #N   │   │
  │  │Priority:3│    │Priority:1│    │Priority:2│   │
  │  │Enabled:Y │    │Enabled:Y │    │Enabled:N │   │
  │  └────┬─────┘    └────┬─────┘    └──────────┘   │
  │       │               │                          │
  │       └───────┬───────┘                          │
  │               ↓                                  │
  │    ┌─────────────────────┐                       │
  │    │ Priority Resolver   │                       │
  │    │ (highest priority   │                       │
  │    │  wins = LOWEST      │                       │
  │    │  number!)           │                       │
  │    └────────┬────────────┘                       │
  │             ↓                                    │
  │          CPU executes ISR                        │
  └──────────────────────────────────────────────────┘
```

> **CRITICAL NOTE:** On ARM Cortex-M, **lower priority number = higher priority**. Priority 0 is the HIGHEST priority. This is counterintuitive but essential to remember!

---

## 6.2 STM32 Hardware Internals

### Interrupt Vector Table

The vector table is an array of function pointers stored at the beginning of Flash. Each entry tells the CPU which function to call for each interrupt:

```
  Vector Table (beginning of Flash, 0x08000000):
  
  Offset  Vector             Handler Name
  ──────  ─────────────────  ────────────────────────
  0x000   Initial SP         (Stack top address)
  0x004   Reset              Reset_Handler
  0x008   NMI                NMI_Handler
  0x00C   HardFault          HardFault_Handler
  0x010   MemManage          MemManage_Handler
  0x014   BusFault           BusFault_Handler
  0x018   UsageFault         UsageFault_Handler
  ...
  0x038   SVCall             SVC_Handler
  0x03C   DebugMonitor       DebugMon_Handler
  0x040   (Reserved)
  0x044   PendSV             PendSV_Handler
  0x048   SysTick            SysTick_Handler
  ─────── External Interrupts ──────────────────────
  0x04C   IRQ0 (WWDG)        WWDG_IRQHandler
  0x050   IRQ1 (PVD)         PVD_IRQHandler
  ...
  0x098   IRQ19 (EXTI9_5)    EXTI9_5_IRQHandler
  ...
  0x0E0   IRQ37 (USART2)     USART2_IRQHandler
  ...
```

### EXTI — External Interrupt/Event Controller

For GPIO-triggered interrupts, STM32 uses the **EXTI** (External Interrupt) controller:

```
  GPIO Pin ──→ EXTI Line ──→ NVIC ──→ CPU executes ISR
  
  EXTI Line Mapping:
  
  EXTI0  ← PA0 or PB0 or PC0 (select via SYSCFG_EXTICR1)
  EXTI1  ← PA1 or PB1 or PC1
  ...
  EXTI13 ← PA13 or PB13 or PC13  ← User Button (B1) on Nucleo!
  ...
  EXTI15 ← PA15 or PB15 or PC15
  
  Note: Only ONE port per EXTI line!
  You can't use PA0 AND PB0 as interrupts simultaneously.
  
  EXTI Trigger Configuration:
  ┌─────────────────────────┐
  │  EXTI_RTSR (Rising)     │  Pin goes LOW → HIGH
  │  EXTI_FTSR (Falling)    │  Pin goes HIGH → LOW
  │  Both can be enabled     │  Both edges
  └─────────────────────────┘
```

### Interrupt Nesting Example

```
  Time →
  
  Priority 3 (Low):    ████████████████          ████████
  UART ISR                 ↑ Interrupted!         ↑ Resumed
  
  Priority 1 (High):              ██████████████
  Timer ISR                       ↑ Preempts!  ↑ Finishes
  
  main() code:         ████                              ████
                       ↑ Running    ← Both ISRs done →   ↑ Resumed
  
  Sequence:
  1. main() runs
  2. UART interrupt fires → UART ISR starts (priority 3)
  3. Timer interrupt fires → preempts UART ISR (priority 1 < 3)
  4. Timer ISR finishes → UART ISR resumes
  5. UART ISR finishes → main() resumes
```

---

## 6.3 Step-by-Step Implementation

### Button Interrupt on Nucleo-F401RE

```c
/******************************************************************************
 * File:    main.c
 * Brief:   Button interrupt toggles LED on STM32 Nucleo-F401RE
 *          B1 (PC13) triggers EXTI13 interrupt → toggles LD2 (PA5)
 *
 * Registers used:
 *   RCC_AHB1ENR  — Enable GPIOA and GPIOC clocks
 *   RCC_APB2ENR  — Enable SYSCFG clock (for EXTI mux)
 *   SYSCFG_EXTICR4 — Connect EXTI13 to port C
 *   EXTI_IMR     — Unmask EXTI line 13
 *   EXTI_FTSR    — Trigger on falling edge (button press)
 *   EXTI_PR      — Pending register (clear in ISR)
 *   NVIC_ISER1   — Enable EXTI15_10 IRQ in NVIC
 *****************************************************************************/

#include <stdint.h>

/* ========================================================================= */
/*                    PERIPHERAL BASE ADDRESSES                              */
/* ========================================================================= */

/* GPIO */
#define GPIOA_BASE      0x40020000UL
#define GPIOA_MODER     (*(volatile uint32_t *)(GPIOA_BASE + 0x00UL))
#define GPIOA_ODR       (*(volatile uint32_t *)(GPIOA_BASE + 0x14UL))

#define GPIOC_BASE      0x40020800UL
#define GPIOC_MODER     (*(volatile uint32_t *)(GPIOC_BASE + 0x00UL))
#define GPIOC_PUPDR     (*(volatile uint32_t *)(GPIOC_BASE + 0x0CUL))

/* RCC */
#define RCC_BASE        0x40023800UL
#define RCC_AHB1ENR     (*(volatile uint32_t *)(RCC_BASE + 0x30UL))
#define RCC_APB2ENR     (*(volatile uint32_t *)(RCC_BASE + 0x44UL))

/* SYSCFG */
#define SYSCFG_BASE     0x40013800UL
#define SYSCFG_EXTICR4  (*(volatile uint32_t *)(SYSCFG_BASE + 0x14UL))

/* EXTI */
#define EXTI_BASE       0x40013C00UL
#define EXTI_IMR        (*(volatile uint32_t *)(EXTI_BASE + 0x00UL))
#define EXTI_FTSR       (*(volatile uint32_t *)(EXTI_BASE + 0x0CUL))
#define EXTI_PR         (*(volatile uint32_t *)(EXTI_BASE + 0x14UL))

/* NVIC */
#define NVIC_BASE       0xE000E100UL
#define NVIC_ISER1      (*(volatile uint32_t *)(NVIC_BASE + 0x04UL))

/* ========================================================================= */
/*                    INTERRUPT CONFIGURATION                                */
/* ========================================================================= */

/* EXTI15_10_IRQn = IRQ #40
 * NVIC_ISERx: each register handles 32 IRQs
 * IRQ 40 → ISER1 (IRQs 32-63), bit 8 (40-32=8)
 */
#define EXTI15_10_IRQ_BIT   (1UL << 8)

/* ========================================================================= */
/*                    ISR (Interrupt Service Routine)                         */
/* ========================================================================= */

/*
 * EXTI15_10_IRQHandler — Handles EXTI lines 10 through 15
 *
 * This function name MUST match exactly what's in the vector table
 * (defined in startup_stm32f401retx.s)
 *
 * ISR RULES:
 * 1. Keep it SHORT — do minimal work
 * 2. Clear the pending bit — or it fires again immediately!
 * 3. Use volatile for shared variables
 * 4. Don't call blocking functions (delay, printf, etc.)
 */
void EXTI15_10_IRQHandler(void)
{
    /* Check if it was line 13 that triggered */
    if (EXTI_PR & (1UL << 13))
    {
        /* Clear the pending bit by WRITING a 1 to it
         * (This is called "write-1-to-clear" — common in hardware) */
        EXTI_PR = (1UL << 13);

        /* Toggle the LED */
        GPIOA_ODR ^= (1UL << 5);
    }
}

/* ========================================================================= */
/*                           MAIN                                            */
/* ========================================================================= */

int main(void)
{
    /*
     * Step 1: Enable peripheral clocks
     */
    RCC_AHB1ENR |= (1UL << 0);    /* GPIOA clock */
    RCC_AHB1ENR |= (1UL << 2);    /* GPIOC clock */
    RCC_APB2ENR |= (1UL << 14);   /* SYSCFG clock (needed for EXTI mux) */

    /*
     * Step 2: Configure PA5 as output (LED)
     */
    GPIOA_MODER &= ~(3UL << 10);
    GPIOA_MODER |=  (1UL << 10);

    /*
     * Step 3: Configure PC13 as input with pull-up (Button)
     * (Nucleo B1 button is active LOW — pressing connects PC13 to GND)
     */
    GPIOC_MODER &= ~(3UL << 26);  /* Input mode (00) */
    GPIOC_PUPDR &= ~(3UL << 26);
    GPIOC_PUPDR |=  (1UL << 26);  /* Pull-up */

    /*
     * Step 4: Connect EXTI13 to Port C
     * SYSCFG_EXTICR4 controls EXTI12-15 (4 bits each)
     * EXTI13 is bits [7:4]
     * Port C = 0x2
     */
    SYSCFG_EXTICR4 &= ~(0xFUL << 4);
    SYSCFG_EXTICR4 |=  (0x2UL << 4);  /* Select Port C for EXTI13 */

    /*
     * Step 5: Configure EXTI13 for falling edge trigger
     * Button press → PC13 goes from HIGH to LOW (falling edge)
     */
    EXTI_FTSR |= (1UL << 13);     /* Enable falling edge trigger */

    /*
     * Step 6: Unmask EXTI13 (allow it to generate interrupt request)
     */
    EXTI_IMR |= (1UL << 13);

    /*
     * Step 7: Enable the EXTI15_10 interrupt in NVIC
     */
    NVIC_ISER1 = EXTI15_10_IRQ_BIT;

    /*
     * Step 8: Main loop does nothing — CPU is idle
     * All the work happens in the ISR!
     */
    while (1)
    {
        /* CPU can sleep here or do other tasks.
         * The interrupt will wake it up when needed. */
    }

    return 0;
}
```

---

## 6.4 ISR Design Patterns

### Pattern 1: Flag-Based (Recommended)

```c
/* ISR sets a flag, main loop processes it */
volatile uint8_t button_pressed_flag = 0;

void EXTI15_10_IRQHandler(void)
{
    if (EXTI_PR & (1UL << 13))
    {
        EXTI_PR = (1UL << 13);       /* Clear pending */
        button_pressed_flag = 1;      /* Just set the flag */
    }
}

/* main loop */
while (1)
{
    if (button_pressed_flag)
    {
        button_pressed_flag = 0;       /* Clear flag */
        gpio_toggle(GPIOA, 5);        /* Process in main context */
        delay_ms(200);                 /* Debounce (safe here, not in ISR!) */
    }
}
```

### Pattern 2: Ring Buffer (For Data Streams)

```c
/* Used for UART RX, ADC samples, etc. */
#define BUFFER_SIZE 64
volatile uint8_t rx_buffer[BUFFER_SIZE];
volatile uint8_t rx_head = 0;
volatile uint8_t rx_tail = 0;

void USART2_IRQHandler(void)
{
    uint8_t data = USART2_DR;        /* Read received byte */
    uint8_t next = (rx_head + 1) % BUFFER_SIZE;
    if (next != rx_tail)             /* Check if buffer is full */
    {
        rx_buffer[rx_head] = data;
        rx_head = next;
    }
}
```

---

## 6.5 Common Mistakes & Debugging

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Forgetting to clear pending bit | ISR fires continuously | Write 1 to EXTI_PR bit |
| ISR name doesn't match vector table | Default handler (infinite loop) | Check startup.s for exact name |
| Forgetting SYSCFG clock | EXTI doesn't trigger | Enable `RCC_APB2ENR` bit 14 |
| Doing too much work in ISR | System feels sluggish | Use flag pattern instead |
| Missing `volatile` on shared variable | Variable appears stuck | Always use volatile for ISR-shared data |
| Wrong NVIC register/bit | Interrupt never fires | Calculate: register = IRQn/32, bit = IRQn%32 |

---

## 6.6 Exercises

**Exercise 6.1:** Implement a button counter: each press increments a counter, and after 5 presses, toggle the LED.

**Exercise 6.2:** Configure EXTI for both rising and falling edges. Turn LED on when button is pressed, off when released.

**Exercise 6.3:** Set up two interrupts with different priorities: a SysTick at priority 2, and the button at priority 1. Use the debugger to verify nesting works.

**Exercise 6.4:** Implement software debouncing in the ISR using a simple timestamp check (ignore presses within 200ms of the last one).

---

## 6.7 Industry & Career Notes

- In real products, keeping ISRs short is a **hard requirement** — ISR execution time directly affects worst-case response latency
- **Interrupt priority assignment** is a critical design decision, often documented in a project's architecture document
- MISRA C has specific rules about what's allowed in ISRs (e.g., no dynamic allocation, limited function calls)
- For real-time systems, engineers must calculate **worst-case interrupt latency** and prove it meets timing requirements

---

## 6.8 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 6 SUMMARY                         │
  │                                                            │
  │  ✓ Interrupts let the CPU react to events without polling  │
  │                                                            │
  │  ✓ NVIC manages priorities — lower number = higher prio    │
  │                                                            │
  │  ✓ EXTI connects GPIO pins to the interrupt system         │
  │    through SYSCFG multiplexer                              │
  │                                                            │
  │  ✓ ISR rules: keep SHORT, clear pending, use volatile      │
  │    for shared variables, no blocking calls                 │
  │                                                            │
  │  ✓ Flag-based design: ISR sets flag, main loop processes   │
  │                                                            │
  │  ✓ ISR name MUST match the vector table exactly            │
  │                                                            │
  │  CONFIGURATION STEPS for GPIO interrupt:                   │
  │  1. Enable GPIO and SYSCFG clocks                          │
  │  2. Configure GPIO pin as input                            │
  │  3. Map EXTI line to port via SYSCFG_EXTICRx               │
  │  4. Set trigger edge (RTSR/FTSR)                           │
  │  5. Unmask the EXTI line (IMR)                             │
  │  6. Enable in NVIC (ISER)                                  │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 7 — UART Communication →](./Chapter_07_UART.md)*
