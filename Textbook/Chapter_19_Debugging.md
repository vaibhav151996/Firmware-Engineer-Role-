# Chapter 19: Debugging & Test Equipment

**Difficulty Level: ★★★☆☆ Intermediate**  
**Estimated Time: 1–2 weeks**

---

## 19.1 Concept Overview

Firmware bugs are harder to find than software bugs. You can't just `printf` and step through everything — you're dealing with timing-critical hardware, interrupts, and analog signals. This chapter covers the essential debugging techniques and test equipment every firmware engineer must master.

### The Debugging Toolkit

```
  Software Tools:               Hardware Tools:
  ┌──────────────────┐          ┌──────────────────┐
  │ GDB (debugger)   │          │ Multimeter       │
  │ SWD / JTAG probe │          │ Oscilloscope     │
  │ ITM / SWO trace  │          │ Logic Analyzer   │
  │ printf via UART  │          │ Current Probe    │
  │ Hard Fault handler│          │ Bus Pirate       │
  │ Watchdog timer   │          │ Saleae / PulseView│
  │ Assertions       │          │ Signal Generator │
  └──────────────────┘          └──────────────────┘
```

---

## 19.2 GDB Debugging via SWD

### What Is SWD?

```
  SWD = Serial Wire Debug
  Only 2 pins: SWDIO + SWCLK (vs JTAG's 4-5 pins)
  
  Nucleo Board:
  ┌───────────────────────────┐
  │  ST-LINK V2/V3 debugger  │ ← Built into the Nucleo board!
  │  (top section of board)   │
  │                           │
  │    SWDIO ——→ PA13         │
  │    SWCLK ——→ PA14         │
  │    SWO   ——→ PB3          │
  │    NRST  ——→ Reset        │
  └───────────────────────────┘
  
  SWD capabilities:
  ✓ Set breakpoints (hardware: up to 6, software: unlimited in Flash)
  ✓ Read/write memory and registers in real-time
  ✓ Step through code (step-in, step-over, step-out)
  ✓ Watch variables and expressions
  ✓ Flash programming
```

### GDB Commands Reference

```
  Essential GDB Commands:
  
  Connection:
    target remote :3333        Connect to OpenOCD
    monitor reset halt         Reset and halt MCU
    load                       Flash the binary
    
  Execution:
    continue / c               Resume execution
    step / s                   Step into (follows function calls)
    next / n                   Step over (skip function calls)
    finish                     Run until current function returns
    
  Breakpoints:
    break main                 Break at main()
    break uart.c:42            Break at line 42 of uart.c
    break flash_write if sz>0  Conditional breakpoint
    delete 1                   Delete breakpoint #1
    info breakpoints           List all breakpoints
    
  Inspection:
    print variable             Print a variable's value
    print /x *0x40020000       Print memory as hex  
    print sizeof(struct_name)  Print struct size
    x/16xw 0x08000000          Examine 16 words at Flash start
    info registers             Show all CPU registers
    display variable           Auto-display each stop
    
  Memory:
    x/4xb 0x20000000           Examine 4 bytes (RAM start)
    x/8xw 0x40020000           Examine 8 words (GPIO base)
    set *(uint32_t*)0x40020018 = 0x20   Write to GPIO BSRR
    
  Watchpoints:
    watch my_variable          Break when my_variable changes
    rwatch my_variable         Break when my_variable is read
    awatch my_variable         Break on read or write
    
  Stack:
    backtrace / bt             Show call stack
    frame 2                    Select stack frame #2
    info locals                Show local variables
```

### OpenOCD Setup

```bash
# Start OpenOCD (in a separate terminal)
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg

# Connect GDB
arm-none-eabi-gdb firmware.elf
(gdb) target remote :3333
(gdb) monitor reset halt
(gdb) load
(gdb) break main
(gdb) continue
```

---

## 19.3 Hard Fault Debugging

When STM32 crashes, it enters a Hard Fault. Without proper handling, the CPU just halts in a default infinite loop. Here's how to make it useful:

### Hard Fault Handler

```c
/******************************************************************************
 * Hard Fault Handler — Extracts crash information
 *****************************************************************************/

/* This struct mirrors what the CPU pushes onto the stack during exception */
typedef struct {
    uint32_t r0;
    uint32_t r1;
    uint32_t r2;
    uint32_t r3;
    uint32_t r12;
    uint32_t lr;     /* Link Register — return address */
    uint32_t pc;     /* Program Counter — WHERE it crashed */
    uint32_t psr;    /* Program Status Register */
} StackFrame_t;

/* Stored crash info for inspection via debugger */
volatile StackFrame_t crash_info;
volatile uint32_t crash_cfsr;   /* Configurable Fault Status Register */
volatile uint32_t crash_hfsr;   /* Hard Fault Status Register */
volatile uint32_t crash_mmfar;  /* MemManage Fault Address Register */
volatile uint32_t crash_bfar;   /* Bus Fault Address Register */

void HardFault_Handler_C(uint32_t *stack_ptr)
{
    /* Copy stacked registers */
    crash_info.r0  = stack_ptr[0];
    crash_info.r1  = stack_ptr[1];
    crash_info.r2  = stack_ptr[2];
    crash_info.r3  = stack_ptr[3];
    crash_info.r12 = stack_ptr[4];
    crash_info.lr  = stack_ptr[5];
    crash_info.pc  = stack_ptr[6];   /* ← THIS IS WHERE IT CRASHED */
    crash_info.psr = stack_ptr[7];

    /* Read fault status registers */
    crash_cfsr  = *(volatile uint32_t *)0xE000ED28UL;
    crash_hfsr  = *(volatile uint32_t *)0xE000ED2CUL;
    crash_mmfar = *(volatile uint32_t *)0xE000ED34UL;
    crash_bfar  = *(volatile uint32_t *)0xE000ED38UL;

    /* If UART is available, print crash info */
    /* uart_print("!!! HARD FAULT !!!\r\n"); */
    /* uart_print_hex("PC  = ", crash_info.pc); */
    /* uart_print_hex("LR  = ", crash_info.lr); */
    /* uart_print_hex("CFSR= ", crash_cfsr);    */

    /* Halt here — inspect variables in debugger
     *
     * In GDB:
     *   print /x crash_info.pc   → shows WHERE it crashed
     *   print /x crash_info.lr   → shows WHO called it
     *   print /x crash_cfsr      → shows WHY it crashed
     *
     * Use addr2line to find the source file and line:
     *   arm-none-eabi-addr2line -e firmware.elf -f 0x080012A4
     */
    while (1);
}

/* Assembly wrapper to extract stack pointer (goes in startup or .s file) */
__attribute__((naked))
void HardFault_Handler(void)
{
    __asm volatile (
        "TST   LR, #4          \n"   /* Check which stack was used */
        "ITE   EQ               \n"
        "MRSEQ R0, MSP          \n"   /* Main Stack Pointer */
        "MRSNE R0, PSP          \n"   /* Process Stack Pointer */
        "B     HardFault_Handler_C \n"
    );
}
```

### Decoding CFSR (Configurable Fault Status Register)

```
  CFSR at 0xE000ED28 (32-bit):
  
  Bits [7:0]   = MMFSR (MemManage Fault Status)
  Bits [15:8]  = BFSR  (Bus Fault Status)
  Bits [31:16] = UFSR  (Usage Fault Status)
  
  Common CFSR bits:
  ┌──────────┬───────────┬──────────────────────────────┐
  │ Bit      │ Name      │ Meaning                      │
  ├──────────┼───────────┼──────────────────────────────┤
  │ [0]      │ IACCVIOL  │ Instruction access violation  │
  │ [1]      │ DACCVIOL  │ Data access violation         │
  │ [4]      │ MSTKERR   │ Stack error on exception      │
  │ [7]      │ MMARVALID │ MMFAR has valid address       │
  │ [8]      │ IBUSERR   │ Instruction bus error         │
  │ [9]      │ PRECISERR │ Precise data bus error        │
  │ [10]     │ IMPRECISE │ Imprecise data bus error      │
  │ [15]     │ BFARVALID │ BFAR has valid address        │
  │ [16]     │ UNDEFINSTR│ Undefined instruction         │
  │ [17]     │ INVSTATE  │ Invalid state (Thumb bit)     │
  │ [18]     │ INVPC     │ Invalid PC                    │
  │ [19]     │ NOCP      │ No coprocessor                │
  │ [24]     │ UNALIGNED │ Unaligned access              │
  │ [25]     │ DIVBYZERO │ Divide by zero                │
  └──────────┴───────────┴──────────────────────────────┘
```

---

## 19.4 Printf Debugging (SWO Trace & UART)

### Method 1: UART Printf

The simplest approach — redirect `printf` to UART:

```c
#include <stdio.h>

/* Retarget _write for printf → UART */
int _write(int fd, char *ptr, int len)
{
    (void)fd;
    for (int i = 0; i < len; i++)
    {
        while (!(USART2_SR & (1UL << 7)));
        USART2_DR = (uint32_t)ptr[i];
    }
    return len;
}

/* Now you can use printf() */
void some_function(void)
{
    uint32_t adc_val = adc_read();
    printf("ADC value: %lu (%.2f V)\r\n", adc_val, adc_val * 3.3 / 4095.0);
}
```

### Method 2: SWO / ITM Trace (Non-Invasive)

```
  SWO (Serial Wire Output) — uses PB3 on STM32F401
  
  Advantages over UART printf:
  ✓ Doesn't use a UART peripheral
  ✓ Doesn't need GPIO pins
  ✓ Very fast (no blocking waits)
  ✓ Can be stripped in release builds
  ✓ Multiple channels (32 stimulus ports)
  
  ITM Port 0 = "printf" channel (convention)
```

```c
/* ITM / SWO Trace Output */
static void itm_send_char(char c)
{
    /* ITM Stimulus Port 0 */
    volatile uint32_t *ITM_STIM0 = (volatile uint32_t *)0xE0000000UL;
    volatile uint32_t *ITM_TER   = (volatile uint32_t *)0xE0000E00UL;

    if ((*ITM_TER & 1UL) == 0UL) return;  /* Port not enabled */

    while ((*ITM_STIM0 & 1UL) == 0UL);    /* Wait for ready */
    *ITM_STIM0 = (uint32_t)c;
}

void itm_print(const char *s)
{
    while (*s) { itm_send_char(*s++); }
}

/* Use in code */
void debug_example(void)
{
    itm_print("[DEBUG] SPI transfer complete\n");
}
```

---

## 19.5 Test Equipment

### Oscilloscope

```
  What It Does: Shows voltage over time on a screen
  
  When to use:
  ✓ Measuring PWM frequency and duty cycle
  ✓ Checking UART baud rate accuracy
  ✓ Debugging I2C/SPI timing
  ✓ Finding glitches on power rails
  ✓ Measuring interrupt latency
  
  Key Settings:
  ┌─────────────────────────────────────────────────┐
  │ Timebase: us/div → ms/div → s/div              │
  │ Voltage:  mV/div → V/div                       │
  │ Trigger:  Rising edge, level = 1.5V             │
  │ Coupling: DC (shows everything)                 │
  │           AC (blocks DC offset, shows ripple)   │
  └─────────────────────────────────────────────────┘
  
  Example: Verify 1 kHz PWM on PA5:
    Set timebase to 500 us/div (see 2 full cycles)
    Set voltage to 1 V/div
    Trigger on rising edge at 1.5V
    Measure: frequency, duty cycle, rise time
```

### Logic Analyzer

```
  What It Does: Captures DIGITAL signals with protocol decoding
  
  Advantages over oscilloscope for digital:
  ✓ 8-16+ channels simultaneously
  ✓ Protocol decoders: UART, SPI, I2C, CAN, etc.
  ✓ Very long capture times
  ✓ Much cheaper than oscilloscope
  ✗ No analog detail (only 0 or 1)
  
  Popular Tools:
  ┌────────────────────┬──────────┬──────────────────┐
  │ Tool               │ Price    │ Notes            │
  ├────────────────────┼──────────┼──────────────────┤
  │ Saleae Logic 8     │ $399     │ Best software    │
  │ Saleae Logic Pro   │ $999     │ + Analog         │
  │ DSLogic Plus       │ $99      │ PulseView compat │
  │ USB logic analyzer │ $10-20   │ PulseView/sigrok │
  └────────────────────┴──────────┴──────────────────┘
  
  Example: Debug SPI communication:
    Channel 0: CLK  (SCK)
    Channel 1: MOSI
    Channel 2: MISO
    Channel 3: CS
    → Logic analyzer decodes: Address = 0x75, Data = 0x68
    → Shows timing: CS to first clock edge = 120 ns
```

### Multimeter

```
  Essential Measurements:
  
  ✓ Voltage at MCU pins (is the GPIO actually going high?)
  ✓ Current consumption (measuring with ammeter in series)
  ✓ Continuity check (find broken traces or solder joints)
  ✓ Resistance (pull-up/pull-down values)
  
  Pro Tip: Use the continuity buzzer first when debugging
           hardware — 80% of board issues are bad connections.
```

---

## 19.6 Debugging Strategies

### The Systematic Approach

```
  1. Reproduce reliably
     → "It sometimes crashes" is not a bug report
     → Find the EXACT trigger conditions
  
  2. Isolate — binary search the problem
     → Comment out half the code. Still crashes? Bug is in remaining half.
     → Repeat until you find the exact line.
  
  3. Form a hypothesis
     → "I think the SPI transfer is corrupting the buffer because..."
  
  4. Test the hypothesis
     → Add a specific check, don't just "try things"
  
  5. Fix and verify
     → Fix the bug AND add a test/assertion to prevent regression
```

### Common Firmware Bug Categories

```
  ┌──────────────────────────────────────────────────────────┐
  │  Category          │  Symptoms         │  Debug Method   │
  ├────────────────────┼───────────────────┼─────────────────┤
  │  Timing            │  Works sometimes  │  Oscilloscope   │
  │  Race condition    │  Fails under load │  Code review    │
  │  Stack overflow    │  Random crashes   │  Stack canary   │
  │  Uninitialized var │  Erratic behavior │  -Wall -Werror  │
  │  Wrong register    │  Peripheral dead  │  Read back reg  │
  │  Interrupt priority│  Missed interrupts│  Check NVIC     │
  │  Missing volatile  │  Optimized away   │  Check ASM      │
  │  Alignment         │  Hard fault       │  Check CFSR     │
  │  Heap exhaustion   │  malloc returns 0 │  Avoid dynamic  │
  │  Watchdog timeout  │  System resets    │  Check WDG reg  │
  └──────────────────────────────────────────────────────────┘
```

### Stack Overflow Detection

```c
/* Fill stack with known pattern at startup (in startup code) */
#define STACK_CANARY  0xDEADBEEFUL

extern uint32_t _estack;
extern uint32_t _Min_Stack_Size;

void stack_paint(void)
{
    uint32_t *stack_bottom = &_estack - (uint32_t)&_Min_Stack_Size / 4;
    uint32_t *p = stack_bottom;

    /* Paint bottom 75% of stack with canary value */
    uint32_t paint_words = ((uint32_t)&_Min_Stack_Size / 4) * 3 / 4;
    for (uint32_t i = 0; i < paint_words; i++)
    {
        *p++ = STACK_CANARY;
    }
}

uint32_t stack_check_usage(void)
{
    uint32_t *stack_bottom = &_estack - (uint32_t)&_Min_Stack_Size / 4;
    uint32_t *p = stack_bottom;
    uint32_t unused = 0;

    while (*p == STACK_CANARY)
    {
        unused++;
        p++;
    }

    return ((uint32_t)&_Min_Stack_Size / 4) - unused;  /* Words used */
}
```

---

## 19.7 Assertions and Defensive Programming

```c
/* Custom assert for embedded systems */
#ifdef DEBUG
#define ASSERT(expr)  do { \
    if (!(expr)) { \
        assert_failed(__FILE__, __LINE__, #expr); \
    } \
} while(0)
#else
#define ASSERT(expr)  ((void)0)  /* Stripped in release */
#endif

void assert_failed(const char *file, uint32_t line, const char *expr)
{
    /* Disable interrupts — we're in an error state */
    __asm volatile ("CPSID I");

    /* Print via UART if available */
    uart_print("\r\n!!! ASSERT FAILED !!!\r\n");
    uart_print("File: "); uart_print(file); uart_print("\r\n");
    uart_print("Expr: "); uart_print(expr); uart_print("\r\n");

    /* Blink LED rapidly to indicate error */
    while (1)
    {
        GPIOA->ODR ^= (1UL << 5);
        for (volatile uint32_t i = 0; i < 100000; i++);
    }
}

/* Usage */
void spi_write_reg(uint8_t reg, uint8_t value)
{
    ASSERT(reg <= 0x7FUL);           /* Valid register range */
    ASSERT(spi_initialized == 1);     /* SPI must be initialized */
    /* ... */
}
```

---

## 19.8 Exercises

**Exercise 19.1:** Implement the Hard Fault handler from Section 19.3. Trigger a hard fault intentionally (e.g., read from address 0xFFFFFFFF) and inspect the crash info in GDB.

**Exercise 19.2:** Measure your UART baud rate with an oscilloscope or logic analyzer. Calculate the error percentage versus the ideal 115200 baud.

**Exercise 19.3:** Implement the stack painting technique. Monitor stack usage over time with `printf` and determine the high-water mark for your application.

**Exercise 19.4:** Connect a logic analyzer to your SPI bus. Capture a register read transaction and verify the timing matches the sensor's datasheet.

---

## 19.9 Chapter Summary

```
  Debugging Hierarchy (try in order):
  
  1. Compiler warnings (-Wall -Wextra -Werror)
  2. Static analysis (PC-lint, cppcheck)
  3. Code review (another pair of eyes)
  4. UART printf (simple but intrusive)
  5. SWO/ITM trace (non-intrusive printf)
  6. GDB breakpoints + watchpoints
  7. Logic analyzer (protocol decode)
  8. Oscilloscope (timing / analog)
  9. Hard Fault handler (crash forensics)
```

---

*Next: [Chapter 20 — Embedded C++ →](./Chapter_20_Embedded_Cpp.md)*
