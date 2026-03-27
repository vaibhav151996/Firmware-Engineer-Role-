# Chapter 4: GPIO — Register-Level Mastery
## Input, Output, Alternate Functions, and Driver Design

**Difficulty Level: ★★☆☆☆ Beginner**
**Estimated Time: 2 weeks**

---

## 4.1 Concept Overview

### What Is GPIO?

GPIO stands for **General-Purpose Input/Output**. Every pin on the microcontroller that you can program to be either an input or an output is a GPIO pin. GPIOs are the most fundamental peripheral — they are how your firmware physically interacts with the outside world.

```
  GPIO — The Bridge Between Software and Physical World:
  
  ┌─────────────────┐                    ┌─────────────────┐
  │    SOFTWARE      │                    │   HARDWARE      │
  │                  │    GPIO Pins       │                  │
  │  GPIOA->ODR = 1 │ ──── PA5 ───────→ │  LED turns ON   │
  │                  │                    │                  │
  │  val = IDR & 1  │ ←─── PA0 ──────── │  Button pressed │
  │                  │                    │                  │
  │  Configure AF   │ ──── PA2 ───────→ │  UART TX signal │
  └─────────────────┘                    └─────────────────┘
```

### GPIO Modes

Each GPIO pin can operate in one of four modes:

```
  ┌────────────────────────────────────────────────────────┐
  │               GPIO PIN MODES (MODER)                   │
  │                                                        │
  │  ┌──────────────┐  ┌──────────────┐                   │
  │  │ 00: INPUT    │  │ 01: OUTPUT   │                   │
  │  │              │  │              │                   │
  │  │ Read the pin │  │ Drive pin    │                   │
  │  │ state (0/1)  │  │ high or low  │                   │
  │  │              │  │              │                   │
  │  │ Example:     │  │ Example:     │                   │
  │  │ Button read  │  │ LED control  │                   │
  │  └──────────────┘  └──────────────┘                   │
  │                                                        │
  │  ┌──────────────┐  ┌──────────────┐                   │
  │  │ 10: ALT FUNC │  │ 11: ANALOG  │                   │
  │  │              │  │              │                   │
  │  │ Pin is       │  │ Pin is       │                   │
  │  │ controlled   │  │ connected to │                   │
  │  │ by a         │  │ ADC/DAC      │                   │
  │  │ peripheral   │  │              │                   │
  │  │              │  │ Example:     │                   │
  │  │ Example:     │  │ Read sensor  │                   │
  │  │ UART TX/RX   │  │ voltage      │                   │
  │  └──────────────┘  └──────────────┘                   │
  └────────────────────────────────────────────────────────┘
```

### Real Industrial Examples

| Application | GPIO Usage | Mode |
|-------------|------------|------|
| LED indicator | Drive pin high/low | Output |
| Button/switch | Read pin state | Input |
| Serial communication | UART TX/RX | Alternate Function |
| Sensor reading | ADC input | Analog |
| Motor enable | Control MOSFET gate | Output |
| Chip select (SPI) | Select slave device | Output |
| Encoder reading | Count pulses | Input / Alt Func (Timer) |

---

## 4.2 STM32 Hardware Internals

### Complete GPIO Register Set

Each GPIO port (A through H) has the same register layout:

```
  GPIO Register Map (per port):
  
  Offset  Name      Size    Description
  ──────  ────────  ──────  ──────────────────────────────────
  0x00    MODER     32-bit  Mode: Input/Output/AF/Analog (2 bits/pin)
  0x04    OTYPER    32-bit  Output Type: Push-Pull / Open-Drain (1 bit/pin)
  0x08    OSPEEDR   32-bit  Output Speed: Low/Med/High/VHigh (2 bits/pin)
  0x0C    PUPDR     32-bit  Pull-Up / Pull-Down (2 bits/pin)
  0x10    IDR       32-bit  Input Data Register (read-only, 1 bit/pin)
  0x14    ODR       32-bit  Output Data Register (read/write, 1 bit/pin)
  0x18    BSRR      32-bit  Bit Set/Reset Register (write-only)
  0x1C    LCKR      32-bit  Lock Register (lock pin configuration)
  0x20    AFRL      32-bit  Alternate Function Low (pins 0-7, 4 bits/pin)
  0x24    AFRH      32-bit  Alternate Function High (pins 8-15, 4 bits/pin)
```

### GPIO Internal Circuit

```
  Simplified GPIO pin internal circuit:
  
                         VDD (3.3V)
                          │
                     ┌────┴────┐
                     │  Pull-Up│ ← Controlled by PUPDR
                     │ Resistor│
                     └────┬────┘
                          │
  ┌───────────┐     ┌─────┴─────┐     ┌──────────┐
  │  Output   │     │           │     │          │
  │  Data     │────→│           │────→│  PAD     │───→ External Pin
  │  Register │     │  Output   │     │ (IC pin) │
  │  (ODR)    │     │  Driver   │     │          │
  └───────────┘     │           │     └──────────┘
                    │ Push-Pull │          │
  ┌───────────┐     │   or      │     ┌────┴────┐
  │  Input    │     │ Open-Drain│     │ Schmitt │
  │  Data     │←────│           │←────│ Trigger │
  │  Register │     └─────┬─────┘     │ (Input) │
  │  (IDR)    │           │           └─────────┘
  └───────────┘     ┌─────┴─────┐
                    │ Pull-Down │ ← Controlled by PUPDR
                    │ Resistor  │
                    └─────┬─────┘
                          │
                         GND
```

### Output Types: Push-Pull vs Open-Drain

```
  PUSH-PULL (OTYPER = 0):               OPEN-DRAIN (OTYPER = 1):
  
       VDD                                    VDD
        │                                      │
   ┌────┴────┐                            External
   │ P-MOS   │ ← ON when output=1         Pull-up
   │(to VDD) │                            Resistor
   └────┬────┘                                 │
        ├────→ Pin                              ├────→ Pin
   ┌────┴────┐                            ┌────┴────┐
   │ N-MOS   │ ← ON when output=0        │ N-MOS   │ ← ON when output=0
   │(to GND) │                            │(to GND) │
   └────┬────┘                            └────┬────┘
        │                                      │
       GND                                    GND
  
  Push-Pull: Can drive HIGH and LOW              
  Open-Drain: Can only drive LOW (or float)
              Needs external pull-up              
              Used for: I2C, shared buses
```

### Alternate Function Mapping

Each pin can connect to different peripherals via the Alternate Function (AF) system:

```
  PA2 Alternate Function Map (STM32F401):
  
  AF0:  (System)
  AF1:  TIM2_CH3        ← Timer 2, Channel 3
  AF2:  TIM5_CH3        ← Timer 5, Channel 3
  AF3:  TIM9_CH1        ← Timer 9, Channel 1
  AF4:  I2C1_SMBA       ← I2C bus
  AF5:  SPI1_MOSI       ← SPI bus
  AF6:  -
  AF7:  USART2_TX       ← UART transmit (ST-Link VCP!)
  AF8:  -
  AF9:  -
  AF10: -
  AF11: -
  AF12: -
  AF13: -
  AF14: -
  AF15: (EVENTOUT)
  
  To use PA2 as UART TX:
  1. Set MODER to 10 (alternate function)
  2. Set AFR to 7 (AF7 = USART2_TX)
```

---

## 4.3 Step-by-Step Implementation

### Building a GPIO Driver Library

Let's build a reusable GPIO driver — the kind you'd write in a professional project:

```c
/******************************************************************************
 * File:    gpio_driver.h
 * Brief:   GPIO driver for STM32F401 — register-level
 *****************************************************************************/
#ifndef GPIO_DRIVER_H
#define GPIO_DRIVER_H

#include <stdint.h>

/* ========================================================================= */
/*                     GPIO TYPE DEFINITIONS                                 */
/* ========================================================================= */

typedef struct {
    volatile uint32_t MODER;
    volatile uint32_t OTYPER;
    volatile uint32_t OSPEEDR;
    volatile uint32_t PUPDR;
    volatile uint32_t IDR;
    volatile uint32_t ODR;
    volatile uint32_t BSRR;
    volatile uint32_t LCKR;
    volatile uint32_t AFR[2];
} GPIO_TypeDef;

/* Port base addresses */
#define GPIOA   ((GPIO_TypeDef *)0x40020000UL)
#define GPIOB   ((GPIO_TypeDef *)0x40020400UL)
#define GPIOC   ((GPIO_TypeDef *)0x40020800UL)
#define GPIOD   ((GPIO_TypeDef *)0x40020C00UL)
#define GPIOH   ((GPIO_TypeDef *)0x40021C00UL)

/* RCC */
#define RCC_BASE        0x40023800UL
#define RCC_AHB1ENR     (*(volatile uint32_t *)(RCC_BASE + 0x30UL))

/* ========================================================================= */
/*                     CONFIGURATION ENUMS                                   */
/* ========================================================================= */

typedef enum {
    GPIO_MODE_INPUT  = 0U,
    GPIO_MODE_OUTPUT = 1U,
    GPIO_MODE_AF     = 2U,
    GPIO_MODE_ANALOG = 3U
} GPIO_Mode_t;

typedef enum {
    GPIO_OTYPE_PUSHPULL  = 0U,
    GPIO_OTYPE_OPENDRAIN = 1U
} GPIO_OType_t;

typedef enum {
    GPIO_SPEED_LOW    = 0U,
    GPIO_SPEED_MEDIUM = 1U,
    GPIO_SPEED_HIGH   = 2U,
    GPIO_SPEED_VHIGH  = 3U
} GPIO_Speed_t;

typedef enum {
    GPIO_PUPD_NONE     = 0U,
    GPIO_PUPD_PULLUP   = 1U,
    GPIO_PUPD_PULLDOWN = 2U
} GPIO_PuPd_t;

/* Configuration structure */
typedef struct {
    uint8_t      pin;       /* Pin number 0-15 */
    GPIO_Mode_t  mode;      /* Input/Output/AF/Analog */
    GPIO_OType_t otype;     /* Push-pull or Open-drain */
    GPIO_Speed_t speed;     /* Output speed */
    GPIO_PuPd_t  pupd;      /* Pull-up/Pull-down */
    uint8_t      af;        /* Alternate function (0-15) */
} GPIO_Config_t;

/* ========================================================================= */
/*                       FUNCTION PROTOTYPES                                 */
/* ========================================================================= */

void gpio_clock_enable(GPIO_TypeDef *port);
void gpio_init(GPIO_TypeDef *port, const GPIO_Config_t *config);
void gpio_write(GPIO_TypeDef *port, uint8_t pin, uint8_t value);
void gpio_toggle(GPIO_TypeDef *port, uint8_t pin);
uint8_t gpio_read(GPIO_TypeDef *port, uint8_t pin);

#endif /* GPIO_DRIVER_H */
```

```c
/******************************************************************************
 * File:    gpio_driver.c
 * Brief:   GPIO driver implementation for STM32F401
 *****************************************************************************/

#include "gpio_driver.h"

/*
 * gpio_clock_enable() - Enable the clock for a GPIO port
 *
 * On STM32F401, GPIO ports are on the AHB1 bus.
 * The RCC_AHB1ENR register controls their clocks:
 *   Bit 0 = GPIOA, Bit 1 = GPIOB, Bit 2 = GPIOC, etc.
 *
 * We calculate which bit to set based on the port's base address.
 */
void gpio_clock_enable(GPIO_TypeDef *port)
{
    /* Calculate port index from base address:
     * GPIOA = 0x40020000, GPIOB = 0x40020400, etc.
     * Difference between ports = 0x400 = 1024 bytes
     * Index = (address - GPIOA_base) / 0x400
     */
    uint32_t port_index = ((uint32_t)port - 0x40020000UL) / 0x400UL;
    RCC_AHB1ENR |= (1UL << port_index);
}

/*
 * gpio_init() - Configure a GPIO pin
 *
 * This function sets all configuration registers for a single pin.
 * It uses the clear-then-set pattern:
 *   1. Clear the relevant bits (AND with inverted mask)
 *   2. Set the new value (OR with shifted value)
 */
void gpio_init(GPIO_TypeDef *port, const GPIO_Config_t *config)
{
    uint8_t pin = config->pin;
    uint32_t temp;

    /* --- MODER: 2 bits per pin --- */
    temp = port->MODER;
    temp &= ~(3UL << (pin * 2U));                    /* Clear */
    temp |=  ((uint32_t)config->mode << (pin * 2U));  /* Set */
    port->MODER = temp;

    /* --- OTYPER: 1 bit per pin --- */
    temp = port->OTYPER;
    temp &= ~(1UL << pin);                            /* Clear */
    temp |=  ((uint32_t)config->otype << pin);         /* Set */
    port->OTYPER = temp;

    /* --- OSPEEDR: 2 bits per pin --- */
    temp = port->OSPEEDR;
    temp &= ~(3UL << (pin * 2U));
    temp |=  ((uint32_t)config->speed << (pin * 2U));
    port->OSPEEDR = temp;

    /* --- PUPDR: 2 bits per pin --- */
    temp = port->PUPDR;
    temp &= ~(3UL << (pin * 2U));
    temp |=  ((uint32_t)config->pupd << (pin * 2U));
    port->PUPDR = temp;

    /* --- AFR: 4 bits per pin --- */
    if (config->mode == GPIO_MODE_AF)
    {
        uint8_t afr_index = pin / 8U;        /* 0 for pins 0-7, 1 for 8-15 */
        uint8_t afr_shift = (pin % 8U) * 4U; /* 4 bits per pin */

        temp = port->AFR[afr_index];
        temp &= ~(0xFUL << afr_shift);
        temp |=  ((uint32_t)config->af << afr_shift);
        port->AFR[afr_index] = temp;
    }
}

/*
 * gpio_write() - Set a pin HIGH or LOW
 *
 * Uses BSRR for atomic operation (no read-modify-write needed):
 *   Bits [15:0]  → SET corresponding pin (write 1 to set)
 *   Bits [31:16] → RESET corresponding pin (write 1 to reset/clear)
 */
void gpio_write(GPIO_TypeDef *port, uint8_t pin, uint8_t value)
{
    if (value != 0U)
    {
        port->BSRR = (1UL << pin);          /* Set pin (bits [15:0]) */
    }
    else
    {
        port->BSRR = (1UL << (pin + 16U));  /* Reset pin (bits [31:16]) */
    }
}

/*
 * gpio_toggle() - Toggle a pin
 *
 * Uses ODR with XOR for toggle operation.
 * Note: This is a read-modify-write — not interrupt-safe!
 * For interrupt-safe toggle, use BSRR with IDR check.
 */
void gpio_toggle(GPIO_TypeDef *port, uint8_t pin)
{
    port->ODR ^= (1UL << pin);
}

/*
 * gpio_read() - Read the current state of a pin
 *
 * Reads from IDR (Input Data Register), which reflects
 * the actual electrical state of the pin, regardless of
 * whether it's configured as input or output.
 */
uint8_t gpio_read(GPIO_TypeDef *port, uint8_t pin)
{
    uint8_t result;

    if ((port->IDR & (1UL << pin)) != 0U)
    {
        result = 1U;
    }
    else
    {
        result = 0U;
    }

    return result;
}
```

### Using the Driver — LED and Button Example

```c
/******************************************************************************
 * File:    main.c
 * Brief:   LED + Button using GPIO driver
 * Board:   STM32 Nucleo-F401RE
 *          LD2 = PA5, B1 = PC13 (active LOW)
 *****************************************************************************/

#include "gpio_driver.h"

#define LED_PIN    5U    /* PA5 = LD2 */
#define BUTTON_PIN 13U   /* PC13 = B1 (active LOW on Nucleo) */

static void delay(volatile uint32_t count)
{
    while (count > 0U) { count--; }
}

int main(void)
{
    /* Enable clocks for GPIOA and GPIOC */
    gpio_clock_enable(GPIOA);
    gpio_clock_enable(GPIOC);

    /* Configure PA5 as output (LED) */
    GPIO_Config_t led_config = {
        .pin   = LED_PIN,
        .mode  = GPIO_MODE_OUTPUT,
        .otype = GPIO_OTYPE_PUSHPULL,
        .speed = GPIO_SPEED_LOW,
        .pupd  = GPIO_PUPD_NONE,
        .af    = 0U
    };
    gpio_init(GPIOA, &led_config);

    /* Configure PC13 as input with pull-up (Button) */
    GPIO_Config_t btn_config = {
        .pin   = BUTTON_PIN,
        .mode  = GPIO_MODE_INPUT,
        .otype = GPIO_OTYPE_PUSHPULL,  /* Not used for input */
        .speed = GPIO_SPEED_LOW,       /* Not used for input */
        .pupd  = GPIO_PUPD_PULLUP,     /* Pull-up: button is active LOW */
        .af    = 0U
    };
    gpio_init(GPIOC, &btn_config);

    /* Main loop: LED follows button state */
    while (1)
    {
        if (gpio_read(GPIOC, BUTTON_PIN) == 0U)
        {
            /* Button pressed (active LOW) → turn LED ON */
            gpio_write(GPIOA, LED_PIN, 1U);
        }
        else
        {
            /* Button released → turn LED OFF */
            gpio_write(GPIOA, LED_PIN, 0U);
        }
    }

    return 0;
}
```

---

## 4.4 Pin Configuration Decision Flowchart

```
  How to configure a GPIO pin:
  
  ┌─────────────────────────────────────┐
  │  What does the pin need to do?      │
  └──────────────┬──────────────────────┘
                 │
      ┌──────────┼──────────┬──────────────┐
      ↓          ↓          ↓              ↓
  ┌────────┐ ┌────────┐ ┌────────┐    ┌────────┐
  │ Read   │ │ Drive  │ │ Use a  │    │ Read   │
  │ digital│ │ digital│ │ periph │    │ analog │
  │ signal │ │ signal │ │ (UART, │    │ signal │
  │        │ │        │ │ SPI..) │    │        │
  └───┬────┘ └───┬────┘ └───┬────┘    └───┬────┘
      │          │          │              │
      ↓          ↓          ↓              ↓
  MODER=00   MODER=01   MODER=10      MODER=11
  (Input)    (Output)   (Alt Func)    (Analog)
      │          │          │              │
      ↓          ↓          ↓              │
  ┌────────┐ ┌────────┐ ┌────────┐        │
  │ Need   │ │ Drive  │ │ Which  │        │
  │ pull-  │ │ LEDs?  │ │ AF#?   │        │
  │ up/dn? │ │ →PP    │ │ Check  │        │
  │        │ │ I2C?   │ │ data-  │        │
  │ Button │ │ →OD    │ │ sheet  │        │
  │ →PU    │ └────────┘ └────────┘        │
  └────────┘                              │
                                     No pull-up,
                                     no output type
                                     needed
```

---

## 4.5 Common Mistakes & Debugging

### Mistake: Wrong Alternate Function Number

```c
/* WRONG: PA2 as UART TX with AF5 */
config.af = 5;  /* AF5 = SPI1_MOSI, NOT USART2_TX! */

/* CORRECT: PA2 as UART TX with AF7 */
config.af = 7;  /* AF7 = USART2_TX — always check the datasheet! */

/* How to verify: STM32F401 Datasheet, Table 9: Alternate function mapping */
```

### Mistake: Forgetting to Enable Port Clock

If you configure a pin but forget `gpio_clock_enable()`, writes to the GPIO registers are **silently ignored**. The pin will remain in its reset state (usually floating input).

---

## 4.6 Exercises

**Exercise 4.1:** Modify the driver to support port-wide operations: write all 16 pins at once using ODR.

**Exercise 4.2:** Implement a software debounce for the button: read the pin multiple times with a small delay, and only change LED state if the reading is stable.

**Exercise 4.3:** Connect 4 external LEDs to PB0-PB3. Write a "Knight Rider" pattern (LEDs sweep left and right).

**Exercise 4.4:** Implement `gpio_lock()` using the LCKR register. Once locked, verify that pin configuration cannot be changed until the next reset.

**Exercise 4.5:** Read the Nucleo board user manual and identify all pins available on the Arduino-compatible headers. Create a pin map diagram.

---

## 4.7 Industry & Career Notes

- In production firmware, GPIO drivers often include **pin multiplexing conflict detection** — ensuring two peripherals don't try to use the same pin
- Large projects use **pin configuration tables** — arrays of `GPIO_Config_t` that are applied at startup
- Safety-critical systems may **lock** GPIO configurations after startup using the LCKR register to prevent accidental reconfiguration
- In automotive, unused pins are configured as **analog** to minimize EMI and power consumption

---

## 4.8 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 4 SUMMARY                         │
  │                                                            │
  │  ✓ GPIO pins have 4 modes: Input, Output, AF, Analog      │
  │                                                            │
  │  ✓ Output types: Push-Pull (default) or Open-Drain        │
  │                                                            │
  │  ✓ BSRR provides atomic set/reset (no RMW needed)         │
  │                                                            │
  │  ✓ Alternate Functions connect pins to peripherals         │
  │    — always check the datasheet for AF number              │
  │                                                            │
  │  ✓ A good GPIO driver uses structs, enums, and clean       │
  │    functions — making code reusable and readable           │
  │                                                            │
  │  ✓ Pull-up/pull-down resistors prevent floating inputs     │
  │                                                            │
  │  DRIVER FUNCTIONS BUILT:                                   │
  │    gpio_clock_enable()  gpio_init()                        │
  │    gpio_write()  gpio_toggle()  gpio_read()                │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 5 — Clock System & Power Management →](./Chapter_05_Clock_System.md)*
