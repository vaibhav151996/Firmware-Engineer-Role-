# Chapter 15: Build Systems — GCC, Make, CMake
## Professional Build Workflows for Firmware Projects

**Difficulty Level: ★★★☆☆ Intermediate**  
**Estimated Time: 1–2 weeks**

---

## 15.1 Concept Overview

### Why Build Systems?

IDE "click to build" works for learning, but professional firmware projects need **reproducible, automated, and scriptable** builds.

```
  Build System Evolution:
  
  Level 0: IDE Button    → Works, but not scriptable
  Level 1: Shell Script  → Scriptable, but ugly
  Level 2: Makefile      → Industry standard for firmware
  Level 3: CMake         → Cross-platform, modern projects
  Level 4: CI/CD         → Automated builds on every commit
```

---

## 15.2 The GCC Compilation Process in Detail

```
  arm-none-eabi-gcc — Complete flag reference for STM32F401:
  
  ARCHITECTURE FLAGS:
    -mcpu=cortex-m4      Target Cortex-M4 core
    -mthumb              Use Thumb-2 instruction set
    -mfpu=fpv4-sp-d16    Hardware FPU (single precision)
    -mfloat-abi=hard     Use FPU registers for float args
  
  OPTIMIZATION FLAGS:
    -O0    No optimization (debug builds)
    -O1    Basic optimization
    -O2    Full optimization (speed)
    -Os    Optimize for size (production!)
    -Og    Optimize for debugging (newer GCC)
    -flto  Link-Time Optimization
  
  WARNING FLAGS:
    -Wall         All standard warnings
    -Wextra       Extra warnings
    -Werror       Treat warnings as errors (strict)
    -Wpedantic    ISO C compliance
    -Wshadow      Warn on variable shadowing
  
  DEBUG FLAGS:
    -g3    Maximum debug info
    -gdwarf-4   DWARF v4 debug format
  
  LINKER FLAGS:
    -T script.ld            Use linker script
    -Wl,-Map=output.map     Generate map file
    -Wl,--gc-sections       Remove unused sections
    -specs=nosys.specs      Don't link system calls
    -specs=nano.specs       Use newlib-nano (smaller printf)
```

---

## 15.3 Step-by-Step: Makefile for STM32

```makefile
###############################################################################
# Makefile for STM32F401RE Bare-Metal Project
###############################################################################

# Project name
PROJECT = firmware

# Toolchain
CC = arm-none-eabi-gcc
AS = arm-none-eabi-gcc
LD = arm-none-eabi-gcc
OBJCOPY = arm-none-eabi-objcopy
SIZE = arm-none-eabi-size

# Directories
SRC_DIR = Src
INC_DIR = Inc
BUILD_DIR = build
STARTUP_DIR = Startup

# Source files
C_SOURCES = $(wildcard $(SRC_DIR)/*.c)
ASM_SOURCES = $(wildcard $(STARTUP_DIR)/*.s)

# Object files
OBJECTS = $(C_SOURCES:$(SRC_DIR)/%.c=$(BUILD_DIR)/%.o)
OBJECTS += $(ASM_SOURCES:$(STARTUP_DIR)/%.s=$(BUILD_DIR)/%.o)

# MCU flags
MCU = -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard

# Compiler flags
CFLAGS = $(MCU) -Wall -Wextra -Os -g3
CFLAGS += -I$(INC_DIR)
CFLAGS += -ffunction-sections -fdata-sections
CFLAGS += -DSTM32F401xE

# Assembler flags
ASFLAGS = $(MCU) -g3

# Linker flags
LDFLAGS = $(MCU) -T STM32F401RETX_FLASH.ld
LDFLAGS += -Wl,-Map=$(BUILD_DIR)/$(PROJECT).map
LDFLAGS += -Wl,--gc-sections
LDFLAGS += -specs=nosys.specs -specs=nano.specs

###############################################################################
# Build Rules
###############################################################################

all: $(BUILD_DIR)/$(PROJECT).elf size

# Link
$(BUILD_DIR)/$(PROJECT).elf: $(OBJECTS)
	$(LD) $(LDFLAGS) $^ -o $@
	$(OBJCOPY) -O binary $@ $(BUILD_DIR)/$(PROJECT).bin
	$(OBJCOPY) -O ihex $@ $(BUILD_DIR)/$(PROJECT).hex

# Compile C
$(BUILD_DIR)/%.o: $(SRC_DIR)/%.c | $(BUILD_DIR)
	$(CC) $(CFLAGS) -c $< -o $@

# Assemble
$(BUILD_DIR)/%.o: $(STARTUP_DIR)/%.s | $(BUILD_DIR)
	$(AS) $(ASFLAGS) -c $< -o $@

# Create build directory
$(BUILD_DIR):
	mkdir -p $(BUILD_DIR)

# Show memory usage
size: $(BUILD_DIR)/$(PROJECT).elf
	$(SIZE) $<

# Flash using OpenOCD
flash: $(BUILD_DIR)/$(PROJECT).elf
	openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
		-c "program $< verify reset exit"

# Clean
clean:
	rm -rf $(BUILD_DIR)

.PHONY: all clean flash size
```

---

## 15.4 CMake for STM32

```cmake
# CMakeLists.txt for STM32F401RE
cmake_minimum_required(VERSION 3.20)

# Cross-compilation toolchain
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)
set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)

project(firmware C ASM)

# MCU flags
set(MCU_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard")
set(CMAKE_C_FLAGS "${MCU_FLAGS} -Wall -Wextra -Os -g3 -ffunction-sections -fdata-sections")
set(CMAKE_ASM_FLAGS "${MCU_FLAGS} -g3")

# Linker flags
set(LINKER_SCRIPT "${CMAKE_SOURCE_DIR}/STM32F401RETX_FLASH.ld")
set(CMAKE_EXE_LINKER_FLAGS "${MCU_FLAGS} -T${LINKER_SCRIPT} -Wl,-Map=firmware.map -Wl,--gc-sections -specs=nosys.specs -specs=nano.specs")

# Source files
file(GLOB C_SOURCES "Src/*.c")
file(GLOB ASM_SOURCES "Startup/*.s")

# Build executable
add_executable(firmware.elf ${C_SOURCES} ${ASM_SOURCES})
target_include_directories(firmware.elf PRIVATE Inc)
target_compile_definitions(firmware.elf PRIVATE STM32F401xE)

# Generate .bin and .hex
add_custom_command(TARGET firmware.elf POST_BUILD
    COMMAND arm-none-eabi-objcopy -O binary firmware.elf firmware.bin
    COMMAND arm-none-eabi-objcopy -O ihex firmware.elf firmware.hex
    COMMAND arm-none-eabi-size firmware.elf)
```

Build with CMake:
```bash
mkdir build && cd build
cmake -G "Unix Makefiles" ..
make
```

---

## 15.5 Exercises

**Exercise 15.1:** Convert any previous chapter's project from STM32CubeIDE to a standalone Makefile build. Verify it produces the same binary.

**Exercise 15.2:** Add a `debug` target to the Makefile that builds with `-O0 -g3` and a `release` target with `-Os`.

**Exercise 15.3:** Create a CMakeLists.txt for a multi-file project with separate GPIO, UART, and timer drivers.

**Exercise 15.4:** Set up a simple CI script (bash) that builds the project and checks that the binary size doesn't exceed a threshold.

---

## 15.6 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 15 SUMMARY                        │
  │                                                            │
  │  ✓ Makefiles are the standard for firmware builds          │
  │  ✓ CMake provides cross-platform build generation          │
  │  ✓ --gc-sections removes unused code (saves Flash)         │
  │  ✓ -Os optimizes for size (typical for production)         │
  │  ✓ Linker scripts (.ld) define memory layout               │
  │  ✓ arm-none-eabi-size shows Flash/RAM usage                │
  │  ✓ Professional projects always use automated builds       │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 16 — Memory Maps & Linker Scripts →](./Chapter_16_Memory_Linker.md)*
