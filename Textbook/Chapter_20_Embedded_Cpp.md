# Chapter 20: Embedded C++ for Firmware

**Difficulty Level: ★★★★☆ Advanced**  
**Estimated Time: 2–3 weeks**

---

## 20.1 Concept Overview

### Why C++ in Firmware?

C++ offers powerful abstractions that can make firmware safer and more maintainable **without runtime cost** — if used correctly. Modern C++ (C++11/14/17) provides features that are genuinely useful for embedded:

```
  ✓ Use These C++ Features:          ✗ Avoid on Embedded:
  ─────────────────────────           ──────────────────────
  Classes & encapsulation             Exceptions (throw/try/catch)
  constexpr (compile-time math)       RTTI (dynamic_cast, typeid)
  Templates (zero-cost generics)      STL containers (std::vector, etc.)
  Namespaces                          iostream (std::cout)
  enum class (scoped enums)           Virtual functions (carefully)
  RAII (resource management)          Dynamic memory (new/delete)
  static_assert (compile checks)      Multiple inheritance
  References                          std::string
```

### C++ vs C — Zero-Cost Abstractions

```
  C Version:
    gpio_init(GPIOA, 5, GPIO_MODE_OUTPUT);
    gpio_write(GPIOA, 5, 1);
  
  C++ Version:
    Gpio led(Port::A, 5, Mode::Output);
    led.set();
  
  Generated assembly: IDENTICAL
  → C++ adds safety + readability with NO runtime cost
```

---

## 20.2 Compiler Setup

### Compiler Flags for Embedded C++

```makefile
# In your Makefile:
CXX = arm-none-eabi-g++
CXXFLAGS  = -mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16
CXXFLAGS += -std=c++17             # Modern C++ standard
CXXFLAGS += -fno-exceptions        # Disable exceptions (saves ~20 KB)
CXXFLAGS += -fno-rtti              # Disable RTTI (saves ~5 KB)
CXXFLAGS += -fno-threadsafe-statics # No guard variables for static init
CXXFLAGS += -fno-use-cxa-atexit    # No destructor registration at exit
CXXFLAGS += -Os                    # Optimize for size
CXXFLAGS += -Wall -Wextra -Werror
```

### Linking C and C++ Together

```cpp
// In C++ code, to call C functions:
extern "C" {
    #include "uart_driver.h"
    #include "gpio_driver.h"
}

// In C++ header, to be callable from C:
#ifdef __cplusplus
extern "C" {
#endif

void SysTick_Handler(void);
int main(void);

#ifdef __cplusplus
}
#endif
```

---

## 20.3 Classes for Hardware Abstraction

### GPIO Class

```cpp
/******************************************************************************
 * File:    gpio.hpp
 * Brief:   Type-safe GPIO driver using C++ classes
 *****************************************************************************/
#pragma once

#include <cstdint>

namespace hw {

/* Scoped enums — no name collisions, type-safe */
enum class Port : uint8_t { A = 0, B, C, D, E, H };
enum class Mode : uint8_t { Input = 0, Output = 1, AltFunc = 2, Analog = 3 };
enum class Pull : uint8_t { None = 0, Up = 1, Down = 2 };
enum class Speed : uint8_t { Low = 0, Medium = 1, Fast = 2, High = 3 };

class Gpio {
public:
    /* Constructor — configures the pin */
    Gpio(Port port, uint8_t pin, Mode mode,
         Pull pull = Pull::None, Speed speed = Speed::Low)
        : port_(port), pin_(pin)
    {
        /* Enable clock for this port */
        volatile uint32_t *RCC_AHB1ENR = 
            reinterpret_cast<volatile uint32_t *>(0x40023830UL);
        *RCC_AHB1ENR |= (1UL << static_cast<uint8_t>(port));

        /* Get GPIO base address */
        volatile uint32_t *base = port_base();

        /* MODER */
        base[0] &= ~(3UL << (pin * 2));
        base[0] |= (static_cast<uint32_t>(mode) << (pin * 2));

        /* OSPEEDR */
        base[2] &= ~(3UL << (pin * 2));
        base[2] |= (static_cast<uint32_t>(speed) << (pin * 2));

        /* PUPDR */
        base[3] &= ~(3UL << (pin * 2));
        base[3] |= (static_cast<uint32_t>(pull) << (pin * 2));
    }

    void set()    const { port_base()[6] = (1UL << pin_); }       /* BSRR set */
    void clear()  const { port_base()[6] = (1UL << (pin_ + 16)); }/* BSRR reset */
    void toggle() const { port_base()[5] ^= (1UL << pin_); }      /* ODR toggle */
    bool read()   const { return (port_base()[4] >> pin_) & 1UL; } /* IDR */

    void write(bool val) const { val ? set() : clear(); }

    /* Alternate function setup */
    void set_alt_func(uint8_t af) const
    {
        volatile uint32_t *afr = &port_base()[8 + (pin_ / 8)]; /* AFRL/AFRH */
        uint8_t shift = (pin_ % 8) * 4;
        *afr &= ~(0xFUL << shift);
        *afr |= (static_cast<uint32_t>(af) << shift);
    }

private:
    Port port_;
    uint8_t pin_;

    volatile uint32_t *port_base() const
    {
        constexpr uint32_t GPIO_BASE = 0x40020000UL;
        constexpr uint32_t GPIO_STRIDE = 0x400UL;
        return reinterpret_cast<volatile uint32_t *>(
            GPIO_BASE + static_cast<uint8_t>(port_) * GPIO_STRIDE
        );
    }
};

} // namespace hw
```

### Usage Example

```cpp
#include "gpio.hpp"

/* LED on PA5, Button on PC13 */
hw::Gpio led(hw::Port::A, 5, hw::Mode::Output);
hw::Gpio button(hw::Port::C, 13, hw::Mode::Input, hw::Pull::None);

int main()
{
    while (true)
    {
        if (!button.read())  /* Active low */
        {
            led.toggle();
            for (volatile uint32_t i = 0; i < 100000; i++);
        }
    }
}
```

---

## 20.4 constexpr — Compile-Time Computation

### Problem

Many firmware calculations involve constants (baud rates, prescalers, frequencies). In C, you'd use `#define` macros. In C++, use `constexpr` — it's type-safe and checked at compile time.

```cpp
/* Calculate UART BRR at compile time */
constexpr uint32_t uart_brr(uint32_t pclk, uint32_t baud)
{
    /* BRR = PCLK / BAUD, rounded */
    return (pclk + baud / 2) / baud;
}

/* These are computed by the COMPILER, not at runtime */
constexpr uint32_t BRR_115200 = uart_brr(16'000'000, 115200);  /* = 139 */
constexpr uint32_t BRR_9600   = uart_brr(16'000'000, 9600);    /* = 1667 */

/* Compile-time assertion! */
static_assert(BRR_115200 > 0, "Invalid baud rate");
static_assert(BRR_115200 < 0xFFFF, "BRR overflow");

/* Timer prescaler */
constexpr uint32_t timer_prescaler(uint32_t clk, uint32_t desired_hz)
{
    return (clk / desired_hz) - 1;
}

constexpr uint32_t PSC_1KHZ = timer_prescaler(84'000'000, 1000);
static_assert(PSC_1KHZ == 83999, "Prescaler mismatch");
```

---

## 20.5 Templates — Zero-Cost Generics

### Type-Safe Register Access

```cpp
/******************************************************************************
 * Register access template — type-safe, zero-overhead
 *****************************************************************************/

template <uint32_t Address>
class Register {
public:
    static void write(uint32_t value)
    {
        *reinterpret_cast<volatile uint32_t *>(Address) = value;
    }

    static uint32_t read()
    {
        return *reinterpret_cast<volatile uint32_t *>(Address);
    }

    static void set_bits(uint32_t mask)
    {
        *reinterpret_cast<volatile uint32_t *>(Address) |= mask;
    }

    static void clear_bits(uint32_t mask)
    {
        *reinterpret_cast<volatile uint32_t *>(Address) &= ~mask;
    }

    static void modify(uint32_t clear_mask, uint32_t set_mask)
    {
        uint32_t val = read();
        val &= ~clear_mask;
        val |= set_mask;
        write(val);
    }
};

/* Define registers */
using RCC_AHB1ENR = Register<0x40023830>;
using GPIOA_MODER = Register<0x40020000>;
using GPIOA_ODR   = Register<0x40020014>;
using GPIOA_BSRR  = Register<0x40020018>;

/* Usage — compiles to EXACTLY the same code as raw pointer access */
void blink()
{
    RCC_AHB1ENR::set_bits(1UL << 0);          /* Enable GPIOA clock */
    GPIOA_MODER::modify(3UL << 10, 1UL << 10); /* PA5 output */
    GPIOA_BSRR::write(1UL << 5);              /* Set PA5 */
}
```

### Circular Buffer Template

```cpp
/******************************************************************************
 * Type-safe, size-configurable circular buffer
 *****************************************************************************/

template <typename T, uint32_t Size>
class CircularBuffer {
    static_assert((Size & (Size - 1)) == 0, "Size must be power of 2");

public:
    bool push(T item)
    {
        uint32_t next = (head_ + 1) & (Size - 1);
        if (next == tail_) return false;  /* Full */
        buffer_[head_] = item;
        head_ = next;
        return true;
    }

    bool pop(T &item)
    {
        if (head_ == tail_) return false;  /* Empty */
        item = buffer_[tail_];
        tail_ = (tail_ + 1) & (Size - 1);
        return true;
    }

    bool is_empty() const { return head_ == tail_; }
    bool is_full()  const { return ((head_ + 1) & (Size - 1)) == tail_; }
    uint32_t count() const { return (head_ - tail_) & (Size - 1); }

private:
    T buffer_[Size]{};
    volatile uint32_t head_ = 0;
    volatile uint32_t tail_ = 0;
};

/* Usage */
CircularBuffer<uint8_t, 256> uart_rx_buf;   /* 256-byte UART receive buffer */
CircularBuffer<uint16_t, 64> adc_samples;   /* 64-sample ADC buffer */
```

---

## 20.6 RAII — Resource Acquisition Is Initialization

### Interrupt Lock Guard

```cpp
/******************************************************************************
 * RAII pattern: automatically re-enable interrupts when scope ends
 *****************************************************************************/

class InterruptLock {
public:
    InterruptLock()
    {
        __asm volatile ("MRS %0, PRIMASK" : "=r" (primask_));
        __asm volatile ("CPSID I");  /* Disable interrupts */
    }

    ~InterruptLock()
    {
        __asm volatile ("MSR PRIMASK, %0" : : "r" (primask_));
    }

    /* Non-copyable */
    InterruptLock(const InterruptLock &) = delete;
    InterruptLock &operator=(const InterruptLock &) = delete;

private:
    uint32_t primask_;
};

/* Usage — interrupts ALWAYS re-enabled, even if function returns early */
void fifo_push_safe(uint8_t data)
{
    InterruptLock lock;  /* Interrupts disabled HERE */

    if (fifo_full()) return;  /* ← Interrupts restored automatically! */

    fifo_buffer[fifo_head] = data;
    fifo_head = (fifo_head + 1) & (FIFO_SIZE - 1);

}  /* ← Interrupts restored automatically HERE too */
```

### Pin Scope Guard

```cpp
/* RAII: Set a debug pin high while in scope (for oscilloscope timing) */
class ScopePin {
public:
    explicit ScopePin(hw::Gpio &pin) : pin_(pin) { pin_.set(); }
    ~ScopePin() { pin_.clear(); }

    ScopePin(const ScopePin &) = delete;
    ScopePin &operator=(const ScopePin &) = delete;

private:
    hw::Gpio &pin_;
};

hw::Gpio debug_pin(hw::Port::B, 0, hw::Mode::Output);

void time_critical_function()
{
    ScopePin scope(debug_pin);  /* Pin goes HIGH */

    /* ... do work here ... */
    /* Oscilloscope shows exactly how long this function takes */

}  /* Pin goes LOW automatically */
```

---

## 20.7 enum class and static_assert

```cpp
/* Type-safe enums (no implicit conversions) */
enum class SpiMode : uint8_t {
    Mode0 = 0,  /* CPOL=0, CPHA=0 */
    Mode1 = 1,  /* CPOL=0, CPHA=1 */
    Mode2 = 2,  /* CPOL=1, CPHA=0 */
    Mode3 = 3   /* CPOL=1, CPHA=1 */
};

/* In C, this compiles (and is a bug): */
/* gpio_init(SPI_MODE_0);  ← wrong enum, wrong function! */

/* In C++, this FAILS to compile: */
/* spi_init(Mode::Output);  ← compiler error! Type mismatch */

/* Compile-time checks */
struct SensorConfig {
    uint8_t address;
    uint8_t reg;
    uint8_t value;
};

static_assert(sizeof(SensorConfig) == 3, "Unexpected padding!");

/* Ensure buffer sizes are correct */
constexpr uint32_t DMA_BUFFER_SIZE = 512;
static_assert(DMA_BUFFER_SIZE % 4 == 0, "DMA buffer must be word-aligned");
```

---

## 20.8 Embedded C++ Patterns

### Singleton for Hardware Peripherals

```cpp
/* Each peripheral exists once in hardware — Singleton makes sense */
class Uart {
public:
    static Uart &instance()
    {
        static Uart uart;  /* Created once, lives forever */
        return uart;
    }

    void send(char c)
    {
        while (!(USART2_SR & (1UL << 7)));
        USART2_DR = static_cast<uint32_t>(c);
    }

    void print(const char *s)
    {
        while (*s) send(*s++);
    }

    /* Delete copy/move */
    Uart(const Uart &) = delete;
    Uart &operator=(const Uart &) = delete;

private:
    Uart()  /* Private constructor — init hardware */
    {
        /* Enable clocks, configure pins, set baud rate... */
        RCC_AHB1ENR::set_bits(1UL << 0);
        RCC_APB1ENR::set_bits(1UL << 17);
        /* ... (same init as C version) ... */
    }
};

/* Usage */
void log_message()
{
    Uart::instance().print("System initialized\r\n");
}
```

### Callback with Function Pointers (No std::function)

```cpp
/* Lightweight callback — no heap, no overhead */
template <typename Signature>
class Callback;

template <typename Ret, typename... Args>
class Callback<Ret(Args...)> {
public:
    using FuncPtr = Ret(*)(Args...);

    Callback() : func_(nullptr) {}
    Callback(FuncPtr f) : func_(f) {}

    Ret operator()(Args... args) const
    {
        if (func_) return func_(args...);
        return Ret{};
    }

    explicit operator bool() const { return func_ != nullptr; }

private:
    FuncPtr func_;
};

/* Usage with ISR-style callbacks */
Callback<void()> on_button_press;

void button_handler()
{
    /* ... debounce ... */
    if (on_button_press) on_button_press();
}

void toggle_led() { led.toggle(); }

void setup()
{
    on_button_press = toggle_led;  /* Assign callback */
}
```

---

## 20.9 What NOT to Use

```
  Feature              Why Avoid          Cost
  ──────────────────   ─────────────────  ──────────────────
  Exceptions           +20KB Flash,       -fno-exceptions
                       stack unwinding    
  RTTI                 +5KB Flash,        -fno-rtti
                       type_info tables   
  std::vector          Dynamic memory     Use fixed arrays
  std::string          Dynamic memory     Use char arrays
  std::cout            Massive binary     Use UART printf
  new/delete           Heap fragmentation Use static alloc
  Virtual functions    vtable + indirection Use templates or
                       (2-4 byte overhead  callbacks instead
                       per object + call)
```

---

## 20.10 Exercises

**Exercise 20.1:** Convert your GPIO driver from Chapter 4 to a C++ class. Verify that the compiled output size is identical to the C version using `arm-none-eabi-size`.

**Exercise 20.2:** Create a `Timer` class that uses `constexpr` to compute prescaler and period values at compile time. Use `static_assert` to verify the computed frequency is within 1% of the target.

**Exercise 20.3:** Implement the `CircularBuffer` template and use it in your UART ISR. Compare code size with the C implementation.

**Exercise 20.4:** Write an `I2cTransaction` RAII class that automatically sends a STOP condition when it goes out of scope, even if an error occurs mid-transfer.

---

## 20.11 Chapter Summary

```
  C++ for Firmware — Rules of Engagement:
  
  ✓ Classes          → Encapsulate peripherals, enforce invariants
  ✓ constexpr        → Replace #define macros, compile-time math
  ✓ Templates        → Zero-cost generics (buffers, registers)
  ✓ RAII             → Automatic resource cleanup (interrupts, locks)
  ✓ enum class       → Type-safe enumerations
  ✓ static_assert    → Catch errors at compile time
  ✓ Namespaces       → Avoid name collisions
  
  Compiler flags:    -fno-exceptions -fno-rtti
  Golden rule:       If it adds runtime cost, don't use it.
```

---

*Next: [Chapter 21 — ARM Assembly Essentials →](./Chapter_21_ARM_Assembly.md)*
