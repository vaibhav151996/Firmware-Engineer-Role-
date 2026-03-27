# Chapter 23: Other Platforms — AVR, ESP32, PIC, NXP

**Difficulty Level: ★★★☆☆ Intermediate**  
**Estimated Time: 1–2 weeks**

---

## 23.1 Concept Overview

### Why Learn Multiple Platforms?

Every MCU family has strengths. A well-rounded firmware engineer can pick the right chip for the job:

```
  Platform Selection Guide:
  
  ┌─────────────┬──────────────┬──────────────────────────────────┐
  │ Platform    │ Best For     │ Key Strength                     │
  ├─────────────┼──────────────┼──────────────────────────────────┤
  │ STM32       │ General      │ Huge ecosystem, wide range       │
  │ AVR (ATmega)│ Education    │ Simple, Arduino compatible       │
  │ ESP32       │ IoT / WiFi   │ Built-in WiFi + Bluetooth        │
  │ PIC         │ Cost-sensitive│ Cheap, proven, industrial        │
  │ NXP (LPC/i.MX)│ Automotive│ Safety, high performance         │
  │ TI MSP430   │ Ultra-low power│ Micro-amps in sleep            │
  │ Nordic nRF  │ BLE devices  │ Bluetooth Low Energy             │
  │ Renesas     │ Automotive   │ Safety-critical, automotive      │
  └─────────────┴──────────────┴──────────────────────────────────┘
```

---

## 23.2 AVR (ATmega328P — Arduino's MCU)

### Architecture Overview

```
  ATmega328P Specs:
  ┌─────────────────────────────────────┐
  │ Core:       8-bit AVR               │
  │ Clock:      16 MHz (external)       │
  │ Flash:      32 KB                   │
  │ SRAM:       2 KB                    │
  │ EEPROM:     1 KB                    │
  │ GPIO:       23 pins                 │
  │ ADC:        6-channel, 10-bit       │
  │ Timers:     3 (8-bit and 16-bit)    │
  │ UART:       1                       │
  │ SPI/I2C:    1 each                  │
  │ Voltage:    1.8V — 5.5V            │
  │ Package:    DIP-28 (breadboard!)    │
  └─────────────────────────────────────┘
  
  Key Differences from ARM:
  ✗ Harvard architecture (separate code & data buses)
  ✗ 8-bit registers (not 32-bit)
  ✗ No barrel shifter, no hardware multiply (some models)
  ✗ No NVIC — simpler interrupt model
  ✓ Simpler — great for learning
  ✓ DIP package — easy to prototype
  ✓ 5V tolerant — interfaces with legacy hardware
```

### AVR Bare-Metal Blink

```c
/* AVR bare-metal LED blink — ATmega328P */
#include <avr/io.h>
#include <util/delay.h>

/* Register access in AVR — it's simpler! */
/* DDRB = Data Direction Register for Port B (1 = output) */
/* PORTB = Output Register for Port B */
/* PB5 = Arduino pin 13 (built-in LED) */

int main(void)
{
    DDRB |= (1 << PB5);        /* PB5 = output */

    while (1)
    {
        PORTB |= (1 << PB5);   /* LED ON */
        _delay_ms(500);
        PORTB &= ~(1 << PB5);  /* LED OFF */
        _delay_ms(500);
    }

    return 0;
}
```

```bash
# Build for AVR
avr-gcc -mmcu=atmega328p -DF_CPU=16000000UL -Os -o blink.elf blink.c
avr-objcopy -O ihex blink.elf blink.hex
avrdude -c arduino -p m328p -P COM3 -U flash:w:blink.hex
```

### AVR UART

```c
#include <avr/io.h>

void uart_init(uint32_t baud)
{
    uint16_t ubrr = F_CPU / 16 / baud - 1;
    UBRR0H = (uint8_t)(ubrr >> 8);
    UBRR0L = (uint8_t)ubrr;
    UCSR0B = (1 << TXEN0) | (1 << RXEN0);  /* Enable TX and RX */
    UCSR0C = (3 << UCSZ00);                 /* 8-bit data, 1 stop */
}

void uart_send(char c)
{
    while (!(UCSR0A & (1 << UDRE0)));  /* Wait for empty buffer */
    UDR0 = c;
}

void uart_print(const char *s)
{
    while (*s) uart_send(*s++);
}
```

---

## 23.3 ESP32 (Espressif)

### Architecture Overview

```
  ESP32 Specs:
  ┌─────────────────────────────────────┐
  │ Core:       Dual Xtensa LX6, 32-bit│
  │ Clock:      Up to 240 MHz          │
  │ Flash:      4 MB (external SPI)    │
  │ SRAM:       520 KB                 │
  │ WiFi:       802.11 b/g/n           │
  │ Bluetooth:  v4.2 BR/EDR + BLE     │
  │ GPIO:       34 pins                │
  │ ADC:        18-channel, 12-bit     │
  │ DAC:        2-channel, 8-bit       │
  │ Timers:     4 (64-bit)            │
  │ UART:       3                      │
  │ SPI/I2C:    4 / 2                  │
  │ CAN:        1 (TWAI)              │
  │ Price:      ~$3-5                  │
  └─────────────────────────────────────┘
  
  Why ESP32 Is Special:
  ✓ Built-in WiFi + BLE — no external modules
  ✓ FreeRTOS included in SDK (ESP-IDF)
  ✓ OTA updates over WiFi
  ✓ Deep sleep: ~10 µA
  ✓ Huge community and libraries
  ✗ Not safety-certified (not for medical/automotive)
  ✗ Power-hungry when WiFi is active (~120 mA)
```

### ESP-IDF LED Blink

```c
/* ESP32 LED blink using ESP-IDF framework */
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#define LED_PIN  GPIO_NUM_2   /* Built-in LED on many ESP32 boards */

void app_main(void)
{
    /* Configure GPIO */
    gpio_config_t io_conf = {
        .pin_bit_mask = (1ULL << LED_PIN),
        .mode = GPIO_MODE_OUTPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_DISABLE,
    };
    gpio_config(&io_conf);

    while (1)
    {
        gpio_set_level(LED_PIN, 1);
        vTaskDelay(pdMS_TO_TICKS(500));
        gpio_set_level(LED_PIN, 0);
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

### ESP32 WiFi HTTP Server

```c
/* Minimal WiFi web server on ESP32 */
#include "esp_wifi.h"
#include "esp_http_server.h"
#include "esp_event.h"
#include "nvs_flash.h"

/* HTTP handler — serves sensor data as JSON */
esp_err_t sensor_handler(httpd_req_t *req)
{
    char response[128];
    int adc_val = 1234;  /* Replace with real ADC reading */
    float temp = adc_val * 0.1f;

    snprintf(response, sizeof(response),
             "{\"adc\": %d, \"temp\": %.1f}", adc_val, temp);

    httpd_resp_set_type(req, "application/json");
    return httpd_resp_send(req, response, strlen(response));
}

/* Start HTTP server */
void start_webserver(void)
{
    httpd_config_t config = HTTPD_DEFAULT_CONFIG();
    httpd_handle_t server = NULL;

    if (httpd_start(&server, &config) == ESP_OK)
    {
        httpd_uri_t uri = {
            .uri = "/sensor",
            .method = HTTP_GET,
            .handler = sensor_handler,
        };
        httpd_register_uri_handler(server, &uri);
    }
}
```

```bash
# Build with ESP-IDF
idf.py set-target esp32
idf.py build
idf.py -p COM3 flash monitor
```

---

## 23.4 PIC (Microchip PIC16/PIC18)

### Architecture Overview

```
  PIC18F4550 Specs:
  ┌─────────────────────────────────────┐
  │ Core:       8-bit PIC18            │
  │ Clock:      48 MHz (with PLL)      │
  │ Flash:      32 KB                  │
  │ SRAM:       2 KB                   │
  │ GPIO:       35 pins                │
  │ ADC:        13-channel, 10-bit     │
  │ USB:        Full-speed USB 2.0     │
  │ UART:       1 (EUSART)            │
  │ SPI/I2C:    1 each (MSSP)         │
  │ Price:      ~$2-4                  │
  └─────────────────────────────────────┘
  
  PIC Quirks:
  ✓ Very cheap — high-volume production favorite
  ✓ Extensive Microchip ecosystem (free MPLAB X IDE)
  ✓ Wide voltage range (2V-5.5V typically)
  ✗ Banking — memory divided into banks (legacy oddity)
  ✗ Limited stack depth (31 levels on PIC18)
  ✗ Non-standard C extensions (e.g., __interrupt)
  ✗ Harvard architecture with separate program/data memory
```

### PIC18 LED Blink (MPLAB XC8)

```c
/* PIC18F4550 LED blink — MPLAB XC8 compiler */
#include <xc.h>

/* Configuration bits */
#pragma config FOSC = INTOSC_NOCLKOUT  /* Internal oscillator */
#pragma config WDT  = OFF              /* Watchdog off */
#pragma config LVP  = OFF              /* Low voltage programming off */

#define _XTAL_FREQ 8000000  /* 8 MHz internal oscillator */

void main(void)
{
    OSCCON = 0x72;          /* 8 MHz internal oscillator */
    TRISBbits.TRISB0 = 0;  /* RB0 = output (LED) */

    while (1)
    {
        LATBbits.LATB0 = 1;   /* LED ON */
        __delay_ms(500);
        LATBbits.LATB0 = 0;   /* LED OFF */
        __delay_ms(500);
    }
}
```

---

## 23.5 NXP (LPC & i.MX RT)

### Architecture Overview

```
  NXP LPC1768 Specs:
  ┌─────────────────────────────────────┐
  │ Core:       ARM Cortex-M3          │
  │ Clock:      100 MHz                │
  │ Flash:      512 KB                 │
  │ SRAM:       64 KB                  │
  │ GPIO:       70 pins                │
  │ ADC:        8-channel, 12-bit      │
  │ DAC:        1-channel, 10-bit      │
  │ UART:       4                      │
  │ SPI/I2C:    2 / 3                  │
  │ CAN:        2                      │
  │ USB:        USB 2.0 Host/Device    │
  │ Ethernet:   10/100 Mbps            │
  └─────────────────────────────────────┘
  
  NXP i.MX RT1060 (High-Performance):
  ┌─────────────────────────────────────┐
  │ Core:       ARM Cortex-M7 @ 600 MHz│
  │ RAM:        1 MB on-chip           │
  │ Features:   LCD controller, camera │
  │             interface, hardware    │
  │             crypto, 2D graphics    │
  │ Used in:    HMI, IoT gateways,    │
  │             audio processing      │
  └─────────────────────────────────────┘
  
  NXP Strengths:
  ✓ Strong automotive lineup (S32K for AUTOSAR)
  ✓ MCUXpresso IDE + SDK (well-documented)
  ✓ Crossover processors (MCU speed with MPU features)
  ✓ Safety-certified variants (ISO 26262)
```

### NXP LPC1768 Bare-Metal Blink

```c
/* NXP LPC1768 bare-metal LED blink */
/* Note: Very similar to STM32 — both are ARM Cortex-M */

#include <stdint.h>

/* LPC1768 GPIO registers */
#define LPC_GPIO1_DIR  (*(volatile uint32_t *)0x2009C020UL)
#define LPC_GPIO1_SET  (*(volatile uint32_t *)0x2009C038UL)
#define LPC_GPIO1_CLR  (*(volatile uint32_t *)0x2009C03CUL)

/* LED on P1.18 (mbed LED1) */
#define LED_PIN 18

int main(void)
{
    LPC_GPIO1_DIR |= (1UL << LED_PIN);  /* P1.18 = output */

    while (1)
    {
        LPC_GPIO1_SET = (1UL << LED_PIN);  /* LED ON */
        for (volatile int i = 0; i < 1000000; i++);

        LPC_GPIO1_CLR = (1UL << LED_PIN);  /* LED OFF */
        for (volatile int i = 0; i < 1000000; i++);
    }
}
```

---

## 23.6 Platform Comparison Table

```
  ┌──────────┬─────────┬─────────┬────────┬─────────┬──────────┐
  │ Feature  │ STM32   │ AVR     │ ESP32  │ PIC     │ NXP LPC  │
  ├──────────┼─────────┼─────────┼────────┼─────────┼──────────┤
  │ Arch     │ ARM-M4  │ 8-bit   │ Xtensa │ 8-bit   │ ARM-M3   │
  │ Clock    │ 84 MHz  │ 16 MHz  │ 240MHz │ 48 MHz  │ 100 MHz  │
  │ Flash    │ 512 KB  │ 32 KB   │ 4 MB   │ 32 KB   │ 512 KB   │
  │ RAM      │ 96 KB   │ 2 KB    │ 520 KB │ 2 KB    │ 64 KB    │
  │ WiFi     │ No*     │ No      │ Yes    │ No      │ No*      │
  │ BLE      │ No*     │ No      │ Yes    │ No      │ No*      │
  │ USB      │ Yes     │ Some    │ Yes    │ Some    │ Yes      │
  │ CAN      │ Yes     │ No      │ Yes    │ Some    │ Yes      │
  │ Price    │ $3-10   │ $1-4    │ $3-5   │ $1-4    │ $3-10    │
  │ FPU      │ Yes     │ No      │ Yes    │ No      │ Some     │
  │ Debugger │ SWD     │ JTAG    │ JTAG   │ PICkit  │ SWD      │
  │ IDE      │ CubeIDE │ Atmel   │ ESP-IDF│ MPLAB X │ MCUXpress│
  │ RTOS     │ FreeRTOS│ Rare    │ Built-in│ Rare   │ FreeRTOS │
  │ Safety   │ Some    │ No      │ No     │ No      │ Yes      │
  └──────────┴─────────┴─────────┴────────┴─────────┴──────────┘
  * Some STM32/NXP variants include WiFi/BLE (STM32WB, i.MX RT)
```

---

## 23.7 Porting Between Platforms

### What Changes

```
  When porting firmware to a new MCU:
  
  MUST Change:
  ├── Register addresses and bit positions
  ├── Clock configuration
  ├── Linker script (memory map, Flash/RAM sizes)
  ├── Startup code (vector table, stack init)
  ├── Compiler flags and toolchain
  └── Pin assignments and peripheral mapping
  
  Should NOT Change (if designed well):
  ├── Application logic
  ├── Protocol state machines
  ├── Data structures and algorithms
  └── Business logic / control loops
  
  Design Pattern: Hardware Abstraction Layer (HAL)
  ┌──────────────────────┐
  │   Application Code   │  ← Platform-independent
  ├──────────────────────┤
  │   HAL Interface       │  ← gpio_write(), uart_send(), etc.
  ├──────────────────────┤
  │   Platform Driver     │  ← STM32/AVR/ESP32 specific code
  └──────────────────────┘
```

### Portable HAL Header

```c
/* hal_gpio.h — Platform-independent GPIO interface */
#pragma once

#include <stdint.h>
#include <stdbool.h>

typedef enum { GPIO_OUTPUT, GPIO_INPUT, GPIO_INPUT_PULLUP } gpio_mode_t;
typedef struct gpio_pin_s gpio_pin_t;  /* Opaque type */

/* Platform must implement these */
gpio_pin_t *gpio_create(uint8_t port, uint8_t pin, gpio_mode_t mode);
void gpio_write(gpio_pin_t *pin, bool value);
bool gpio_read(gpio_pin_t *pin);
void gpio_toggle(gpio_pin_t *pin);

/* Application code uses the same API regardless of platform */
```

---

## 23.8 Exercises

**Exercise 23.1:** If you have an Arduino (ATmega328P), port your LED blink to AVR bare-metal (no Arduino framework). Compare the binary size with Arduino's `digitalWrite()` version.

**Exercise 23.2:** If you have an ESP32, write a WiFi-connected sensor that reads an ADC value and serves it as JSON on a local HTTP endpoint. Access it from your browser.

**Exercise 23.3:** Design a HAL interface for UART that would work on STM32, AVR, and ESP32. Write the STM32 implementation you already have, and stub out the AVR version.

**Exercise 23.4:** Research one MCU from each manufacturer's latest lineup (STM32U5, ATtiny1616, ESP32-C6, PIC18-Q43, LPC55S69). Create a comparison table including: core, clock, Flash, RAM, special features, and price.

---

## 23.9 Chapter Summary

```
  Platform Knowledge Summary:
  
  ✓ ARM Cortex-M (STM32, NXP, Nordic) — industry standard
  ✓ AVR — great for learning, simple architecture
  ✓ ESP32 — IoT champion with WiFi/BLE built in
  ✓ PIC — cost-effective, legacy presence
  ✓ NXP — automotive, safety-critical applications
  
  Career Advice:
  - Master ONE platform deeply (STM32 is ideal)
  - Be familiar with 2-3 others
  - The concepts transfer — registers work the same way
  - What changes: addresses, bits, clock trees, toolchains
  - What stays: ISR design, RTOS patterns, protocol knowledge
```

---

*Next: [Chapter 24 — Capstone Project: Complete IoT Sensor Node →](./Chapter_24_Capstone.md)*
