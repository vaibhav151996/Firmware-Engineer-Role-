# Chapter 8: Timers & PWM
## Hardware Timers, Output Compare, and Pulse Width Modulation

**Difficulty Level: ★★★☆☆ Intermediate**
**Estimated Time: 2 weeks**

---

## 8.1 Concept Overview

### What Are Hardware Timers?

A hardware timer is a **counter** that increments (or decrements) automatically with every clock cycle (or at a divided rate). Unlike software delays that waste CPU time, hardware timers run **independently** — the CPU is free to do other work.

```
  Software Delay (Chapter 1):          Hardware Timer:
  
  CPU: Count... count... count...      CPU: Start timer → Do other work!
       (CPU is 100% busy doing            Timer: counting in background
        nothing useful!)                   Timer interrupt: Time's up!
  
  ┌─────────────────────────────┐      ┌─────────────────────────────┐
  │ CPU Utilization: 100%       │      │ CPU Utilization: ~1%        │
  │ Can't respond to events     │      │ Responds to events instantly│
  └─────────────────────────────┘      └─────────────────────────────┘
```

### Timer Applications

```
  ┌────────────────────────────────────────────────────┐
  │             TIMER USE CASES                         │
  │                                                    │
  │  ┌─────────────────┐  ┌─────────────────┐         │
  │  │  Periodic       │  │  PWM Generation │         │
  │  │  Interrupts     │  │  (Motor, LED    │         │
  │  │  (1ms tick,     │  │   dimming, servo│         │
  │  │   scheduling)   │  │   control)      │         │
  │  └─────────────────┘  └─────────────────┘         │
  │                                                    │
  │  ┌─────────────────┐  ┌─────────────────┐         │
  │  │  Input Capture  │  │  One-Shot       │         │
  │  │  (Measure pulse │  │  (Timeout,      │         │
  │  │   width, freq)  │  │   debounce)     │         │
  │  └─────────────────┘  └─────────────────┘         │
  │                                                    │
  │  ┌─────────────────┐  ┌─────────────────┐         │
  │  │  Encoder        │  │  Event Counter  │         │
  │  │  (Rotary        │  │  (Count external│         │
  │  │   position)     │  │   pulses)       │         │
  │  └─────────────────┘  └─────────────────┘         │
  └────────────────────────────────────────────────────┘
```

### What Is PWM?

PWM (Pulse Width Modulation) controls the **average power** delivered to a load by rapidly switching between ON and OFF:

```
  PWM Signals at Different Duty Cycles:
  
  25% Duty Cycle (dim LED, slow motor):
  ┌──┐            ┌──┐            ┌──┐
  │  │            │  │            │  │
  ┘  └────────────┘  └────────────┘  └────────────
  |←── Period ──→|
  
  50% Duty Cycle (half brightness, half speed):
  ┌──────┐        ┌──────┐        ┌──────┐
  │      │        │      │        │      │
  ┘      └────────┘      └────────┘      └────────
  
  75% Duty Cycle (bright LED, fast motor):
  ┌──────────────┐┌──────────────┐┌──────────────┐
  │              ││              ││              │
  ┘              └┘              └┘              └
  
  
  PWM Parameters:
  ┌──────────────────────────────────────────┐
  │  Frequency = 1 / Period                  │
  │  Duty Cycle = ON_time / Period × 100%    │
  │                                          │
  │  Example: 1 kHz PWM, 50% duty           │
  │    Period = 1 ms                         │
  │    ON time = 0.5 ms                      │
  │    OFF time = 0.5 ms                     │
  └──────────────────────────────────────────┘
```

---

## 8.2 STM32 Hardware Internals

### STM32F401 Timer Overview

```
  STM32F401 Timers:
  
  Timer    Type         Bits   Bus    Channels   Notes
  ──────   ──────────   ────   ────   ────────   ──────────────────
  TIM1     Advanced     16     APB2   4          Motor control, complementary
  TIM2     General      32     APB1   4          32-bit! Good for long periods
  TIM3     General      16     APB1   4          Standard general purpose
  TIM4     General      16     APB1   4          Standard general purpose
  TIM5     General      32     APB1   4          32-bit
  TIM9     General      16     APB2   2          Simple timer
  TIM10    General      16     APB2   1          Simple timer
  TIM11    General      16     APB2   1          Simple timer
```

### Timer Block Diagram

```
  Timer Internal Architecture:
  
  Timer Clock (APB × 2 if APB prescaler > 1)
       │
       ↓
  ┌──────────────┐
  │  Prescaler   │  Divides clock by (PSC + 1)
  │  (PSC)       │  
  └──────┬───────┘
         │
         ↓  CK_CNT (counter clock)
  ┌──────────────┐
  │   Counter    │  Counts from 0 to ARR
  │   (CNT)      │  Generates UPDATE event when CNT = ARR
  └──────┬───────┘
         │
    ┌────┴────┬────────────┬────────────┐
    │         │            │            │
    ↓         ↓            ↓            ↓
  ┌─────┐  ┌─────┐     ┌─────┐     ┌─────┐
  │ CH1 │  │ CH2 │     │ CH3 │     │ CH4 │
  │CCR1 │  │CCR2 │     │CCR3 │     │CCR4 │
  └──┬──┘  └──┬──┘     └──┬──┘     └──┬──┘
     │        │            │            │
     ↓        ↓            ↓            ↓
  Output   Output       Output       Output
  Compare  Compare      Compare      Compare
  → Pin    → Pin        → Pin        → Pin
  
  
  KEY REGISTERS:
  
  PSC (Prescaler):
    Divides the timer clock
    CK_CNT = TimerClock / (PSC + 1)
  
  ARR (Auto-Reload Register):
    Counter counts from 0 to ARR, then resets to 0
    Period = (ARR + 1) × (1 / CK_CNT)
  
  CCRx (Capture/Compare Register for channel x):
    In PWM mode: output is HIGH when CNT < CCRx
                 output is LOW when CNT >= CCRx
    Duty cycle = CCRx / (ARR + 1)
```

### PWM Generation Timing

```
  PWM Mode 1 (CNT < CCR → output HIGH):
  
  ARR = 999     ┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈
                                    /|            /|
  CCR = 500   ┈┈┈┈┈┈┈┈┈┈┈┈┈ / |┈┈┈┈┈┈┈┈┈┈┈ / |┈┈┈┈
                          /   |          /   |
                       /      |       /      |
  CNT:              /         |    /         |
                 /            | /            |
  0           /               |/             |/
              ←── Period ────→
  
  Output:
  ┌─────────────┐             ┌─────────────┐
  │  HIGH       │             │  HIGH       │
  │  (CNT<CCR)  │   LOW      │  (CNT<CCR)  │
  └─────────────┘─────────────└─────────────┘
                  (CNT>=CCR)
  
  Duty Cycle = CCR / (ARR + 1) = 500 / 1000 = 50%
```

---

## 8.3 Step-by-Step Implementation

### Basic Timer with Interrupt (Periodic 1-second blink)

```c
/******************************************************************************
 * File:    timer_basic.c
 * Brief:   TIM2 generates 1-second periodic interrupt to blink LED
 * Board:   Nucleo-F401RE (16 MHz HSI default)
 *
 * Timer clock = APB1 clock = 16 MHz (when APB1 prescaler = 1)
 * PSC = 15999 → CK_CNT = 16MHz / 16000 = 1 kHz (1ms period)
 * ARR = 999   → Update every 1000 counts = 1 second
 *****************************************************************************/

#include <stdint.h>

/* RCC */
#define RCC_BASE        0x40023800UL
#define RCC_AHB1ENR     (*(volatile uint32_t *)(RCC_BASE + 0x30UL))
#define RCC_APB1ENR     (*(volatile uint32_t *)(RCC_BASE + 0x40UL))

/* GPIO */
#define GPIOA_BASE      0x40020000UL
#define GPIOA_MODER     (*(volatile uint32_t *)(GPIOA_BASE + 0x00UL))
#define GPIOA_ODR       (*(volatile uint32_t *)(GPIOA_BASE + 0x14UL))

/* TIM2 Registers */
#define TIM2_BASE       0x40000000UL
#define TIM2_CR1        (*(volatile uint32_t *)(TIM2_BASE + 0x00UL))
#define TIM2_DIER       (*(volatile uint32_t *)(TIM2_BASE + 0x0CUL))
#define TIM2_SR         (*(volatile uint32_t *)(TIM2_BASE + 0x10UL))
#define TIM2_CNT        (*(volatile uint32_t *)(TIM2_BASE + 0x24UL))
#define TIM2_PSC        (*(volatile uint32_t *)(TIM2_BASE + 0x28UL))
#define TIM2_ARR        (*(volatile uint32_t *)(TIM2_BASE + 0x2CUL))

/* NVIC */
#define NVIC_ISER0      (*(volatile uint32_t *)(0xE000E100UL))
#define TIM2_IRQn       28

void TIM2_IRQHandler(void)
{
    if (TIM2_SR & (1UL << 0))      /* Update interrupt flag */
    {
        TIM2_SR &= ~(1UL << 0);    /* Clear flag */
        GPIOA_ODR ^= (1UL << 5);   /* Toggle LED */
    }
}

int main(void)
{
    /* Enable clocks */
    RCC_AHB1ENR |= (1UL << 0);    /* GPIOA */
    RCC_APB1ENR |= (1UL << 0);    /* TIM2 */

    /* Configure PA5 as output */
    GPIOA_MODER &= ~(3UL << 10);
    GPIOA_MODER |=  (1UL << 10);

    /* Configure TIM2 */
    TIM2_PSC = 15999UL;            /* 16 MHz / 16000 = 1 kHz */
    TIM2_ARR = 999UL;              /* 1 kHz / 1000 = 1 Hz (1 second) */
    TIM2_DIER |= (1UL << 0);      /* Enable update interrupt */
    TIM2_CR1 |= (1UL << 0);       /* Start timer */

    /* Enable TIM2 interrupt in NVIC */
    NVIC_ISER0 = (1UL << TIM2_IRQn);

    while (1)
    {
        /* CPU is free! Timer handles the blinking. */
    }

    return 0;
}
```

### PWM Output on PA5 (LED Dimming)

```c
/******************************************************************************
 * File:    pwm_led.c
 * Brief:   PWM on TIM2 CH1 (PA5) — LED brightness control
 * Board:   Nucleo-F401RE
 *
 * PA5 is TIM2_CH1 (AF1)
 * PWM frequency: 1 kHz
 * Duty cycle: variable (for LED dimming)
 *****************************************************************************/

#include <stdint.h>

/* Register definitions (abbreviated — use structs in real code) */
#define RCC_AHB1ENR     (*(volatile uint32_t *)0x40023830UL)
#define RCC_APB1ENR     (*(volatile uint32_t *)0x40023840UL)

#define GPIOA_MODER     (*(volatile uint32_t *)0x40020000UL)
#define GPIOA_AFRL      (*(volatile uint32_t *)0x40020020UL)

#define TIM2_CR1        (*(volatile uint32_t *)0x40000000UL)
#define TIM2_CCMR1      (*(volatile uint32_t *)0x40000018UL)
#define TIM2_CCER       (*(volatile uint32_t *)0x40000020UL)
#define TIM2_PSC        (*(volatile uint32_t *)0x40000028UL)
#define TIM2_ARR        (*(volatile uint32_t *)0x4000002CUL)
#define TIM2_CCR1       (*(volatile uint32_t *)0x40000034UL)

static void delay(volatile uint32_t cnt) { while(cnt--); }

int main(void)
{
    /* Enable clocks */
    RCC_AHB1ENR |= (1UL << 0);   /* GPIOA */
    RCC_APB1ENR |= (1UL << 0);   /* TIM2 */

    /* PA5 as AF1 (TIM2_CH1) */
    GPIOA_MODER &= ~(3UL << 10);
    GPIOA_MODER |=  (2UL << 10);  /* Alternate function */
    GPIOA_AFRL  &= ~(0xFUL << 20);
    GPIOA_AFRL  |=  (1UL << 20);  /* AF1 = TIM2 */

    /* Configure TIM2 for PWM */
    TIM2_PSC = 15UL;               /* 16 MHz / 16 = 1 MHz */
    TIM2_ARR = 999UL;              /* 1 MHz / 1000 = 1 kHz PWM */

    /* Channel 1: PWM Mode 1 */
    TIM2_CCMR1 &= ~(7UL << 4);    /* Clear OC1M bits */
    TIM2_CCMR1 |=  (6UL << 4);    /* OC1M = 110 (PWM mode 1) */
    TIM2_CCMR1 |=  (1UL << 3);    /* OC1PE: preload enable */

    /* Enable CH1 output */
    TIM2_CCER |= (1UL << 0);      /* CC1E: enable channel 1 */

    /* Set initial duty cycle to 0% */
    TIM2_CCR1 = 0UL;

    /* Start timer */
    TIM2_CR1 |= (1UL << 0);

    /* Breathing LED effect */
    while (1)
    {
        /* Fade in */
        for (uint32_t i = 0; i <= 999; i += 5)
        {
            TIM2_CCR1 = i;
            delay(5000);
        }
        /* Fade out */
        for (uint32_t i = 999; i > 0; i -= 5)
        {
            TIM2_CCR1 = i;
            delay(5000);
        }
    }

    return 0;
}
```

---

## 8.4 Common Mistakes & Debugging

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing AF configuration | Pin doesn't output PWM | PA5 needs MODER=AF, AFR=AF1 for TIM2 |
| PSC or ARR off by one | Wrong frequency | Remember: divides by (PSC+1) and counts to (ARR+1) |
| Not enabling channel in CCER | No output | Set CC1E bit |
| Not enabling preload | Glitches when updating CCR | Set OC1PE in CCMR1 |
| Timer clock assumption wrong | Wrong frequency | APB1 timer clock = APB1 × 2 if APB1 prescaler > 1 |

---

## 8.5 Exercises

**Exercise 8.1:** Create a 50 Hz servo control signal (20 ms period). Write a function that takes an angle (0°–180°) and sets the appropriate pulse width (1ms–2ms).

**Exercise 8.2:** Use TIM3 to measure the width of a button press (input capture mode). Display the result in milliseconds via UART.

**Exercise 8.3:** Generate two PWM signals at different frequencies on two different channels. Use the oscilloscope to verify.

**Exercise 8.4:** Implement "musical tones" — use a timer to generate square waves at specific frequencies to play a simple melody through a buzzer.

---

## 8.6 Industry & Career Notes

- **Motor control** (BLDC, stepper) heavily relies on advanced timer features like complementary PWM, dead-time generation, and break inputs
- PWM frequency selection depends on the load: LEDs work at >100 Hz, motors at >20 kHz (above hearing), switching regulators at >100 kHz
- In automotive, timer-based RPM measurement is critical for engine control
- Power electronics use timers with sub-microsecond precision

---

## 8.7 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 8 SUMMARY                         │
  │                                                            │
  │  ✓ Hardware timers count autonomously — CPU is free        │
  │                                                            │
  │  ✓ Frequency = TimerClock / ((PSC+1) × (ARR+1))           │
  │                                                            │
  │  ✓ PWM duty cycle = CCRx / (ARR + 1) × 100%               │
  │                                                            │
  │  ✓ PWM mode configured via CCMR, CCER, CCR registers      │
  │                                                            │
  │  ✓ Timer interrupt via DIER + NVIC for periodic events     │
  │                                                            │
  │  ✓ GPIO must be in AF mode with correct AF number          │
  │    for timer output                                        │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 9 — ADC: Analog to Digital Conversion →](./Chapter_09_ADC.md)*
