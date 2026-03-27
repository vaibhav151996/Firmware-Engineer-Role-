# Chapter 3: Development Environment & Toolchain
## Setting Up STM32CubeIDE, GCC, and Your First Debug Session

**Difficulty Level: ★☆☆☆☆ Beginner**
**Estimated Time: 1 week**

---

## 3.1 Concept Overview

### What Is a Toolchain?

A **toolchain** is the set of software tools that converts your C source code into a binary that runs on the microcontroller. Unlike desktop programming where you just hit "Run," embedded development requires a multi-step process:

```
  FROM SOURCE CODE TO RUNNING FIRMWARE:
  
  main.c ─────┐
  gpio.c ─────┤    ┌──────────┐    ┌─────────┐    ┌──────────┐
  uart.c ─────┼───→│ COMPILER │───→│ LINKER  │───→│ BINARY   │
  startup.s ──┤    │ (GCC)    │    │ (LD)    │    │ (.elf)   │
  headers ────┘    └──────────┘    └─────────┘    └────┬─────┘
                        │               │               │
                   Produces .o      Uses linker     ┌───┴────┐
                   object files     script (.ld)    │ FLASHER│
                                    to place code   │(ST-Link)│
                                    in memory       └───┬────┘
                                                        │
                                                        ↓
                                                  ┌──────────┐
                                                  │  STM32   │
                                                  │ FLASH    │
                                                  │ MEMORY   │
                                                  └──────────┘
```

### Why This Matters

Without understanding the toolchain:
- You can't diagnose build errors
- You can't optimize code size (critical when Flash is 64 KB)
- You can't create custom memory layouts
- You can't debug at the instruction level
- You can't work outside of an IDE

### The ARM GCC Toolchain

The toolchain for STM32 (ARM Cortex-M) is called **arm-none-eabi-gcc**:

| Tool | Name | Purpose |
|------|------|---------|
| Compiler | `arm-none-eabi-gcc` | Compiles C/C++ to ARM assembly |
| Assembler | `arm-none-eabi-as` | Assembles .s files to object code |
| Linker | `arm-none-eabi-ld` | Links objects into final binary |
| Object Copy | `arm-none-eabi-objcopy` | Converts ELF to binary/hex |
| Object Dump | `arm-none-eabi-objdump` | Disassembles binary (shows assembly) |
| Size | `arm-none-eabi-size` | Shows memory usage (Flash/RAM) |
| GDB | `arm-none-eabi-gdb` | Debugger |

The "none" means no operating system. "eabi" means Embedded Application Binary Interface.

---

## 3.2 STM32 Hardware Internals

### The Build-Flash-Debug Pipeline

```
  ┌────────────────────────────────────────────────────────────┐
  │                 DEVELOPMENT WORKFLOW                        │
  │                                                            │
  │  ┌─────────┐   ┌──────────┐   ┌───────────┐               │
  │  │  EDIT   │──→│  BUILD   │──→│   FLASH   │               │
  │  │ (IDE)   │   │ (GCC)    │   │ (ST-Link) │               │
  │  └─────────┘   └──────────┘   └─────┬─────┘               │
  │       ↑                              │                     │
  │       │         ┌──────────┐         ↓                     │
  │       └─────────│  DEBUG   │←── ┌─────────┐               │
  │       Fix bugs  │ (GDB +   │    │  RUN    │               │
  │                 │  ST-Link)│    │ on MCU  │               │
  │                 └──────────┘    └─────────┘               │
  └────────────────────────────────────────────────────────────┘
```

### How ST-Link Works

The ST-Link is a **debug probe** built into every Nucleo board:

```
  ┌───────────────────────────────────────────────────┐
  │                NUCLEO BOARD                        │
  │                                                   │
  │  ┌──────────────┐         ┌──────────────────┐   │
  │  │   ST-LINK    │  SWD    │    TARGET MCU    │   │
  │  │   V2-1       │────────→│    STM32F401     │   │
  │  │              │ SWDIO   │                  │   │
  │  │  USB ←──────→│ SWCLK   │    Your code    │   │
  │  │  (to PC)     │         │    runs here!   │   │
  │  └──────────────┘         └──────────────────┘   │
  │                                                   │
  │  SWD = Serial Wire Debug (2-wire debug protocol) │
  │  SWDIO = Data line                                │
  │  SWCLK = Clock line                               │
  └───────────────────────────────────────────────────┘
  
  What ST-Link can do:
  ├── Flash firmware to the MCU's Flash memory
  ├── Start/stop/reset the CPU
  ├── Set breakpoints
  ├── Read/write CPU registers
  ├── Read/write any memory address
  ├── Single-step through instructions
  └── Provide a Virtual COM Port (VCP) for UART
```

---

## 3.3 Step-by-Step Implementation

### 3.3.1 Installing STM32CubeIDE

**Step 1: Download**
1. Go to https://www.st.com/en/development-tools/stm32cubeide.html
2. Download the version for your OS (Windows/macOS/Linux)
3. You'll need a free ST.com account

**Step 2: Install**
- Windows: Run the installer, accept all defaults
- The installer includes: ARM GCC compiler, ST-Link drivers, GDB debugger

**Step 3: First Launch**
- Choose a workspace directory (e.g., `C:\STM32_Projects`)
- Accept the Information Center (or close it)

### 3.3.2 Creating an Empty Project

```
  Step-by-step project creation:
  
  File → New → STM32 Project
       │
       ↓
  ┌─────────────────────────────────────┐
  │ Target Selection                    │
  │                                     │
  │ Tab: "Board Selector"               │
  │ Search: NUCLEO-F401RE               │
  │ Select it → Click "Next"            │
  └───────────────┬─────────────────────┘
                  ↓
  ┌─────────────────────────────────────┐
  │ Project Setup                       │
  │                                     │
  │ Project Name: my_project            │
  │ Targeted Language: C                │
  │ Targeted Binary: Executable         │
  │ Targeted Project Type: Empty        │ ← NOT "STM32Cube"!
  │                                     │
  │ Click "Finish"                      │
  └───────────────┬─────────────────────┘
                  ↓
  ┌─────────────────────────────────────┐
  │ Generated Project Structure:        │
  │                                     │
  │ my_project/                         │
  │ ├── Inc/           (header files)   │
  │ ├── Src/                            │
  │ │   ├── main.c     (YOUR CODE)      │
  │ │   ├── syscalls.c (system stubs)   │
  │ │   └── sysmem.c   (memory stubs)   │
  │ ├── Startup/                        │
  │ │   └── startup_stm32f401retx.s     │
  │ └── STM32F401RETX_FLASH.ld         │
  └─────────────────────────────────────┘
```

### 3.3.3 Understanding the Generated Files

**startup_stm32f401retx.s** — The startup file (ARM assembly):
```
  What it does:
  1. Defines the initial stack pointer
  2. Defines the vector table (interrupt handler addresses)
  3. Implements Reset_Handler:
     a. Copies .data section from Flash to RAM
     b. Zeros out .bss section
     c. Calls SystemInit() (if defined)
     d. Calls main()
  4. Implements default handlers for all interrupts
```

**STM32F401RETX_FLASH.ld** — The linker script:
```
  What it defines:
  1. MEMORY regions:
     FLASH: 512KB starting at 0x08000000
     RAM:   96KB starting at 0x20000000
  
  2. SECTIONS placement:
     .text    → FLASH (code)
     .rodata  → FLASH (read-only data)
     .data    → RAM (initialized, copied from Flash at startup)
     .bss     → RAM (zero-initialized)
     ._user_heap_stack → RAM (stack/heap)
```

### 3.3.4 Building the Project

**Method 1: IDE**
- Click the hammer icon (Build) or press `Ctrl+B`
- Check the Console view for output

**Method 2: Command Line**
```bash
# From the project directory:
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 \
    -mfloat-abi=hard -O0 -g3 -Wall \
    -c Src/main.c -o build/main.o

arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 \
    -mfloat-abi=hard -T STM32F401RETX_FLASH.ld \
    -Wl,-Map=output.map \
    build/main.o Startup/startup_stm32f401retx.o \
    -o build/firmware.elf

arm-none-eabi-objcopy -O binary build/firmware.elf build/firmware.bin
```

**Understanding compiler flags:**

| Flag | Meaning |
|------|---------|
| `-mcpu=cortex-m4` | Target CPU architecture |
| `-mthumb` | Use Thumb instruction set (16/32-bit mixed) |
| `-mfpu=fpv4-sp-d16` | Enable hardware FPU |
| `-mfloat-abi=hard` | Use hardware floating point |
| `-O0` | No optimization (best for debugging) |
| `-O2` | Optimize for speed (production) |
| `-Os` | Optimize for size (production, tight Flash) |
| `-g3` | Maximum debug info |
| `-Wall` | All warnings |
| `-c` | Compile only, don't link |
| `-T script.ld` | Use this linker script |
| `-Wl,-Map=file.map` | Generate memory map file |

### 3.3.5 Checking Memory Usage

After building, check how much Flash and RAM your firmware uses:

```bash
arm-none-eabi-size build/firmware.elf
```

Output:
```
   text    data     bss     dec     hex filename
    832       8       4     844     34c build/firmware.elf
```

```
  Understanding the size output:
  
  text = 832 bytes   → Code + read-only data → goes to FLASH
  data =   8 bytes   → Initialized globals   → copied to RAM at startup
  bss  =   4 bytes   → Uninitialized globals → zeroed in RAM at startup
  
  Flash usage: text + data = 840 bytes (out of 512 KB → 0.16%)
  RAM usage:   data + bss  =  12 bytes (out of 96 KB  → 0.01%)
  
  Your LED blink uses less than 1 KB of Flash!
```

### 3.3.6 Flashing the Firmware

**Method 1: STM32CubeIDE**
- Click Run → Run (or the green ▶ button)
- Or click Run → Debug (F11) for debug mode

**Method 2: ST-Link Utility / STM32CubeProgrammer**
- Open STM32CubeProgrammer
- Connect to the board
- Load the .bin or .elf file
- Click "Start Programming"

**Method 3: Command Line (OpenOCD)**
```bash
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
    -c "program build/firmware.elf verify reset exit"
```

### 3.3.7 Your First Debug Session

```
  Debug Workflow:
  
  ┌──────────────────────────────────────────────────────┐
  │  1. Set a breakpoint                                  │
  │     Double-click the left margin of a line in main() │
  │     A blue circle appears = breakpoint set            │
  └──────────────────┬───────────────────────────────────┘
                     ↓
  ┌──────────────────────────────────────────────────────┐
  │  2. Start debug session                               │
  │     F11 or Run → Debug                                │
  │     Code flashes to board, CPU halts at main()       │
  └──────────────────┬───────────────────────────────────┘
                     ↓
  ┌──────────────────────────────────────────────────────┐
  │  3. Step through code                                 │
  │     F5 = Step Into (enter functions)                  │
  │     F6 = Step Over (execute line, skip functions)     │
  │     F7 = Step Return (exit current function)          │
  │     F8 = Resume (run until next breakpoint)           │
  └──────────────────┬───────────────────────────────────┘
                     ↓
  ┌──────────────────────────────────────────────────────┐
  │  4. Inspect state                                     │
  │     Variables view: see local/global variable values   │
  │     SFRs view: see peripheral register values         │
  │     Memory view: browse raw memory                    │
  │     Disassembly view: see actual ARM instructions     │
  └──────────────────┬───────────────────────────────────┘
                     ↓
  ┌──────────────────────────────────────────────────────┐
  │  5. Stop/Restart                                      │
  │     Ctrl+F2 = Stop debugging                          │
  │     You can also reset the board or press its button  │
  └──────────────────────────────────────────────────────┘
```

**Key debug views in STM32CubeIDE:**

| View | How to Open | What It Shows |
|------|-------------|---------------|
| Variables | Automatic in Debug perspective | Local and global variable values |
| Expressions | Window → Show View → Expressions | Custom watch expressions |
| Registers | Window → Show View → Registers | CPU core registers (R0-R15, SP, PC) |
| SFRs | Window → Show View → SFRs | Peripheral registers (GPIOA, RCC, etc.) |
| Memory Browser | Window → Show View → Memory Browser | Raw memory at any address |
| Disassembly | Window → Show View → Disassembly | ARM assembly instructions |

### 3.3.8 Reading the Map File

The `.map` file (generated with `-Wl,-Map=output.map`) shows you exactly where everything is placed in memory:

```
  Map file excerpt (simplified):

  .text           0x08000000    0x2F0
   startup_stm32f401retx.o(.isr_vector)
                  0x08000000    0x188     Vector table
   main.o(.text)  0x08000188    0x098     Your code
   startup.o(.text) 0x08000220  0x0D0     Startup code

  .data           0x20000000    0x008
   main.o(.data)  0x20000000    0x004     Your globals

  .bss            0x20000008    0x004
   main.o(.bss)   0x20000008    0x004     Your uninitialized globals

  This tells you:
  - Vector table starts at Flash base (0x08000000)
  - Your main() code is at 0x08000188
  - Total code size is ~0x2F0 = 752 bytes
```

---

## 3.4 Practical STM32 Nucleo Example

### Complete IDE Walkthrough: Build, Flash, and Debug the LED Blink

1. Create the project as described in 3.3.2
2. Paste the bare-metal LED blink code from Chapter 1 into `Src/main.c`
3. Build (`Ctrl+B`) — verify 0 errors
4. Set a breakpoint on the first line of `main()`
5. Press F11 to start debugging
6. In the SFR view, expand "RCC" → find "AHB1ENR"
7. Step over (`F6`) the clock enable line
8. Observe: AHB1ENR bit 0 changes from 0 to 1
9. Step over the MODER configuration lines
10. Expand "GPIOA" → "MODER" in SFRs
11. Verify bits [11:10] = 01
12. Press F8 to resume — the LED should start blinking
13. Press the pause button (⏸) while the LED is on
14. Check GPIOA → ODR → bit 5 should be 1

---

## 3.5 Diagrams

### The Compilation Pipeline

```
  Detailed compilation flow:
  
  Source Files (C):
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  main.c  │  │  gpio.c  │  │  uart.c  │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │              │              │
       ↓              ↓              ↓
  ┌──────────────────────────────────────────┐
  │          PREPROCESSOR (cpp)              │
  │  - Expands #include                      │
  │  - Expands #define macros                │
  │  - Processes #ifdef conditionals         │
  └─────────────────┬────────────────────────┘
                    ↓
  ┌──────────────────────────────────────────┐
  │          COMPILER (cc1)                  │
  │  - Translates C to ARM assembly          │
  │  - Applies optimizations (-O2, -Os)      │
  │  - Generates .s files                    │
  └─────────────────┬────────────────────────┘
                    ↓
  ┌──────────────────────────────────────────┐
  │          ASSEMBLER (as)                  │
  │  - Translates assembly to machine code   │
  │  - Generates .o (object) files           │
  │  - Symbols are unresolved at this stage  │
  └─────────────────┬────────────────────────┘
                    ↓
  ┌──────────────────────────────────────────┐
  │          LINKER (ld)                     │
  │  - Combines all .o files                 │
  │  - Resolves symbol references            │
  │  - Places code/data per linker script    │
  │  - Generates .elf (with debug info)      │
  │  - Generates .map (memory layout)        │
  └─────────────────┬────────────────────────┘
                    ↓
  ┌──────────────────────────────────────────┐
  │          OBJCOPY                         │
  │  - Strips debug info                     │
  │  - Creates .bin (raw binary)             │
  │  - Creates .hex (Intel HEX format)       │
  └─────────────────┬────────────────────────┘
                    ↓
              firmware.bin
          (ready to flash!)
```

---

## 3.6 Common Mistakes & Debugging

| Mistake | Symptom | Solution |
|---------|---------|----------|
| Wrong board selected | Build succeeds but won't flash | Verify project target in properties |
| Using "STM32Cube" project type | Too many generated files | Use "Empty" project for bare-metal |
| Optimization level too high for debug | Variables show "optimized out" | Use `-O0` for debug builds |
| ST-Link drivers not installed | "No ST-Link detected" | Install from st.com/stlink |
| Board USB not recognized | Nothing in device manager | Try different USB cable (data cable, not charge-only) |
| Windows USB power issues | Intermittent connection | Use a USB hub with external power |

---

## 3.7 Exercises

**Exercise 3.1:** Build the LED blink project and examine the `.map` file. Find where `main()` is placed in memory. What address is it at?

**Exercise 3.2:** Use `arm-none-eabi-size` to compare builds at `-O0`, `-O1`, `-O2`, and `-Os` optimization levels. Create a table of the results.

**Exercise 3.3:** In the debugger, set a breakpoint inside the delay function. How many times does the CPU enter the delay function in 10 seconds?

**Exercise 3.4:** Use `arm-none-eabi-objdump -d firmware.elf` to view the disassembled code. Find the instructions that correspond to `GPIOA_ODR ^= (1UL << 5)`. How many ARM instructions does this C line produce?

**Exercise 3.5:** Intentionally introduce a bug (wrong pin number or missing clock enable). Use the debugger to identify and fix the issue.

---

## 3.8 Industry & Career Notes

- Professional teams use **CI/CD build servers** that compile firmware automatically on every git push
- Many companies use **command-line builds** (Make/CMake) rather than IDE builds — understanding the GCC flags is essential
- The `.map` file is your best friend for **tracking memory usage** — many projects have strict Flash/RAM budgets
- **Debug builds** (-O0) are typically 2-3× larger than release builds (-Os) — always check both
- In safety-critical development, every compiler flag must be **documented and justified**

---

## 3.9 Chapter Summary

```
  ┌─────────────────────────────────────────────────────────┐
  │                  CHAPTER 3 SUMMARY                       │
  │                                                         │
  │  ✓ The ARM GCC toolchain (arm-none-eabi-gcc) compiles   │
  │    C code into ARM machine code for the STM32           │
  │                                                         │
  │  ✓ The build pipeline: Preprocess → Compile → Assemble  │
  │    → Link → Objcopy → Flash                             │
  │                                                         │
  │  ✓ STM32CubeIDE integrates editor, compiler, debugger   │
  │    and flasher in one environment                        │
  │                                                         │
  │  ✓ The linker script (.ld) defines where code and data  │
  │    are placed in Flash and RAM                           │
  │                                                         │
  │  ✓ ST-Link provides SWD debugging: breakpoints,         │
  │    stepping, register/memory inspection                  │
  │                                                         │
  │  ✓ arm-none-eabi-size shows Flash and RAM usage          │
  │                                                         │
  │  ✓ Understanding the toolchain is essential for          │
  │    debugging, optimization, and professional work        │
  └─────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 4 — GPIO Register-Level Mastery →](./Chapter_04_GPIO_Mastery.md)*
