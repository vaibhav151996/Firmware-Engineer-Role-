# Chapter 22: Python for Firmware Engineering

**Difficulty Level: ★★★☆☆ Intermediate**  
**Estimated Time: 1–2 weeks**

---

## 22.1 Concept Overview

### Why Python for Firmware Engineers?

Python is not used **on** the microcontroller — it's used **around** it:

```
  Python in the Firmware Workflow:
  
  ┌──────────────────────────────────────────────────┐
  │              DEVELOPMENT HOST (PC)               │
  │                                                  │
  │  ┌─────────┐  ┌──────────┐  ┌─────────────────┐ │
  │  │ Serial  │  │  Test    │  │  Build / Flash  │ │
  │  │ Monitor │  │  Scripts │  │  Automation     │ │
  │  │ & Tools │  │  & HIL   │  │  (CI/CD)        │ │
  │  └────┬────┘  └────┬─────┘  └───────┬─────────┘ │
  │       │            │                │            │
  │       └────────────┼────────────────┘            │
  │                    │ USB / UART / SWD             │
  └────────────────────┼─────────────────────────────┘
                       │
                ┌──────┴──────┐
                │   STM32     │
                │ (Firmware)  │
                └─────────────┘
  
  Key Python use cases:
  ✓ Serial terminal with logging and parsing
  ✓ Automated hardware testing (HIL)
  ✓ Firmware upload scripts (bootloader client)
  ✓ Data visualization (real-time plotting)
  ✓ Code generation (register maps, config headers)
  ✓ Build system scripting and CI/CD
  ✓ Binary file manipulation (.bin, .hex, .elf)
```

### Required Python Packages

```bash
pip install pyserial              # Serial port communication
pip install matplotlib            # Data plotting
pip install numpy                 # Numerical computation
pip install intelhex              # Intel HEX file parsing
pip install pytest                # Testing framework
pip install pyyaml                # YAML config parsing
pip install struct                # (built-in) Binary data packing
```

---

## 22.2 Serial Communication (pyserial)

### Basic Serial Monitor

```python
#!/usr/bin/env python3
"""
serial_monitor.py — Terminal-style serial monitor with logging
"""

import serial
import sys
import time
from datetime import datetime

def serial_monitor(port, baud=115200, log_file=None):
    """Open serial port and display received data with timestamps."""
    
    ser = serial.Serial(port, baud, timeout=0.1)
    print(f"Connected to {port} at {baud} baud")
    print("Press Ctrl+C to exit\n")
    
    log = None
    if log_file:
        log = open(log_file, 'a')
        log.write(f"\n--- Session started {datetime.now()} ---\n")
    
    try:
        while True:
            # Read available data
            if ser.in_waiting > 0:
                data = ser.read(ser.in_waiting)
                text = data.decode('utf-8', errors='replace')
                
                # Add timestamp for each line
                timestamp = datetime.now().strftime('%H:%M:%S.%f')[:-3]
                for line in text.splitlines(keepends=True):
                    output = f"[{timestamp}] {line}"
                    sys.stdout.write(output)
                    if log:
                        log.write(output)
                        log.flush()
            
            # Read keyboard input and send to device
            # (In a full implementation, use threading or select)
            
    except KeyboardInterrupt:
        print("\nDisconnected")
    finally:
        ser.close()
        if log:
            log.close()

if __name__ == '__main__':
    port = sys.argv[1] if len(sys.argv) > 1 else 'COM3'
    serial_monitor(port, log_file='serial_log.txt')
```

### Command-Response Protocol

```python
#!/usr/bin/env python3
"""
device_commander.py — Send commands to firmware and parse responses
"""

import serial
import time

class DeviceCommander:
    """Communicate with STM32 firmware using a simple text protocol."""
    
    def __init__(self, port, baud=115200):
        self.ser = serial.Serial(port, baud, timeout=2.0)
        time.sleep(0.1)  # Wait for device to initialize
        self.ser.reset_input_buffer()
    
    def send_command(self, cmd):
        """Send command and return response lines."""
        self.ser.write(f"{cmd}\r\n".encode())
        time.sleep(0.05)
        
        response = []
        deadline = time.time() + 2.0  # 2 second timeout
        
        while time.time() < deadline:
            if self.ser.in_waiting > 0:
                line = self.ser.readline().decode().strip()
                if line:
                    response.append(line)
                    if line.startswith("OK") or line.startswith("ERROR"):
                        break
        
        return response
    
    def get_version(self):
        """Query firmware version."""
        resp = self.send_command("VERSION")
        return resp[0] if resp else "No response"
    
    def read_adc(self, channel=0):
        """Read ADC value from specified channel."""
        resp = self.send_command(f"ADC {channel}")
        for line in resp:
            if line.startswith("ADC="):
                return int(line.split('=')[1])
        return None
    
    def set_led(self, state):
        """Turn LED on or off."""
        cmd = "LED ON" if state else "LED OFF"
        resp = self.send_command(cmd)
        return "OK" in str(resp)
    
    def set_pwm(self, duty_percent):
        """Set PWM duty cycle (0-100)."""
        resp = self.send_command(f"PWM {duty_percent}")
        return "OK" in str(resp)
    
    def close(self):
        self.ser.close()

# Usage
if __name__ == '__main__':
    dev = DeviceCommander('COM3')
    
    print(f"Firmware: {dev.get_version()}")
    
    # Read ADC 10 times
    for i in range(10):
        adc = dev.read_adc(0)
        voltage = adc * 3.3 / 4095 if adc else 0
        print(f"ADC[0] = {adc} ({voltage:.3f} V)")
        time.sleep(0.5)
    
    dev.close()
```

---

## 22.3 Automated Testing (Hardware-in-the-Loop)

### Test Framework for Firmware

```python
#!/usr/bin/env python3
"""
test_firmware.py — Automated hardware-in-the-loop tests using pytest

Run with: pytest test_firmware.py -v
"""

import serial
import time
import pytest

# ── Fixture: shared serial connection ──
@pytest.fixture(scope="session")
def device():
    """Connect to device for the entire test session."""
    ser = serial.Serial('COM3', 115200, timeout=2.0)
    time.sleep(0.5)
    ser.reset_input_buffer()
    yield ser
    ser.close()

def send_and_receive(ser, cmd, timeout=2.0):
    """Helper: send command, return response lines."""
    ser.reset_input_buffer()
    ser.write(f"{cmd}\r\n".encode())
    
    response = []
    deadline = time.time() + timeout
    while time.time() < deadline:
        if ser.in_waiting:
            line = ser.readline().decode().strip()
            if line:
                response.append(line)
                if line.startswith("OK") or line.startswith("ERROR"):
                    break
    return response

# ── Test Cases ──

class TestBootup:
    """Verify device boots correctly."""
    
    def test_version_response(self, device):
        """Device should respond to VERSION command."""
        resp = send_and_receive(device, "VERSION")
        assert len(resp) > 0, "No response to VERSION"
        assert "v1." in resp[0], f"Unexpected version: {resp[0]}"
    
    def test_status_response(self, device):
        """Device should report OK status."""
        resp = send_and_receive(device, "STATUS")
        assert "OK" in str(resp), f"Bad status: {resp}"

class TestGPIO:
    """Test GPIO functionality."""
    
    def test_led_on(self, device):
        resp = send_and_receive(device, "LED ON")
        assert "OK" in str(resp)
    
    def test_led_off(self, device):
        resp = send_and_receive(device, "LED OFF")
        assert "OK" in str(resp)
    
    def test_led_invalid_command(self, device):
        resp = send_and_receive(device, "LED BLINK_FAST")
        assert "ERROR" in str(resp) or "UNKNOWN" in str(resp)

class TestADC:
    """Test ADC readings."""
    
    def test_adc_returns_valid_range(self, device):
        """ADC reading should be 0-4095."""
        resp = send_and_receive(device, "ADC 0")
        for line in resp:
            if line.startswith("ADC="):
                value = int(line.split('=')[1])
                assert 0 <= value <= 4095, f"ADC out of range: {value}"
                return
        pytest.fail("No ADC value in response")
    
    def test_adc_stability(self, device):
        """Multiple ADC readings should be within 5% of each other."""
        readings = []
        for _ in range(10):
            resp = send_and_receive(device, "ADC 0")
            for line in resp:
                if line.startswith("ADC="):
                    readings.append(int(line.split('=')[1]))
            time.sleep(0.1)
        
        assert len(readings) >= 8, "Too many failed ADC reads"
        avg = sum(readings) / len(readings)
        for r in readings:
            assert abs(r - avg) / max(avg, 1) < 0.05, \
                f"ADC unstable: {r} vs avg {avg:.0f}"

class TestUART:
    """Test UART echo and throughput."""
    
    def test_echo(self, device):
        """Device should echo back data in ECHO mode."""
        send_and_receive(device, "MODE ECHO")
        test_string = "Hello_STM32_Test_123"
        device.write(f"{test_string}\r\n".encode())
        time.sleep(0.1)
        
        response = device.read(device.in_waiting).decode()
        assert test_string in response, f"Echo failed: {response}"
    
    def test_uart_stress(self, device):
        """Send 1000 bytes rapidly, verify no data loss."""
        send_and_receive(device, "MODE ECHO")
        test_data = bytes(range(256)) * 4  # 1024 bytes
        
        device.write(test_data)
        time.sleep(1.0)
        
        received = device.read(device.in_waiting)
        assert len(received) >= len(test_data) * 0.99, \
            f"Data loss: sent {len(test_data)}, received {len(received)}"
```

---

## 22.4 Real-Time Data Plotting

### Live Plot of ADC Data

```python
#!/usr/bin/env python3
"""
live_plot.py — Real-time plot of sensor data from STM32
"""

import serial
import matplotlib.pyplot as plt
import matplotlib.animation as animation
from collections import deque
import re

# Configuration
PORT = 'COM3'
BAUD = 115200
WINDOW = 200  # Number of data points visible

# Data buffers
timestamps = deque(maxlen=WINDOW)
adc_values = deque(maxlen=WINDOW)
voltage_values = deque(maxlen=WINDOW)

# Setup serial
ser = serial.Serial(PORT, BAUD, timeout=0.1)

# Setup plot
fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(10, 6))
fig.suptitle('STM32 ADC Live Monitor')

line1, = ax1.plot([], [], 'b-', linewidth=1)
ax1.set_ylabel('ADC Raw (0-4095)')
ax1.set_ylim(0, 4095)
ax1.grid(True)

line2, = ax2.plot([], [], 'r-', linewidth=1)
ax2.set_ylabel('Voltage (V)')
ax2.set_xlabel('Sample')
ax2.set_ylim(0, 3.3)
ax2.grid(True)

def update(frame):
    """Called periodically by animation — read serial and update plot."""
    while ser.in_waiting > 0:
        try:
            line = ser.readline().decode().strip()
            # Expected format: "ADC=1234" or "V=1.234"
            match = re.match(r'ADC=(\d+)', line)
            if match:
                adc = int(match.group(1))
                voltage = adc * 3.3 / 4095.0
                
                adc_values.append(adc)
                voltage_values.append(voltage)
                timestamps.append(len(timestamps))
        except (UnicodeDecodeError, ValueError):
            pass
    
    if timestamps:
        t = list(timestamps)
        line1.set_data(t, list(adc_values))
        line2.set_data(t, list(voltage_values))
        
        for ax in (ax1, ax2):
            ax.set_xlim(max(0, t[-1] - WINDOW), t[-1] + 10)
    
    return line1, line2

ani = animation.FuncAnimation(fig, update, interval=50, blit=True, cache_frame_data=False)
plt.tight_layout()
plt.show()
ser.close()
```

---

## 22.5 Binary File Manipulation

### Working with .bin and .hex Files

```python
#!/usr/bin/env python3
"""
firmware_tools.py — Utilities for firmware binary manipulation
"""

import struct
import hashlib
from pathlib import Path

def analyze_binary(bin_path):
    """Analyze a firmware .bin file."""
    data = Path(bin_path).read_bytes()
    
    print(f"File: {bin_path}")
    print(f"Size: {len(data)} bytes ({len(data)/1024:.1f} KB)")
    print(f"MD5:  {hashlib.md5(data).hexdigest()}")
    print(f"SHA256: {hashlib.sha256(data).hexdigest()[:32]}...")
    
    # Parse vector table (first 16 entries)
    print("\nVector Table:")
    vectors = struct.unpack_from('<16I', data, 0)
    vector_names = [
        "Initial SP", "Reset Handler", "NMI Handler", "HardFault",
        "MemManage", "BusFault", "UsageFault", "Reserved",
        "Reserved", "Reserved", "Reserved", "SVCall",
        "Debug Monitor", "Reserved", "PendSV", "SysTick"
    ]
    for i, (val, name) in enumerate(zip(vectors, vector_names)):
        print(f"  [{i:2d}] 0x{val:08X}  {name}")
    
    # Validate stack pointer (should point to RAM)
    sp = vectors[0]
    if 0x20000000 <= sp <= 0x20020000:
        print(f"\n✓ Stack pointer valid: 0x{sp:08X}")
    else:
        print(f"\n✗ Stack pointer INVALID: 0x{sp:08X}")
    
    # Check for padding (find last non-0xFF byte)
    last_data = len(data)
    while last_data > 0 and data[last_data - 1] == 0xFF:
        last_data -= 1
    padding = len(data) - last_data
    if padding > 0:
        print(f"Padding: {padding} bytes of 0xFF at end")
        print(f"Actual code size: {last_data} bytes ({last_data/1024:.1f} KB)")

def add_crc_header(bin_path, output_path):
    """Add a CRC-32 header to firmware binary for bootloader verification."""
    data = Path(bin_path).read_bytes()
    
    # Calculate CRC-32
    import binascii
    crc = binascii.crc32(data) & 0xFFFFFFFF
    
    # Create header: [magic][size][crc32][reserved]
    MAGIC = 0xDEADBEEF
    header = struct.pack('<III4x', MAGIC, len(data), crc)  # 16 bytes
    
    output = header + data
    Path(output_path).write_bytes(output)
    
    print(f"Added CRC header: magic=0x{MAGIC:08X}, size={len(data)}, crc=0x{crc:08X}")
    print(f"Output: {output_path} ({len(output)} bytes)")

def bin_to_c_array(bin_path, array_name="firmware_image"):
    """Convert binary file to a C array (for embedding firmware in bootloader)."""
    data = Path(bin_path).read_bytes()
    
    lines = [f"/* Auto-generated from {bin_path} */"]
    lines.append(f"/* Size: {len(data)} bytes */")
    lines.append(f"const uint8_t {array_name}[] = {{")
    
    for i in range(0, len(data), 16):
        chunk = data[i:i+16]
        hex_vals = ', '.join(f'0x{b:02X}' for b in chunk)
        lines.append(f"    {hex_vals},")
    
    lines.append("};")
    lines.append(f"const uint32_t {array_name}_size = {len(data)};")
    
    return '\n'.join(lines)

# Usage
if __name__ == '__main__':
    import sys
    if len(sys.argv) > 1:
        analyze_binary(sys.argv[1])
```

---

## 22.6 Code Generation

### Generate Register Definitions from YAML

```python
#!/usr/bin/env python3
"""
codegen.py — Generate C header files from YAML register maps

Input YAML format:
  peripherals:
    GPIOA:
      base: 0x40020000
      registers:
        MODER:  { offset: 0x00, desc: "Mode register" }
        OTYPER: { offset: 0x04, desc: "Output type" }
        ...
"""

import yaml

TEMPLATE_HEADER = """\
/******************************************************************************
 * AUTO-GENERATED — DO NOT EDIT
 * Generated from: {source}
 *****************************************************************************/
#pragma once

#include <stdint.h>

"""

def generate_header(yaml_path, output_path):
    """Generate C header from YAML register map."""
    with open(yaml_path) as f:
        config = yaml.safe_load(f)
    
    lines = [TEMPLATE_HEADER.format(source=yaml_path)]
    
    for periph_name, periph in config['peripherals'].items():
        base = periph['base']
        lines.append(f"/* ── {periph_name} (Base: 0x{base:08X}) ── */")
        
        for reg_name, reg in periph['registers'].items():
            offset = reg['offset']
            addr = base + offset
            desc = reg.get('desc', '')
            full_name = f"{periph_name}_{reg_name}"
            lines.append(
                f"#define {full_name:<24s} "
                f"(*(volatile uint32_t *)0x{addr:08X}UL)  "
                f"/* {desc} */"
            )
        lines.append("")
    
    with open(output_path, 'w') as f:
        f.write('\n'.join(lines))
    
    print(f"Generated: {output_path}")

# Example YAML (registers.yaml):
EXAMPLE_YAML = """
peripherals:
  GPIOA:
    base: 0x40020000
    registers:
      MODER:   { offset: 0x00, desc: "Port mode register" }
      OTYPER:  { offset: 0x04, desc: "Output type register" }
      OSPEEDR: { offset: 0x08, desc: "Output speed register" }
      PUPDR:   { offset: 0x0C, desc: "Pull-up/pull-down register" }
      IDR:     { offset: 0x10, desc: "Input data register" }
      ODR:     { offset: 0x14, desc: "Output data register" }
      BSRR:    { offset: 0x18, desc: "Bit set/reset register" }

  USART2:
    base: 0x40004400
    registers:
      SR:   { offset: 0x00, desc: "Status register" }
      DR:   { offset: 0x04, desc: "Data register" }
      BRR:  { offset: 0x08, desc: "Baud rate register" }
      CR1:  { offset: 0x0C, desc: "Control register 1" }
      CR2:  { offset: 0x10, desc: "Control register 2" }
      CR3:  { offset: 0x14, desc: "Control register 3" }
"""
```

---

## 22.7 Build Automation

### Build + Flash Script

```python
#!/usr/bin/env python3
"""
build_and_flash.py — Automate build, size check, and flash process
"""

import subprocess
import sys
import re
from pathlib import Path

PROJECT_DIR = Path(".")
BUILD_DIR = PROJECT_DIR / "build"
ELF_FILE = BUILD_DIR / "firmware.elf"
BIN_FILE = BUILD_DIR / "firmware.bin"

FLASH_SIZE_KB = 512
RAM_SIZE_KB = 96

def run(cmd, **kwargs):
    """Run a command and return output."""
    print(f"  $ {cmd}")
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True, **kwargs)
    if result.returncode != 0:
        print(f"ERROR: {result.stderr}")
        sys.exit(1)
    return result.stdout

def build():
    """Build the project."""
    print("\n=== Building ===")
    BUILD_DIR.mkdir(exist_ok=True)
    run("make -j$(nproc) all", cwd=PROJECT_DIR)
    print("Build OK")

def check_size():
    """Check that firmware fits in Flash and RAM."""
    print("\n=== Size Check ===")
    output = run(f"arm-none-eabi-size {ELF_FILE}")
    
    # Parse size output
    # Format:  text   data    bss    dec    hex  filename
    lines = output.strip().split('\n')
    parts = lines[1].split()
    text = int(parts[0])
    data = int(parts[1])
    bss  = int(parts[2])
    
    flash_used = text + data
    ram_used = data + bss
    flash_pct = flash_used / (FLASH_SIZE_KB * 1024) * 100
    ram_pct = ram_used / (RAM_SIZE_KB * 1024) * 100
    
    print(f"  Flash: {flash_used:,} bytes ({flash_pct:.1f}% of {FLASH_SIZE_KB} KB)")
    print(f"  RAM:   {ram_used:,} bytes ({ram_pct:.1f}% of {RAM_SIZE_KB} KB)")
    
    if flash_pct > 90:
        print("  ⚠ WARNING: Flash usage above 90%!")
    if ram_pct > 80:
        print("  ⚠ WARNING: RAM usage above 80%!")
    
    return flash_pct < 100 and ram_pct < 100

def flash():
    """Flash firmware via OpenOCD."""
    print("\n=== Flashing ===")
    run(f"openocd -f interface/stlink.cfg -f target/stm32f4x.cfg "
        f"-c 'program {ELF_FILE} verify reset exit'")
    print("Flash OK — device running!")

if __name__ == '__main__':
    build()
    if check_size():
        if '--flash' in sys.argv:
            flash()
        else:
            print("\nUse --flash to program the device")
    else:
        print("\nERROR: Firmware exceeds memory limits!")
        sys.exit(1)
```

---

## 22.8 Exercises

**Exercise 22.1:** Write a Python script that connects to your STM32 UART, sends "ADC 0" every 100ms, parses the response, and saves data to a CSV file with timestamps.

**Exercise 22.2:** Create a real-time plot showing both raw ADC values and a moving average (window of 10 samples).

**Exercise 22.3:** Write a `pytest` test suite for your bootloader: send 'U', wait for READY, send a small test binary, verify DONE response.

**Exercise 22.4:** Create a code generator that reads your STM32's SVD file (XML register description) and generates a complete register header file.

---

## 22.9 Chapter Summary

```
  Python Toolkit for Firmware Engineers:
  
  ✓ pyserial      → Talk to your device (monitor, commands, upload)
  ✓ pytest         → Automated hardware-in-the-loop testing
  ✓ matplotlib     → Real-time data visualization
  ✓ struct/pathlib → Binary file manipulation
  ✓ yaml + codegen → Generate C headers from descriptions
  ✓ subprocess     → Build and flash automation
  
  Golden Rule: Python runs on the HOST, not the TARGET.
  It's your most powerful tool for testing and automation.
```

---

*Next: [Chapter 23 — Other Platforms: AVR, ESP32, PIC, NXP →](./Chapter_23_Other_Platforms.md)*
