# Chapter 2: Embedded C Deep Dive
## Memory, Pointers, Volatile, Bitwise Operations, and MISRA Mindset

**Difficulty Level: ★★☆☆☆ Beginner**
**Estimated Time: 2–3 weeks**

---

## 2.1 Concept Overview

### Why Embedded C Is Different

Standard C taught in university focuses on algorithms, data structures, and application logic. **Embedded C** is a specialized discipline where your code directly controls hardware. The same language — but a completely different world.

```
  Standard C (Desktop):                Embedded C (Firmware):
  
  ┌───────────────────────┐            ┌───────────────────────┐
  │ - Heap is "infinite"  │            │ - 96 KB SRAM total    │
  │ - malloc() freely     │            │ - malloc() is BANNED  │
  │ - printf() to console │            │ - No console exists   │
  │ - OS handles I/O      │            │ - YOU handle I/O      │
  │ - int is always 32b   │            │ - int could be 16b    │
  │ - Memory is uniform   │            │ - Flash ≠ RAM ≠ I/O   │
  │ - Segfault? OS saves  │            │ - Hard fault? Crash.  │
  │   you with an error   │            │   Silence. Nothing.   │
  └───────────────────────┘            └───────────────────────┘
```

In this chapter, we'll master the C language features that firmware engineers use **every single day**.

### What This Chapter Covers

| Topic | Why It Matters in Firmware |
|-------|---------------------------|
| Fixed-width integers | Registers are exactly 32 bits — `int` is not guaranteed |
| Pointers | Memory-mapped I/O is all pointers |
| `volatile` | Without it, the compiler breaks your hardware code |
| Bitwise operations | Every register is manipulated bit-by-bit |
| Structs & unions | Peripheral register blocks map to structs |
| `const` and `static` | Flash vs. RAM, scope control |
| Function pointers | Interrupt vectors, callbacks |
| MISRA C mindset | Industry coding standards |

### Real Industrial Examples

- Every STM32 HAL library uses `volatile` structs for peripheral access
- Automotive ECUs follow MISRA C strictly — violating it can mean legal liability
- Safety-critical medical devices ban `malloc()` and recursion entirely

---

## 2.2 STM32 Hardware Internals

### Memory Layout of an STM32 Program

When your firmware runs on the STM32, memory is divided into distinct regions:

```
  STM32F401 Memory Map for Firmware:
  
  Address Range          Region        Contents
  ─────────────────────  ──────────    ──────────────────────────────
  0x0800_0000-0x0807_FFFF  FLASH       Your compiled code (.text)
                                       Read-only data (.rodata)
                                       Initial values for globals (.data init)
  
  0x2000_0000-0x2001_7FFF  SRAM        Initialized globals (.data)
                                       Zero-initialized globals (.bss)
                                       Heap (if used — avoid!)
                                       Stack (grows downward from top)
  
  ┌────────────────────────────────────────────────────┐
  │                      SRAM Layout                    │
  │                                                    │
  │  0x2000_0000  ┌──────────────────┐                 │
  │               │  .data section   │ Initialized     │
  │               │  (globals with   │ variables       │
  │               │   initial values)│                 │
  │               ├──────────────────┤                 │
  │               │  .bss section    │ Zero-init       │
  │               │  (uninitialized  │ globals         │
  │               │   globals)       │                 │
  │               ├──────────────────┤                 │
  │               │                  │                 │
  │               │  Heap ↓          │ Dynamic alloc   │
  │               │  (grows down)    │ (AVOID IN       │
  │               │                  │  EMBEDDED!)     │
  │               │                  │                 │
  │               │  Stack ↑         │ Local vars,     │
  │               │  (grows up      │ return addrs    │
  │               │   toward heap)   │                 │
  │  0x2001_7FFF  └──────────────────┘                 │
  └────────────────────────────────────────────────────┘
```

### How Variables Map to Memory Sections

```c
/* These end up in different memory regions: */

const uint32_t FIRMWARE_VERSION = 0x0100;  /* .rodata → FLASH (read-only) */

uint32_t sensor_reading = 42;              /* .data → SRAM (initialized) */

uint32_t error_count;                      /* .bss → SRAM (zero-initialized) */

static uint32_t module_state = 0;          /* .bss → SRAM (file-scoped) */

void foo(void) {
    uint32_t local_var = 10;               /* Stack → SRAM (temporary) */
    static uint32_t call_count = 0;        /* .data → SRAM (persists!) */
    call_count++;
}
```

---

## 2.3 Step-by-Step Implementation

### 2.3.1 Fixed-Width Integer Types

**The Problem:** Standard C does not guarantee the size of `int`, `short`, or `long`. On a 16-bit MCU, `int` might be 16 bits. On a 64-bit desktop, it might be 64 bits. Registers are **exactly** 32 bits — using `int` is dangerous.

**The Solution:** `<stdint.h>` provides fixed-width types:

```c
#include <stdint.h>

/* Exact-width types — use these for ALL register/hardware code */
uint8_t   byte_val;      /*  8 bits, unsigned (0 to 255) */
uint16_t  half_val;      /* 16 bits, unsigned (0 to 65535) */
uint32_t  word_val;      /* 32 bits, unsigned (0 to 4294967295) */
int8_t    signed_byte;   /*  8 bits, signed (-128 to 127) */
int16_t   signed_half;   /* 16 bits, signed (-32768 to 32767) */
int32_t   signed_word;   /* 32 bits, signed */

/* Boolean (use stdbool.h or define your own) */
#include <stdbool.h>
bool led_on = false;
```

**Rule:** In embedded code, **never use `int` for hardware interaction**. Always use `uint32_t`, `uint16_t`, etc.

### 2.3.2 Pointers and Memory-Mapped I/O

Pointers are the **foundation** of all firmware programming. Every register access is essentially a pointer dereference.

```c
/* A pointer stores a MEMORY ADDRESS */
uint32_t value = 42;
uint32_t *ptr = &value;     /* ptr holds the address of 'value' */
*ptr = 100;                 /* Dereference: write 100 to that address */

/* In firmware, we KNOW the address (from the datasheet): */
uint32_t *gpio_odr = (uint32_t *)0x40020014;  /* GPIOA_ODR address */
*gpio_odr = 0x00000020;    /* Write to the register → sets PA5 high */
```

**Pointer arithmetic in firmware:**

```
  Pointer Arithmetic Visualization:
  
  uint32_t *p = (uint32_t *)0x40020000;  // Points to GPIOA_MODER
  
  p + 0 → 0x40020000  (GPIOA_MODER)    Each increment
  p + 1 → 0x40020004  (GPIOA_OTYPER)   adds sizeof(uint32_t)
  p + 2 → 0x40020008  (GPIOA_OSPEEDR)  = 4 bytes
  p + 3 → 0x4002000C  (GPIOA_PUPDR)
  p + 4 → 0x40020010  (GPIOA_IDR)
  p + 5 → 0x40020014  (GPIOA_ODR)      ← 5 * 4 = 20 = 0x14
```

### 2.3.3 The `volatile` Keyword — In Depth

`volatile` is arguably the **most important keyword** in embedded C. Let's understand it completely.

**What the compiler normally does (optimization):**

```c
/* Without volatile, the compiler sees this: */
uint32_t *reg = (uint32_t *)0x40020014;

*reg = 1;    /* Write 1 */
*reg = 2;    /* Write 2 */
*reg = 3;    /* Write 3 */

/* Compiler thinks: "Only the last write matters. 
   I'll optimize away the first two writes."
   Generated code: just writes 3 once. */

/* But for a UART data register, each write sends a different byte!
   Removing the first two writes loses data! */
```

**With volatile:**

```c
volatile uint32_t *reg = (volatile uint32_t *)0x40020014;

*reg = 1;    /* Compiler MUST write 1 to hardware */
*reg = 2;    /* Compiler MUST write 2 to hardware */
*reg = 3;    /* Compiler MUST write 3 to hardware */
/* All three writes are preserved! */
```

**Volatile also prevents read optimization:**

```c
/* Without volatile — compiler may read once and cache the value */
uint32_t *status = (uint32_t *)0x40020010;
while (*status == 0) {
    /* Compiler: "I already know *status is 0.
       I don't need to re-read it. Infinite loop!" */
}
/* BUG: This becomes an infinite loop even when hardware changes the value */

/* With volatile — compiler re-reads every time */
volatile uint32_t *status = (volatile uint32_t *)0x40020010;
while (*status == 0) {
    /* Compiler: "It's volatile. I must re-read the actual 
       memory address every iteration." */
}
/* CORRECT: Loop exits when hardware sets the status register */
```

```
  When to use volatile — The 3 Rules:
  
  ┌──────────────────────────────────────────────────────────┐
  │  Use volatile when:                                       │
  │                                                          │
  │  1. Accessing HARDWARE REGISTERS                         │
  │     → Hardware can change values at any time              │
  │                                                          │
  │  2. Variables shared between MAIN CODE and ISRs           │
  │     → Interrupt can modify the variable asynchronously    │
  │                                                          │
  │  3. Variables shared between MULTIPLE THREADS             │
  │     → Another thread/task can modify the value            │
  │                                                          │
  │  Do NOT use volatile for:                                │
  │  × Normal local variables                                │
  │  × Constants                                             │
  │  × Variables only accessed from one context              │
  └──────────────────────────────────────────────────────────┘
```

### 2.3.4 Bitwise Operations — The Complete Toolkit

Every hardware register is manipulated bit-by-bit. Master these six operations:

```c
/* ================================================================
 * THE SIX BITWISE OPERATIONS
 * ================================================================ */

/* 1. AND (&) — Masking / checking bits */
uint32_t result = 0xFF & 0x0F;    /* result = 0x0F */
/*   1111 1111
   & 0000 1111
   = 0000 1111   — keeps only the lower 4 bits */

/* 2. OR (|) — Setting bits */
uint32_t result = 0xF0 | 0x0F;    /* result = 0xFF */
/*   1111 0000
   | 0000 1111
   = 1111 1111   — both sets of bits are now 1 */

/* 3. XOR (^) — Toggling bits */
uint32_t result = 0xFF ^ 0x0F;    /* result = 0xF0 */
/*   1111 1111
   ^ 0000 1111
   = 1111 0000   — lower 4 bits flipped */

/* 4. NOT (~) — Inverting all bits */
uint32_t result = ~0x0F;          /* result = 0xFFFFFFF0 */
/*   0000 ... 0000 1111
   → 1111 ... 1111 0000   — every bit flipped */

/* 5. Left Shift (<<) — Moving bits left */
uint32_t result = 1 << 5;         /* result = 0x20 = 32 */
/*   0000 0001  →  0010 0000   — bit moves left 5 positions */

/* 6. Right Shift (>>) — Moving bits right */
uint32_t result = 0x80 >> 3;      /* result = 0x10 = 16 */
/*   1000 0000  →  0001 0000   — bit moves right 3 positions */
```

**Common firmware patterns using bitwise operations:**

```c
/* ================================================================
 * PATTERN 1: Set a single bit (turn something ON)
 * ================================================================ */
register |= (1UL << bit_number);

/* Example: Enable GPIOA clock (bit 0 of RCC_AHB1ENR) */
RCC_AHB1ENR |= (1UL << 0);

/*   Before: xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxx0
     Mask:   0000 0000 0000 0000 0000 0000 0000 0001
     OR:     xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxx1
     Only bit 0 changed! */


/* ================================================================
 * PATTERN 2: Clear a single bit (turn something OFF)
 * ================================================================ */
register &= ~(1UL << bit_number);

/* Example: Disable GPIOA clock */
RCC_AHB1ENR &= ~(1UL << 0);

/*   Before: xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxx1
     ~Mask:  1111 1111 1111 1111 1111 1111 1111 1110
     AND:    xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxx0
     Only bit 0 cleared! */


/* ================================================================
 * PATTERN 3: Toggle a bit (flip the state)
 * ================================================================ */
register ^= (1UL << bit_number);

/* Example: Toggle LED (PA5) */
GPIOA_ODR ^= (1UL << 5);


/* ================================================================
 * PATTERN 4: Check if a bit is set (read status)
 * ================================================================ */
if (register & (1UL << bit_number)) {
    /* Bit is set (1) */
}

/* Example: Check if button is pressed (PA0 low = pressed) */
if (!(GPIOA_IDR & (1UL << 0))) {
    /* Button is pressed (active low) */
}


/* ================================================================
 * PATTERN 5: Set a multi-bit field (clear first, then set)
 * ================================================================ */
register &= ~(mask << position);   /* Clear the field */
register |=  (value << position);  /* Set new value */

/* Example: Set PA5 to output mode (2-bit field in MODER) */
GPIOA_MODER &= ~(0x3UL << 10);   /* Clear bits [11:10] */
GPIOA_MODER |=  (0x1UL << 10);   /* Set to 01 (output) */


/* ================================================================
 * PATTERN 6: Extract a multi-bit field (mask and shift)
 * ================================================================ */
value = (register >> position) & mask;

/* Example: Read PA5 mode from MODER */
uint32_t pa5_mode = (GPIOA_MODER >> 10) & 0x3;
/* Returns: 0=input, 1=output, 2=AF, 3=analog */
```

### 2.3.5 Structs for Peripheral Access

Professional firmware code maps entire peripheral register blocks to structs:

```c
/*
 * GPIO Register Structure
 * Maps to the actual hardware register layout in memory.
 * Each member corresponds to a 32-bit register at a specific offset.
 */
typedef struct {
    volatile uint32_t MODER;     /* Offset 0x00: Mode register */
    volatile uint32_t OTYPER;    /* Offset 0x04: Output type register */
    volatile uint32_t OSPEEDR;   /* Offset 0x08: Output speed register */
    volatile uint32_t PUPDR;     /* Offset 0x0C: Pull-up/pull-down register */
    volatile uint32_t IDR;       /* Offset 0x10: Input data register */
    volatile uint32_t ODR;       /* Offset 0x14: Output data register */
    volatile uint32_t BSRR;      /* Offset 0x18: Bit set/reset register */
    volatile uint32_t LCKR;      /* Offset 0x1C: Lock register */
    volatile uint32_t AFR[2];    /* Offset 0x20-0x24: AF registers */
} GPIO_TypeDef;

/*
 * Now we can define a pointer to the struct at the peripheral base address.
 * This maps the struct directly onto the hardware registers!
 */
#define GPIOA   ((GPIO_TypeDef *)0x40020000UL)
#define GPIOB   ((GPIO_TypeDef *)0x40020400UL)
#define GPIOC   ((GPIO_TypeDef *)0x40020800UL)

/* Usage — much cleaner! */
GPIOA->MODER |= (1UL << 10);    /* PA5 output mode */
GPIOA->ODR   ^= (1UL << 5);     /* Toggle PA5 */

/* The -> operator accesses struct members through a pointer */
```

```
  How struct mapping works in memory:
  
  GPIOA = (GPIO_TypeDef *)0x40020000
  
  &GPIOA->MODER   = 0x40020000 + 0x00 = 0x40020000  ✓
  &GPIOA->OTYPER  = 0x40020000 + 0x04 = 0x40020004  ✓
  &GPIOA->OSPEEDR = 0x40020000 + 0x08 = 0x40020008  ✓
  &GPIOA->PUPDR   = 0x40020000 + 0x0C = 0x4002000C  ✓
  &GPIOA->IDR     = 0x40020000 + 0x10 = 0x40020010  ✓
  &GPIOA->ODR     = 0x40020000 + 0x14 = 0x40020014  ✓
  &GPIOA->BSRR    = 0x40020000 + 0x18 = 0x40020018  ✓
  
  The compiler calculates these offsets automatically based on
  member sizes (each uint32_t is 4 bytes = 0x04 offset).
```

### 2.3.6 Unions and Bit Fields

Unions allow viewing the same memory in different ways:

```c
/*
 * A union for a status register — view as:
 *   - A whole 32-bit word
 *   - Individual bit fields
 */
typedef union {
    uint32_t raw;            /* Access as a single 32-bit value */
    struct {
        uint32_t  busy:1;    /* Bit 0: Device busy flag */
        uint32_t  error:1;   /* Bit 1: Error flag */
        uint32_t  done:1;    /* Bit 2: Transfer complete */
        uint32_t  rsvd:29;   /* Bits 3-31: Reserved */
    } bits;
} StatusReg_t;

/* Usage: */
volatile StatusReg_t *status = (volatile StatusReg_t *)0x40004000;

/* Read individual flags: */
if (status->bits.busy) {
    /* Device is busy */
}

/* Read raw register value: */
uint32_t value = status->raw;
```

**Warning about bit fields:** The C standard does not guarantee bit-field layout across compilers. For portable firmware, prefer explicit bitwise operations. Bit fields are useful for debugging and documentation but should be used cautiously in production code.

### 2.3.7 `const`, `static`, and Scope

```c
/* ================================================================
 * const — This value CANNOT be changed after initialization
 * ================================================================ */

/* In embedded: const data goes to FLASH, not RAM */
const uint32_t lookup_table[4] = {100, 200, 300, 400};
/* Saves precious SRAM! The table lives in Flash (read-only memory) */

/* const pointer to volatile register — the address is fixed,
   but the register value can change */
volatile uint32_t * const GPIOA_ODR_PTR = 
    (volatile uint32_t *)0x40020014UL;
/* Can't change the pointer, but can modify what it points to */

/* ================================================================
 * static — Two different meanings depending on context
 * ================================================================ */

/* 1. Static LOCAL variable — persists between function calls */
void count_events(void) {
    static uint32_t event_count = 0;  /* Initialized ONCE, persists */
    event_count++;                     /* Increments each call */
    /* Without 'static', count would reset to 0 each call */
}

/* 2. Static GLOBAL variable/function — file-scope only (private) */
static uint32_t module_state = 0;  /* Only visible in this file */
static void helper_function(void); /* Only callable from this file */
/* This prevents name collisions between files — crucial in large
   firmware projects with no namespace support in C */

/* ================================================================
 * const volatile — Yes, this combination exists!
 * ================================================================ */
/* A register the CPU can read but not write, 
   but hardware can change at any time */
const volatile uint32_t *IDR = 
    (const volatile uint32_t *)0x40020010UL;
/* This is the Input Data Register — CPU reads it, but hardware
   (the pin state) can change it at any time */
```

### 2.3.8 Function Pointers — For Interrupt Vectors and Callbacks

```c
/* A function pointer stores the address of a function */
typedef void (*callback_t)(void);  /* Pointer to a void func(void) */

/* Define a callback variable */
callback_t on_button_press = NULL;

/* Register a callback */
void register_callback(callback_t cb) {
    on_button_press = cb;
}

/* Use the callback */
void check_button(void) {
    if (button_pressed() && on_button_press != NULL) {
        on_button_press();  /* Call the registered function */
    }
}

/* Example: Register a handler */
void my_handler(void) {
    toggle_led();
}

int main(void) {
    register_callback(my_handler);
    while (1) {
        check_button();
    }
}
```

The **interrupt vector table** is essentially an array of function pointers — it tells the CPU which function to call when each interrupt occurs. We'll dive deep into this in Chapter 6.

### 2.3.9 MISRA C Mindset

MISRA C is a set of coding guidelines developed for safety-critical embedded systems. While following every MISRA rule is overkill for learning, adopting the **mindset** early makes you a better firmware engineer.

```c
/* ================================================================
 * Key MISRA-inspired rules for your code:
 * ================================================================ */

/* Rule: Always use braces, even for single-line if/else */
/* BAD: */
if (x) do_something();

/* GOOD: */
if (x) {
    do_something();
}

/* Rule: Use unsigned suffix (U, UL) on constants */
/* BAD: */
uint32_t mask = 1 << 5;            /* 1 is signed int! */

/* GOOD: */
uint32_t mask = 1UL << 5;          /* 1UL is unsigned long */

/* Rule: No implicit type conversions */
/* BAD: */
uint32_t big = 0xFFFFFFFF;
uint8_t  small = big;               /* Silent truncation! */

/* GOOD: */
uint8_t  small = (uint8_t)(big & 0xFFU);  /* Explicit */

/* Rule: Every switch must have a default case */
switch (state) {
    case STATE_IDLE:    handle_idle();    break;
    case STATE_ACTIVE:  handle_active();  break;
    default:            handle_error();   break;  /* ALWAYS */
}

/* Rule: No dynamic memory allocation (malloc/free) */
/* malloc() can fragment memory and fail unpredictably.
   In a medical device or car, that's not acceptable.
   Use static allocation: */
static uint8_t uart_buffer[256];  /* Known size, always available */

/* Rule: Functions should have a single point of exit */
/* BAD: */
int read_sensor(void) {
    if (error) return -1;
    if (busy) return -2;
    return sensor_value;  /* Three return points! */
}

/* GOOD: */
int read_sensor(void) {
    int result;
    if (error) {
        result = -1;
    } else if (busy) {
        result = -2;
    } else {
        result = sensor_value;
    }
    return result;  /* Single point of exit */
}
```

---

## 2.4 Practical STM32 Nucleo Example

### LED Blink Using Struct-Based Register Access

Let's rewrite the Chapter 1 LED blink using professional struct-based access:

```c
/******************************************************************************
 * File:    main.c
 * Brief:   LED blink using struct-mapped peripheral access
 * Board:   STM32 Nucleo-F401RE
 * 
 * Demonstrates:
 *   - GPIO_TypeDef struct for register access
 *   - RCC_TypeDef for clock control
 *   - MISRA-style coding practices
 *   - Fixed-width types throughout
 *****************************************************************************/

#include <stdint.h>

/* ========================================================================= */
/*                     PERIPHERAL TYPE DEFINITIONS                           */
/* ========================================================================= */

typedef struct {
    volatile uint32_t MODER;     /* 0x00: Mode register */
    volatile uint32_t OTYPER;    /* 0x04: Output type */
    volatile uint32_t OSPEEDR;   /* 0x08: Output speed */
    volatile uint32_t PUPDR;     /* 0x0C: Pull-up/Pull-down */
    volatile uint32_t IDR;       /* 0x10: Input data */
    volatile uint32_t ODR;       /* 0x14: Output data */
    volatile uint32_t BSRR;      /* 0x18: Bit set/reset */
    volatile uint32_t LCKR;      /* 0x1C: Lock */
    volatile uint32_t AFR[2];    /* 0x20-0x24: Alternate function */
} GPIO_TypeDef;

typedef struct {
    volatile uint32_t CR;        /* 0x00: Clock control */
    volatile uint32_t PLLCFGR;   /* 0x04: PLL configuration */
    volatile uint32_t CFGR;      /* 0x08: Clock configuration */
    volatile uint32_t CIR;       /* 0x0C: Clock interrupt */
    volatile uint32_t AHB1RSTR;  /* 0x10: AHB1 reset */
    volatile uint32_t AHB2RSTR;  /* 0x14: AHB2 reset */
    volatile uint32_t RESERVED0[2]; /* 0x18-0x1C */
    volatile uint32_t APB1RSTR;  /* 0x20: APB1 reset */
    volatile uint32_t APB2RSTR;  /* 0x24: APB2 reset */
    volatile uint32_t RESERVED1[2]; /* 0x28-0x2C */
    volatile uint32_t AHB1ENR;   /* 0x30: AHB1 clock enable */
    volatile uint32_t AHB2ENR;   /* 0x34: AHB2 clock enable */
    volatile uint32_t RESERVED2[2]; /* 0x38-0x3C */
    volatile uint32_t APB1ENR;   /* 0x40: APB1 clock enable */
    volatile uint32_t APB2ENR;   /* 0x44: APB2 clock enable */
} RCC_TypeDef;

/* ========================================================================= */
/*                     PERIPHERAL BASE ADDRESSES                             */
/* ========================================================================= */

#define GPIOA       ((GPIO_TypeDef *)0x40020000UL)
#define GPIOB       ((GPIO_TypeDef *)0x40020400UL)
#define GPIOC       ((GPIO_TypeDef *)0x40020800UL)
#define RCC         ((RCC_TypeDef  *)0x40023800UL)

/* ========================================================================= */
/*                          BIT DEFINITIONS                                  */
/* ========================================================================= */

#define RCC_AHB1ENR_GPIOAEN     (1UL << 0)
#define GPIO_MODER_OUTPUT       (1UL)         /* Mode value for output */
#define LED_PIN                 (5U)

/* ========================================================================= */
/*                           FUNCTIONS                                       */
/* ========================================================================= */

static void delay(volatile uint32_t count)
{
    while (count > 0U)
    {
        count--;
    }
}

int main(void)
{
    /* Step 1: Enable GPIOA clock */
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;

    /* Step 2: Configure PA5 as output */
    GPIOA->MODER &= ~(3UL << (LED_PIN * 2U));    /* Clear mode bits */
    GPIOA->MODER |=  (GPIO_MODER_OUTPUT << (LED_PIN * 2U)); /* Set output */

    /* Step 3: Main loop — blink forever */
    while (1)
    {
        GPIOA->ODR ^= (1UL << LED_PIN);  /* Toggle LED */
        delay(500000UL);                   /* Wait */
    }

    return 0;   /* Never reached */
}
```

---

## 2.5 Diagrams

### Data Types and Their Size in Memory

```
  Memory representation of fixed-width types:
  
  uint8_t  (1 byte):   [████████]
                         8 bits
  
  uint16_t (2 bytes):  [████████ ████████]
                        16 bits
  
  uint32_t (4 bytes):  [████████ ████████ ████████ ████████]
                        32 bits
                        ← This matches STM32 register width →
  
  ARM Cortex-M4 data bus is 32 bits wide, so uint32_t
  is the natural register access size. Single-cycle access!
  
  For uint8_t/uint16_t register access, the CPU still reads
  32 bits and masks the result.
```

### Bitwise Operation Visual Guide

```
  Setting bit 5:
  
  Before:   0 0 1 0 0 0 1 0   (0x22)
  Mask:     0 0 1 0 0 0 0 0   (1 << 5 = 0x20)
  OR:       0 0 1 0 0 0 1 0   
          | 0 0 1 0 0 0 0 0
          = 0 0 1 0 0 0 1 0   (0x22 — bit 5 was already set)
  
  
  Clearing bit 1:
  
  Before:   0 0 1 0 0 0 1 0   (0x22)
  Mask:     0 0 0 0 0 0 1 0   (1 << 1 = 0x02)
  ~Mask:    1 1 1 1 1 1 0 1   (~0x02 = 0xFD)
  AND:      0 0 1 0 0 0 1 0
          & 1 1 1 1 1 1 0 1
          = 0 0 1 0 0 0 0 0   (0x20 — bit 1 cleared!)
  
  
  Toggling bit 5:
  
  Before:   0 0 1 0 0 0 0 0   (0x20)
  Mask:     0 0 1 0 0 0 0 0   (1 << 5 = 0x20)
  XOR:      0 0 1 0 0 0 0 0
          ^ 0 0 1 0 0 0 0 0
          = 0 0 0 0 0 0 0 0   (0x00 — bit 5 toggled OFF!)
```

---

## 2.6 Common Mistakes & Debugging

### Mistake #1: Signed vs. Unsigned Shift

```c
/* DANGEROUS: Shifting a signed value */
int32_t val = 1;
uint32_t result = val << 31;  /* UNDEFINED BEHAVIOR in C! */
/* Shifting a 1 into the sign bit of a signed integer is UB */

/* SAFE: Always use unsigned literals */
uint32_t result = 1UL << 31;  /* Well-defined: 0x80000000 */
```

### Mistake #2: Operator Precedence Trap

```c
/* WRONG — & has lower precedence than == */
if (GPIOA->IDR & (1UL << 5) == 1) {
    /* This evaluates as: GPIOA->IDR & ((1UL << 5) == 1)
       which is: GPIOA->IDR & 0 = always 0! */
}

/* CORRECT — Always use explicit parentheses */
if ((GPIOA->IDR & (1UL << 5)) != 0U) {
    /* PA5 is high */
}
```

### Mistake #3: Read-Modify-Write Race Condition

```c
/* If an interrupt fires between the read and write: */
GPIOA->ODR |= (1UL << 5);
/* This is actually: temp = GPIOA->ODR; temp |= (1<<5); GPIOA->ODR = temp; */
/* If an ISR modifies ODR between read and write → ISR change is lost */

/* SAFER: Use BSRR for atomic bit operations */
GPIOA->BSRR = (1UL << 5);     /* Set PA5 — atomic, no read-modify-write */
GPIOA->BSRR = (1UL << 21);    /* Reset PA5 — atomic */
```

### Debugging Tips

1. **Check variable types in the debugger:** Hover over variables to verify they're the expected width
2. **Use the Expression view:** Type `RCC->AHB1ENR` to see the current clock enable state
3. **Memory browser:** Navigate to `0x40020000` to see raw GPIOA register values
4. **Breakpoint + step:** Set a breakpoint on the MODER write and step through to verify the value changes

---

## 2.7 Exercises

### Conceptual Questions

1. What is the difference between `uint32_t *p` and `volatile uint32_t *p`? When do you need each?

2. Explain what Read-Modify-Write (RMW) means and why it matters when an ISR shares a register with the main loop.

3. Why is dynamic memory allocation (`malloc`) generally avoided in embedded firmware? Give three reasons.

4. What happens if you forget the `U` suffix: `uint32_t x = 1 << 31;`? What about `uint32_t x = 1UL << 31;`?

5. A struct has the following layout: `{ uint8_t a; uint32_t b; uint8_t c; }`. How much memory might this use? Why might it not be 6 bytes?

### Coding Exercises

**Exercise 2.1:** Write macros `BIT_SET(reg, bit)`, `BIT_CLEAR(reg, bit)`, `BIT_TOGGLE(reg, bit)`, and `BIT_READ(reg, bit)` that work with any register and bit number.

**Exercise 2.2:** Write a function `uint8_t extract_field(uint32_t reg, uint8_t start_bit, uint8_t width)` that extracts any multi-bit field from a register.

**Exercise 2.3:** Create a complete `GPIO_TypeDef` struct and use it to configure PB3 as output with push-pull type, medium speed, and no pull-up/pull-down.

**Exercise 2.4:** Write a function that counts the number of set bits in a 32-bit register value (population count). This is useful for checking how many interrupts are pending.

---

## 2.8 Industry & Career Notes

### How Professionals Use These Concepts

- **CMSIS headers** (from ARM/ST) provide pre-built `GPIO_TypeDef` structs, base address defines, and bit masks. You should understand how they work before using them.
- **Code review** always checks for missing `volatile`, signed/unsigned issues, and hardcoded magic numbers.
- **Static analysis tools** (PC-lint, Polyspace, Coverity) automatically check for MISRA violations — knowing the rules helps you write code that passes these tools.

### Coding Standards in Real Projects

| Standard | Industry | Key Focus |
|----------|----------|-----------|
| MISRA C:2012 | Automotive | Safety, no undefined behavior |
| CERT C | Cybersecurity | Secure coding practices |
| IEC 62304 | Medical | Software lifecycle for medical devices |
| DO-178C | Aerospace | Software assurance levels |
| BARR-C | General embedded | Readable, maintainable firmware |

---

## 2.9 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 2 SUMMARY                         │
  │                                                            │
  │  ✓ Use <stdint.h> types (uint32_t, etc.) — ALWAYS         │
  │                                                            │
  │  ✓ volatile is MANDATORY for hardware registers,           │
  │    ISR-shared variables, and multi-thread variables        │
  │                                                            │
  │  ✓ Six bitwise operations: & | ^ ~ << >>                  │
  │    Master the patterns: SET, CLEAR, TOGGLE, CHECK          │
  │                                                            │
  │  ✓ Structs map directly to peripheral register blocks      │
  │    using pointer casts to base addresses                   │
  │                                                            │
  │  ✓ const puts data in Flash, static controls visibility    │
  │                                                            │
  │  ✓ Function pointers power interrupt vectors & callbacks   │
  │                                                            │
  │  ✓ MISRA mindset: no malloc, explicit types, no magic      │
  │    numbers, braces everywhere, single return point         │
  │                                                            │
  │  ✓ In embedded: every byte matters, every bit matters      │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 3 — Development Environment & Toolchain →](./Chapter_03_Development_Environment.md)*
