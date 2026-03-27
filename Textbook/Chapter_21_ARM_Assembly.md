# Chapter 21: ARM Assembly Essentials

**Difficulty Level: ★★★★☆ Advanced**  
**Estimated Time: 2–3 weeks**

---

## 21.1 Concept Overview

### Why Learn Assembly?

You will rarely **write** assembly, but you must be able to **read** it:

```
  When you NEED assembly knowledge:
  
  ✓ Reading disassembly to debug optimized code
  ✓ Understanding Hard Fault crash dumps
  ✓ Verifying compiler output (did it optimize correctly?)
  ✓ Writing time-critical routines (rare but important)
  ✓ Understanding startup code and context switching
  ✓ Interview questions ("What happens at reset?")
  ✓ Bootloaders and secure boot (jump-to-app sequence)
```

### ARM Cortex-M4 Architecture

```
  CPU Registers (ARM Cortex-M4):
  ┌──────────┬──────────────────────────────────────────────┐
  │ Register │ Purpose                                      │
  ├──────────┼──────────────────────────────────────────────┤
  │ R0-R3    │ Arguments & return values (caller-saved)     │
  │ R4-R11   │ General purpose (callee-saved)               │
  │ R12 (IP) │ Intra-procedure scratch register             │
  │ R13 (SP) │ Stack Pointer (MSP or PSP)                   │
  │ R14 (LR) │ Link Register (return address)               │
  │ R15 (PC) │ Program Counter (current instruction)        │
  │ xPSR     │ Program Status Register (flags: N,Z,C,V)    │
  └──────────┴──────────────────────────────────────────────┘
  
  AAPCS Calling Convention:
    R0-R3:  Function arguments (first 4) and return value (R0)
    R4-R11: Must be preserved across function calls
    R13:    Must be 8-byte aligned at function entry
    R14:    Return address (set by BL instruction)
  
  ┌─────────────────────────────────────────────────────┐
  │  Example: int add(int a, int b)                     │
  │                                                     │
  │  R0 = a (first argument)                            │
  │  R1 = b (second argument)                           │
  │  R0 = return value                                  │
  │                                                     │
  │  Assembly:                                          │
  │    add:                                             │
  │      ADD R0, R0, R1     ; R0 = R0 + R1              │
  │      BX  LR             ; Return                    │
  └─────────────────────────────────────────────────────┘
```

---

## 21.2 Instruction Set Basics (Thumb-2)

### Data Processing Instructions

```arm
; ── Move ──
MOV   R0, #42          ; R0 = 42
MOV   R1, R0           ; R1 = R0
MVN   R0, R1           ; R0 = ~R1 (bitwise NOT)
MOVW  R0, #0x1234      ; Move 16-bit immediate to lower half
MOVT  R0, #0x5678      ; Move 16-bit immediate to upper half
                        ; Result: R0 = 0x56781234

; ── Arithmetic ──
ADD   R0, R1, R2       ; R0 = R1 + R2
ADD   R0, R0, #1       ; R0 = R0 + 1
ADDS  R0, R1, R2       ; R0 = R1 + R2 (update flags: N,Z,C,V)
SUB   R0, R1, R2       ; R0 = R1 - R2
RSB   R0, R1, #0       ; R0 = 0 - R1 (negate)
MUL   R0, R1, R2       ; R0 = R1 * R2
MLA   R0, R1, R2, R3   ; R0 = (R1 * R2) + R3 (multiply-accumulate)

; ── Logic ──
AND   R0, R1, R2       ; R0 = R1 & R2
ORR   R0, R1, R2       ; R0 = R1 | R2
EOR   R0, R1, R2       ; R0 = R1 ^ R2
BIC   R0, R1, R2       ; R0 = R1 & ~R2 (bit clear)
ORN   R0, R1, R2       ; R0 = R1 | ~R2

; ── Shifts ──
LSL   R0, R1, #3       ; R0 = R1 << 3 (logical shift left)
LSR   R0, R1, #3       ; R0 = R1 >> 3 (logical shift right, zero fill)
ASR   R0, R1, #3       ; R0 = R1 >> 3 (arithmetic, sign-extend)
ROR   R0, R1, #3       ; Rotate right by 3

; ── Compare (updates flags but discards result) ──
CMP   R0, R1           ; Flags = R0 - R1
CMP   R0, #0           ; Test if R0 == 0
CMN   R0, R1           ; Flags = R0 + R1
TST   R0, R1           ; Flags = R0 & R1 (test bits)
TEQ   R0, R1           ; Flags = R0 ^ R1 (test equal)
```

### Memory Access Instructions

```arm
; ── Load (memory → register) ──
LDR   R0, [R1]         ; R0 = *(uint32_t *)R1
LDR   R0, [R1, #4]     ; R0 = *(uint32_t *)(R1 + 4)
LDR   R0, [R1, #4]!    ; R1 += 4; R0 = *R1 (pre-increment)
LDR   R0, [R1], #4     ; R0 = *R1; R1 += 4 (post-increment)

LDRB  R0, [R1]         ; Load byte (8-bit, zero-extended)
LDRH  R0, [R1]         ; Load halfword (16-bit, zero-extended)
LDRSB R0, [R1]         ; Load byte (sign-extended)
LDRSH R0, [R1]         ; Load halfword (sign-extended)

; ── Store (register → memory) ──
STR   R0, [R1]         ; *(uint32_t *)R1 = R0
STR   R0, [R1, #4]     ; *(uint32_t *)(R1 + 4) = R0
STRB  R0, [R1]         ; Store byte
STRH  R0, [R1]         ; Store halfword

; ── Multiple Load/Store (for stack operations) ──
PUSH  {R4-R7, LR}      ; Push registers onto stack
POP   {R4-R7, PC}      ; Pop registers (PC = return)
STMDB SP!, {R4-R11, LR} ; Same as PUSH (store multiple, decrement before)
LDMIA SP!, {R4-R11, PC} ; Same as POP (load multiple, increment after)

; ── PC-Relative Load (constants) ──
LDR   R0, =0x40020000  ; Load address via literal pool
                        ; Assembler places the constant nearby and
                        ; generates: LDR R0, [PC, #offset]
```

### Branch Instructions

```arm
; ── Unconditional ──
B     label             ; Branch (jump)
BL    function          ; Branch with Link (call function, LR = return addr)
BX    LR                ; Branch to address in LR (return from function)
BLX   R0                ; Call function at address in R0

; ── Conditional (use after CMP/TST/ADDS) ──
BEQ   label             ; Branch if Equal (Z=1)
BNE   label             ; Branch if Not Equal (Z=0)
BCS   label             ; Branch if Carry Set (unsigned >=)
BCC   label             ; Branch if Carry Clear (unsigned <)
BMI   label             ; Branch if Minus (N=1)
BPL   label             ; Branch if Plus (N=0)
BVS   label             ; Branch if Overflow Set
BVC   label             ; Branch if Overflow Clear
BHI   label             ; Branch if Higher (unsigned >)
BLS   label             ; Branch if Lower or Same (unsigned <=)
BGE   label             ; Branch if Greater or Equal (signed)
BLT   label             ; Branch if Less Than (signed)
BGT   label             ; Branch if Greater Than (signed)
BLE   label             ; Branch if Less or Equal (signed)

; ── IT Block (If-Then) — Cortex-M specialty ──
CMP   R0, #0
ITE   EQ                ; If-Then-Else based on Equal
MOVEQ R1, #1            ; If R0 == 0: R1 = 1
MOVNE R1, #0            ; Else:        R1 = 0
```

---

## 21.3 Reading Compiler Output

### How to Generate Assembly Listing

```bash
# Method 1: Compiler flag to generate .s file alongside .o
arm-none-eabi-gcc -S -O2 main.c -o main.s

# Method 2: Disassemble the .elf file
arm-none-eabi-objdump -d firmware.elf > firmware.lst

# Method 3: Interleave C source with assembly
arm-none-eabi-objdump -d -S firmware.elf > firmware_annotated.lst

# Method 4: Just one function
arm-none-eabi-objdump -d firmware.elf | grep -A 30 "<gpio_toggle>:"
```

### Example: C to Assembly

```c
/* C code */
void gpio_toggle(void)
{
    volatile uint32_t *odr = (volatile uint32_t *)0x40020014;
    *odr ^= (1 << 5);
}
```

```arm
; Compiler output (arm-none-eabi-gcc -O2):
gpio_toggle:
    LDR   R1, =0x40020014    ; R1 = address of GPIOA->ODR
    LDR   R0, [R1]           ; R0 = current ODR value
    EOR   R0, R0, #32        ; R0 ^= (1 << 5) = 0x20
    STR   R0, [R1]           ; Write back to ODR
    BX    LR                 ; Return

; Literal pool (constant stored near the function):
    .word 0x40020014
```

### Example: Struct Access

```c
/* C code */
typedef struct {
    volatile uint32_t MODER;    /* Offset 0x00 */
    volatile uint32_t OTYPER;   /* Offset 0x04 */
    volatile uint32_t OSPEEDR;  /* Offset 0x08 */
    volatile uint32_t PUPDR;    /* Offset 0x0C */
    volatile uint32_t IDR;      /* Offset 0x10 */
    volatile uint32_t ODR;      /* Offset 0x14 */
    volatile uint32_t BSRR;     /* Offset 0x18 */
} GPIO_TypeDef;

#define GPIOA ((GPIO_TypeDef *)0x40020000)

void led_on(void)
{
    GPIOA->BSRR = (1 << 5);
}
```

```arm
; Compiler output — identical to raw pointer access!
led_on:
    LDR   R0, =0x40020018    ; R0 = address of GPIOA->BSRR (base + 0x18)
    MOV   R1, #32            ; R1 = (1 << 5) = 0x20
    STR   R1, [R0]           ; GPIOA->BSRR = 0x20
    BX    LR
```

---

## 21.4 The Startup Sequence in Assembly

### What Happens at Reset

```arm
; Vector Table (first entries in Flash at 0x08000000)
;
; The vector table is an array of 32-bit function pointers.
; The CPU reads these to know where to begin execution.

    .section .isr_vector, "a"
    .word  _estack           ; 0x00: Initial Stack Pointer
    .word  Reset_Handler     ; 0x04: Reset vector — ENTRY POINT
    .word  NMI_Handler       ; 0x08: Non-Maskable Interrupt
    .word  HardFault_Handler ; 0x0C: Hard Fault
    .word  MemManage_Handler ; 0x10: Memory Management
    .word  BusFault_Handler  ; 0x14: Bus Fault
    .word  UsageFault_Handler; 0x18: Usage Fault
    ; ... more handlers ...

; =======================================================================
; Reset Handler — first code that runs
; =======================================================================
    .section .text
    .global Reset_Handler
    .type   Reset_Handler, %function
Reset_Handler:

    ; Step 1: Copy .data from Flash to RAM
    ;   .data contains initialized global variables
    ;   The linker places initial values in Flash
    ;   We must copy them to RAM at startup
    LDR   R0, =_sdata       ; Destination: RAM .data start
    LDR   R1, =_edata       ; Destination: RAM .data end
    LDR   R2, =_sidata      ; Source: Flash (LMA of .data)

.copy_data:
    CMP   R0, R1            ; Done?
    BGE   .zero_bss         ; Yes → move to BSS
    LDR   R3, [R2], #4      ; Load word from Flash, advance pointer
    STR   R3, [R0], #4      ; Store to RAM, advance pointer
    B     .copy_data

    ; Step 2: Zero out .bss
    ;   .bss contains uninitialized globals
    ;   C standard requires them to be zero
.zero_bss:
    LDR   R0, =_sbss        ; BSS start
    LDR   R1, =_ebss        ; BSS end
    MOV   R2, #0

.zero_bss_loop:
    CMP   R0, R1
    BGE   .call_main
    STR   R2, [R0], #4
    B     .zero_bss_loop

    ; Step 3: Call main()
.call_main:
    BL    main               ; Call main (LR = return address)

    ; Step 4: If main returns, loop forever
.loop_forever:
    B     .loop_forever
```

---

## 21.5 Inline Assembly in C

### Basic Syntax

```c
/* GNU C inline assembly syntax */
__asm volatile (
    "assembly template"
    : output operands       /* optional */
    : input operands        /* optional */
    : clobbered registers   /* optional */
);

/* ── Simple examples ── */

/* NOP (no operation) */
__asm volatile ("NOP");

/* Disable interrupts */
__asm volatile ("CPSID I" ::: "memory");

/* Enable interrupts */
__asm volatile ("CPSIE I" ::: "memory");

/* Data Synchronization Barrier */
__asm volatile ("DSB" ::: "memory");

/* Instruction Synchronization Barrier */
__asm volatile ("ISB" ::: "memory");

/* Wait For Interrupt (enter sleep mode) */
__asm volatile ("WFI");
```

### With Operands

```c
/* Read PRIMASK register (interrupt status) */
static inline uint32_t get_primask(void)
{
    uint32_t result;
    __asm volatile ("MRS %0, PRIMASK" : "=r" (result));
    return result;
}

/* Set Main Stack Pointer */
static inline void set_msp(uint32_t stack_ptr)
{
    __asm volatile ("MSR MSP, %0" : : "r" (stack_ptr));
}

/* Read cycle counter (DWT) */
static inline uint32_t get_cycles(void)
{
    return *(volatile uint32_t *)0xE0001004UL;  /* DWT_CYCCNT */
}

/* Critical section with save/restore */
static inline uint32_t enter_critical(void)
{
    uint32_t primask;
    __asm volatile (
        "MRS   %0, PRIMASK \n"
        "CPSID I           \n"
        : "=r" (primask)
        :
        : "memory"
    );
    return primask;
}

static inline void exit_critical(uint32_t primask)
{
    __asm volatile ("MSR PRIMASK, %0" : : "r" (primask) : "memory");
}

/* Usage */
void safe_increment(volatile uint32_t *counter)
{
    uint32_t state = enter_critical();
    (*counter)++;
    exit_critical(state);
}
```

---

## 21.6 Context Switching (How RTOS Works)

```arm
; ===================================================================
; PendSV_Handler — Context switch between tasks (simplified)
;
; When PendSV fires:
;   CPU automatically pushes R0-R3, R12, LR, PC, xPSR onto PSP
;   We must manually save R4-R11 and switch the PSP
; ===================================================================

    .global PendSV_Handler
    .type   PendSV_Handler, %function
PendSV_Handler:
    ; ---- Save current task context ----
    MRS    R0, PSP              ; R0 = current task's stack pointer
    STMDB  R0!, {R4-R11}        ; Push R4-R11 onto current task's stack
                                ; (CPU already pushed R0-R3, R12, LR, PC, xPSR)

    ; Save updated PSP to current task's TCB
    LDR    R1, =current_tcb     ; R1 = &current_tcb
    LDR    R2, [R1]             ; R2 = current_tcb (pointer to TCB)
    STR    R0, [R2]             ; current_tcb->sp = R0

    ; ---- Load next task context ----
    PUSH   {LR}                 ; Save LR (we'll call C function)
    BL     scheduler_next_task  ; C function: picks next task,
                                ;   updates current_tcb
    POP    {LR}

    LDR    R1, =current_tcb
    LDR    R2, [R1]             ; R2 = new current_tcb
    LDR    R0, [R2]             ; R0 = new task's saved stack pointer

    LDMIA  R0!, {R4-R11}        ; Pop R4-R11 from new task's stack
    MSR    PSP, R0              ; Set PSP to new task's stack

    ; Return — CPU restores R0-R3, R12, LR, PC, xPSR from new PSP
    BX     LR

; ===================================================================
; Task Stack Layout (after full context save):
;
; High Address
;   ┌──────────┐ ← Original PSP
;   │ xPSR     │   (auto-pushed by hardware)
;   │ PC       │
;   │ LR       │
;   │ R12      │
;   │ R3       │
;   │ R2       │
;   │ R1       │
;   │ R0       │
;   ├──────────┤
;   │ R11      │   (manually pushed by PendSV)
;   │ R10      │
;   │ R9       │
;   │ R8       │
;   │ R7       │
;   │ R6       │
;   │ R5       │
;   │ R4       │
;   └──────────┘ ← Saved PSP (stored in TCB)
; Low Address
; ===================================================================
```

---

## 21.7 Cycle-Counted Delay

```arm
; Precise delay loop — useful for bit-banged protocols
; Each iteration = exactly 4 cycles

    .global delay_cycles
    .type   delay_cycles, %function
delay_cycles:
    ; R0 = number of 4-cycle iterations
    SUBS  R0, R0, #1        ; 1 cycle
    NOP                      ; 1 cycle
    BNE   delay_cycles       ; 1 cycle (not taken) or 2 cycles (taken)
                             ; Total per iteration: 4 cycles
    BX    LR
```

```c
/* C wrapper */
extern void delay_cycles(uint32_t iterations);

/* Delay in microseconds at 84 MHz */
static inline void delay_us(uint32_t us)
{
    delay_cycles(us * 21);  /* 84 MHz / 4 cycles = 21 iterations per us */
}
```

---

## 21.8 Common Assembly Patterns in Disassembly

```arm
; Pattern 1: Function Prologue / Epilogue
my_function:
    PUSH  {R4-R7, LR}       ; Save callee-saved registers + return address
    ; ... function body ...
    POP   {R4-R7, PC}       ; Restore registers and return (PC = saved LR)

; Pattern 2: Switch Statement (jump table)
    TBB   [PC, R0]          ; Table Branch Byte — index in R0
    .byte (case0 - base) / 2
    .byte (case1 - base) / 2
    .byte (case2 - base) / 2

; Pattern 3: Bit manipulation (register set/clear)
    LDR   R0, =0x40020018   ; GPIOA->BSRR
    MOV   R1, #0x20          ; Bit 5
    STR   R1, [R0]          ; Set PA5

; Pattern 4: Volatile read (compiler won't optimize away)
    LDR   R0, [R1]          ; Load from memory-mapped register
    LDR   R0, [R1]          ; Load AGAIN (volatile = must re-read)

; Pattern 5: Loop with counter
    MOV   R0, #100
.loop:
    ; ... loop body ...
    SUBS  R0, R0, #1        ; Decrement and set flags
    BNE   .loop              ; Branch if not zero

; Pattern 6: 64-bit addition (R0:R1 = R2:R3 + R4:R5)
    ADDS  R0, R2, R4        ; Low word + carry flag
    ADC   R1, R3, R5        ; High word + carry
```

---

## 21.9 Exercises

**Exercise 21.1:** Compile your LED blink program with `-O0` and `-O2`. Disassemble both and compare the `main()` function. How many instructions does the optimizer remove?

**Exercise 21.2:** Write an inline assembly function that counts the number of set bits in a `uint32_t` (use the CLZ instruction and a loop, or the RBIT instruction for reverse).

**Exercise 21.3:** Read the disassembly of your Hard Fault handler from Chapter 19. Trace through each instruction manually to understand how it extracts the stack frame.

**Exercise 21.4:** Measure the exact execution time (in CPU cycles) of your `gpio_toggle()` function using `DWT_CYCCNT`. Enable the DWT counter and read it before and after the function call.

---

## 21.10 Chapter Summary

```
  ARM Assembly Checklist:
  
  ✓ Register convention: R0-R3 = args, R4-R11 = callee-saved
  ✓ Load/Store: LDR (read memory), STR (write memory)
  ✓ Branch: B (jump), BL (call), BX LR (return)
  ✓ Flags: ADDS/CMP set condition codes, Bcc uses them
  ✓ Stack: PUSH/POP = STMDB/LDMIA on SP
  ✓ Startup: Copy .data, zero .bss, call main
  ✓ Inline asm: __asm volatile ("instr" : out : in : clobber)
  
  You don't need to WRITE assembly often.
  You MUST be able to READ disassembly output.
```

---

*Next: [Chapter 22 — Python for Firmware Engineering →](./Chapter_22_Python_FW.md)*
