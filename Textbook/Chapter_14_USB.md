# Chapter 14: USB Communication
## CDC Virtual COM Port and HID on STM32

**Difficulty Level: ★★★★☆ Advanced**  
**Estimated Time: 2 weeks**

---

## 14.1 Concept Overview

### What Is USB?

USB (Universal Serial Bus) is the most common connection standard. Unlike UART, SPI, and I2C, USB is an extremely complex protocol with multiple layers, descriptors, endpoints, and classes. STM32 microcontrollers have built-in **USB OTG** (On-The-Go) peripheral hardware.

```
  USB Communication Architecture:
  
  ┌──────────┐     USB Cable      ┌──────────────┐
  │  HOST    │  ┌──────────────┐  │   DEVICE     │
  │  (PC)    │──│ D+ / D-      │──│   (STM32)    │
  │          │  │ VBUS (5V)    │  │              │
  │  Driver  │  │ GND          │  │  USB OTG FS  │
  │  (CDC)   │  └──────────────┘  │  Peripheral  │
  └──────────┘                    └──────────────┘
  
  USB Device Classes commonly used in firmware:
  ┌──────────────────────────────────────────────┐
  │  CDC (Communication Device Class):           │
  │    Virtual COM Port — appears as serial port  │
  │    Replaces UART for PC communication         │
  │                                              │
  │  HID (Human Interface Device):               │
  │    Keyboard, mouse, custom controllers       │
  │    No driver installation needed!             │
  │                                              │
  │  MSC (Mass Storage Class):                   │
  │    USB flash drive emulation                 │
  │    Access SD card from PC                     │
  │                                              │
  │  DFU (Device Firmware Update):               │
  │    Firmware update over USB                  │
  └──────────────────────────────────────────────┘
```

### Why USB Is Complex

```
  USB Protocol Stack:
  
  ┌─────────────────────────┐
  │  Application Layer      │  Your firmware (send/receive data)
  ├─────────────────────────┤
  │  Class Driver (CDC/HID) │  Device class protocol
  ├─────────────────────────┤
  │  USB Core               │  Descriptors, endpoints, control
  ├─────────────────────────┤
  │  USB Hardware (OTG FS)  │  PHY, packet handling
  └─────────────────────────┘
  
  Unlike UART (write a byte → it sends), USB requires:
  - Descriptor tables (device, config, interface, endpoint)
  - Enumeration process (host queries device capabilities)
  - Endpoint management (control, bulk, interrupt, isochronous)
  - Protocol-specific handshakes
  
  For this reason, USB is typically implemented using a USB stack
  library rather than bare-metal register manipulation.
```

---

## 14.2 STM32 Hardware Internals

### USB OTG FS on STM32F401

```
  USB OTG FS Peripheral:
  
  Pins:
    PA11 = USB_DM (D-)
    PA12 = USB_DP (D+)
  
  On Nucleo boards: USB OTG is available on the CN10/Morpho headers
  (NOT on the ST-Link USB connector — that's UART!)
  
  Key features:
  - USB 2.0 Full Speed (12 Mbit/s)
  - Device, Host, or OTG mode
  - 4 IN endpoints + 4 OUT endpoints
  - Dedicated 1.25 KB FIFO RAM
  - Built-in pull-up on D+ for device mode
  
  Clock requirement: USB needs exactly 48 MHz clock
  → PLL must be configured with PLLQ divider for 48 MHz
```

---

## 14.3 Step-by-Step Implementation

### USB CDC Using STM32 USB Device Library

Due to USB's complexity, we'll use ST's USB Device library. Here's the approach:

```
  Project Setup for USB CDC:
  
  1. Create STM32CubeIDE project with CubeMX
  2. Enable USB_OTG_FS in Device mode
  3. Add USB_DEVICE middleware → CDC class
  4. Configure PLL for 48 MHz USB clock
  5. Generate code
  6. Implement CDC_Receive_FS() callback
  
  ┌────────────────────────────────────────┐
  │  CubeMX USB CDC Configuration:         │
  │                                        │
  │  Connectivity → USB_OTG_FS             │
  │    Mode: Device_Only                   │
  │                                        │
  │  Middleware → USB_DEVICE               │
  │    Class: CDC (Virtual Port Com)       │
  │                                        │
  │  Clock Configuration:                  │
  │    USB needs 48 MHz from PLL           │
  │    PLLQ must output 48 MHz             │
  └────────────────────────────────────────┘
```

### Key CDC Functions

```c
/******************************************************************************
 * How USB CDC works in your application:
 * 
 * Sending data to PC:
 *   CDC_Transmit_FS(buffer, length);
 *
 * Receiving data from PC:
 *   Implement the callback in usbd_cdc_if.c:
 *****************************************************************************/

/* In usbd_cdc_if.c — modify this function */
static int8_t CDC_Receive_FS(uint8_t *Buf, uint32_t *Len)
{
    /* Buf contains the received data from PC */
    /* *Len contains the number of bytes received */
    
    /* Example: Echo back to PC */
    CDC_Transmit_FS(Buf, *Len);
    
    /* Re-enable reception */
    USBD_CDC_SetRxBuffer(&hUsbDeviceFS, &Buf[0]);
    USBD_CDC_ReceivePacket(&hUsbDeviceFS);
    
    return USBD_OK;
}

/* In main.c — Sending data */
#include "usbd_cdc_if.h"

int main(void)
{
    /* ... initialization (generated by CubeMX) ... */
    
    char msg[] = "Hello from USB CDC!\r\n";
    
    while (1)
    {
        CDC_Transmit_FS((uint8_t *)msg, strlen(msg));
        HAL_Delay(1000);
    }
}
```

### Understanding USB Descriptors

```
  USB Device Descriptor (tells the host what this device is):
  
  Field                 Value          Meaning
  ──────────────────    ────────      ─────────────────────
  bDeviceClass          0x02          Communication Device
  idVendor              0x0483        STMicroelectronics
  idProduct             0x5740        Virtual COM Port
  bNumConfigurations    1             One configuration
  
  The host OS reads these descriptors during enumeration
  and loads the appropriate driver (CDC ACM driver).
  
  On Windows: appears as "STMicroelectronics Virtual COM Port"
  On Linux: appears as /dev/ttyACM0
  On macOS: appears as /dev/cu.usbmodem*
```

---

## 14.4 Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| USB clock not 48 MHz | Device not recognized | Configure PLL PLLQ for exactly 48 MHz |
| No USB cable on OTG pins | Nothing happens | USB OTG uses PA11/PA12, not ST-Link USB |
| Sending too fast | Data lost or CDC busy | Check CDC_Transmit_FS return for USBD_BUSY |
| Missing USB pull-up | Host doesn't detect device | Some boards need external pull-up on D+ |

---

## 14.5 Exercises

**Exercise 14.1:** Create a USB CDC project using CubeMX. Echo all received characters back to the PC.

**Exercise 14.2:** Send ADC readings over USB CDC at 10 Hz. Plot them using a Python script.

**Exercise 14.3:** Implement a USB HID keyboard that types "Hello" when the user button is pressed.

**Exercise 14.4:** Compare latency and throughput of USB CDC vs UART for data transfer.

---

## 14.6 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 14 SUMMARY                        │
  │                                                            │
  │  ✓ USB is complex — use ST's USB library for most tasks    │
  │  ✓ CDC class creates a Virtual COM Port (like UART)        │
  │  ✓ HID class for input devices (no driver needed)          │
  │  ✓ USB needs exactly 48 MHz clock from PLL                 │
  │  ✓ USB OTG uses PA11/PA12, not the ST-Link USB port       │
  │  ✓ Understanding descriptors helps with debugging          │
  │    enumeration issues                                      │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 15 — Build Systems: GCC, Make, CMake →](./Chapter_15_Build_Systems.md)*
