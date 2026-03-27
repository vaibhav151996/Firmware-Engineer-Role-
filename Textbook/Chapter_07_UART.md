# Chapter 7: UART Communication
## ★ MILESTONE: Building a UART Driver from the Datasheet

**Difficulty Level: ★★★☆☆ Intermediate**
**Estimated Time: 2–3 weeks**

---

## 7.1 Concept Overview

### What Is UART?

UART (Universal Asynchronous Receiver/Transmitter) is one of the most fundamental communication protocols in embedded systems. It allows two devices to exchange data over just **two wires**: TX (transmit) and RX (receive).

```
  UART Connection:
  
  ┌──────────┐        ┌──────────┐
  │ Device A │        │ Device B │
  │          │  TX────→RX        │
  │          │        │          │
  │          │  RX←────TX        │
  │          │        │          │
  │    GND───┼────────┼──GND    │
  └──────────┘        └──────────┘
  
  On the Nucleo board:
  
  ┌──────────┐  USB   ┌──────────┐  SWD    ┌──────────┐
  │    PC    │←──────→│ ST-Link  │←───────→│ STM32    │
  │          │        │          │         │          │
  │ Terminal │  VCP   │ USART2   │         │ PA2(TX)  │
  │ (PuTTY)  │←──────→│ bridge   │←───────→│ PA3(RX)  │
  └──────────┘        └──────────┘         └──────────┘
  
  VCP = Virtual COM Port
  The ST-Link on the Nucleo board bridges USART2 to USB,
  so your PC sees it as a COM port.
```

### UART Frame Format

```
  UART Byte Transmission (8N1 — 8 data bits, No parity, 1 stop bit):
  
  Idle ─────┐   ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌───┐ ─────── Idle
  (HIGH)     │   │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │   │ (HIGH)
             └───┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘   │
             Start D0  D1  D2  D3  D4  D5  D6  D7 Stop
             Bit  (LSB)                         (MSB) Bit
  
  Timing (at 9600 baud):
  |←───────────────── ~1.04 ms total ──────────────────→|
  |← 104 µs →|  ← Each bit is exactly 1/9600 seconds →
  
  Example: Sending ASCII 'A' = 0x41 = 0100 0001
  
  Idle ─┐   ┌───┐   ┌───┐                       ┌───┐ ┌─── Idle
        │   │   │   │   │                       │   │ │
        └───┘   └───┘   └───────────────────────┘   └─┘
        Start  D0=1  D1=0  D2=0 D3=0 D4=0 D5=0  D6=1  D7=0  Stop
               (LSB first: 1,0,0,0,0,0,1,0 = 0x41)
```

### Why Firmware Engineers Need UART

- **Debugging:** `printf`-style debug output via serial terminal
- **Device-to-device communication:** Sensor modules, GPS, Bluetooth (HC-06)
- **Console/CLI:** Command-line interface for testing and configuration
- **Logging:** Record events, errors, and performance data
- **Bootloaders:** Update firmware via serial port

---

## 7.2 STM32 Hardware Internals

### USART2 on STM32F401

On the Nucleo board, USART2 is connected to the ST-Link VCP:
- **PA2** → USART2_TX (AF7)
- **PA3** → USART2_RX (AF7)

```
  USART Block Diagram (Simplified):
  
  APB1 Clock (42 MHz after PLL)
       │
       ↓
  ┌────────────────────────────────────────────┐
  │                 USART2                      │
  │                                            │
  │  ┌──────────────┐    ┌──────────────────┐  │
  │  │ Baud Rate    │    │   TX Shift       │  │
  │  │ Generator    │───→│   Register       │──────→ TX Pin (PA2)
  │  │ (BRR)        │    │   (SR → DR)      │  │
  │  └──────────────┘    └──────────────────┘  │
  │         │                                  │
  │         │            ┌──────────────────┐  │
  │         │            │   RX Shift       │  │
  │         └───────────→│   Register       │←──── RX Pin (PA3)
  │                      │   (SR ← DR)      │  │
  │                      └──────────────────┘  │
  │                                            │
  │  Key Registers:                            │
  │    SR   — Status Register (flags)          │
  │    DR   — Data Register (read/write data)  │
  │    BRR  — Baud Rate Register               │
  │    CR1  — Control Register 1 (enable, IE)  │
  │    CR2  — Control Register 2 (stop bits)   │
  │    CR3  — Control Register 3 (DMA, flow)   │
  └────────────────────────────────────────────┘
```

### USART Registers Detail

```
  USART2 Base Address: 0x40004400

  Offset  Register  Key Bits
  ──────  ────────  ───────────────────────────────────────────
  0x00    SR        Bit 7 (TXE): TX data register empty
                    Bit 6 (TC):  Transmission complete
                    Bit 5 (RXNE): RX data register not empty
                    Bit 3 (ORE): Overrun error
  
  0x04    DR        Bits [8:0]: Data (read = RX, write = TX)
  
  0x08    BRR       Bits [15:4]: Mantissa (integer part)
                    Bits [3:0]:  Fraction (fractional part)
  
  0x0C    CR1       Bit 13 (UE):   USART enable
                    Bit 12 (M):    Word length (0=8bit, 1=9bit)
                    Bit 5 (RXNEIE): RX interrupt enable
                    Bit 3 (TE):    Transmitter enable
                    Bit 2 (RE):    Receiver enable
  
  0x10    CR2       Bits [13:12] (STOP): Stop bits
                                         00=1, 10=2, 01=0.5, 11=1.5
```

### Baud Rate Calculation

The baud rate is set via the BRR (Baud Rate Register):

```
  Formula:
  USARTDIV = f_CK / (16 × desired_baud)
  
  Example: 9600 baud with APB1 clock = 16 MHz (HSI, no PLL)
  
  USARTDIV = 16,000,000 / (16 × 9600)
           = 16,000,000 / 153,600
           = 104.1667
  
  BRR encoding:
    Mantissa (integer part) = 104 = 0x68
    Fraction = 0.1667 × 16 = 2.667 ≈ 3 = 0x3
    
    BRR = (Mantissa << 4) | Fraction
        = (0x68 << 4) | 0x3
        = 0x683
  
  ─────────────────────────────────────────────────
  
  Example: 115200 baud with APB1 clock = 42 MHz (84 MHz SYSCLK, /2)
  
  USARTDIV = 42,000,000 / (16 × 115,200)
           = 42,000,000 / 1,843,200
           = 22.7865
  
  BRR encoding:
    Mantissa = 22 = 0x16
    Fraction = 0.7865 × 16 = 12.58 ≈ 13 = 0xD
    
    BRR = (0x16 << 4) | 0xD
        = 0x16D
```

---

## 7.3 Step-by-Step Implementation

### Complete UART Driver

```c
/******************************************************************************
 * File:    uart_driver.h
 * Brief:   UART driver for STM32F401 — register-level
 *          Supports polling and interrupt-based TX/RX
 *****************************************************************************/
#ifndef UART_DRIVER_H
#define UART_DRIVER_H

#include <stdint.h>

/* USART register structure */
typedef struct {
    volatile uint32_t SR;    /* 0x00: Status register */
    volatile uint32_t DR;    /* 0x04: Data register */
    volatile uint32_t BRR;   /* 0x08: Baud rate register */
    volatile uint32_t CR1;   /* 0x0C: Control register 1 */
    volatile uint32_t CR2;   /* 0x10: Control register 2 */
    volatile uint32_t CR3;   /* 0x14: Control register 3 */
    volatile uint32_t GTPR;  /* 0x18: Guard time/prescaler */
} USART_TypeDef;

#define USART1  ((USART_TypeDef *)0x40011000UL)
#define USART2  ((USART_TypeDef *)0x40004400UL)  /* ST-Link VCP */
#define USART6  ((USART_TypeDef *)0x40011400UL)

/* SR bits */
#define USART_SR_TXE    (1UL << 7)   /* TX data register empty */
#define USART_SR_TC     (1UL << 6)   /* Transmission complete */
#define USART_SR_RXNE   (1UL << 5)   /* RX data register not empty */
#define USART_SR_ORE    (1UL << 3)   /* Overrun error */

/* CR1 bits */
#define USART_CR1_UE    (1UL << 13)  /* USART enable */
#define USART_CR1_TE    (1UL << 3)   /* Transmitter enable */
#define USART_CR1_RE    (1UL << 2)   /* Receiver enable */
#define USART_CR1_RXNEIE (1UL << 5)  /* RXNE interrupt enable */
#define USART_CR1_TXEIE  (1UL << 7)  /* TXE interrupt enable */

/* Function prototypes */
void uart_init(USART_TypeDef *uart, uint32_t pclk, uint32_t baud);
void uart_send_char(USART_TypeDef *uart, char c);
void uart_send_string(USART_TypeDef *uart, const char *str);
char uart_receive_char(USART_TypeDef *uart);
int  uart_receive_char_nonblocking(USART_TypeDef *uart, char *c);
void uart_send_number(USART_TypeDef *uart, int32_t num);

#endif /* UART_DRIVER_H */
```

```c
/******************************************************************************
 * File:    uart_driver.c
 * Brief:   UART driver implementation — register-level, polling mode
 *****************************************************************************/

#include "uart_driver.h"

/* RCC */
#define RCC_BASE        0x40023800UL
#define RCC_AHB1ENR     (*(volatile uint32_t *)(RCC_BASE + 0x30UL))
#define RCC_APB1ENR     (*(volatile uint32_t *)(RCC_BASE + 0x40UL))

/* GPIO */
#define GPIOA_BASE      0x40020000UL
#define GPIOA_MODER     (*(volatile uint32_t *)(GPIOA_BASE + 0x00UL))
#define GPIOA_AFRL      (*(volatile uint32_t *)(GPIOA_BASE + 0x20UL))

/*
 * uart_init() — Initialize USART2 for TX/RX at specified baud rate
 *
 * Steps:
 * 1. Enable clocks (GPIO and USART)
 * 2. Configure GPIO pins for alternate function (AF7)
 * 3. Set baud rate
 * 4. Enable transmitter and receiver
 * 5. Enable USART
 */
void uart_init(USART_TypeDef *uart, uint32_t pclk, uint32_t baud)
{
    uint32_t usartdiv;
    uint32_t mantissa;
    uint32_t fraction;

    if (uart == USART2)
    {
        /* Enable GPIOA clock (PA2=TX, PA3=RX) */
        RCC_AHB1ENR |= (1UL << 0);

        /* Enable USART2 clock (on APB1 bus) */
        RCC_APB1ENR |= (1UL << 17);

        /* Configure PA2 as Alternate Function (AF7 = USART2_TX) */
        GPIOA_MODER &= ~(3UL << 4);     /* Clear PA2 mode */
        GPIOA_MODER |=  (2UL << 4);     /* Set to AF mode */

        /* Configure PA3 as Alternate Function (AF7 = USART2_RX) */
        GPIOA_MODER &= ~(3UL << 6);     /* Clear PA3 mode */
        GPIOA_MODER |=  (2UL << 6);     /* Set to AF mode */

        /* Set AF7 for PA2 and PA3 (AFRL register, 4 bits per pin) */
        GPIOA_AFRL &= ~(0xFUL << 8);    /* Clear PA2 AF */
        GPIOA_AFRL |=  (7UL << 8);      /* Set AF7 for PA2 */
        GPIOA_AFRL &= ~(0xFUL << 12);   /* Clear PA3 AF */
        GPIOA_AFRL |=  (7UL << 12);     /* Set AF7 for PA3 */
    }

    /*
     * Calculate baud rate:
     * USARTDIV = pclk / (16 * baud)
     * BRR = (mantissa << 4) | fraction
     *
     * We use fixed-point math to avoid floating point:
     * USARTDIV × 16 = pclk / baud
     * mantissa = (pclk / baud) / 16
     * fraction = (pclk / baud) % 16
     */
    usartdiv = pclk / baud;             /* This gives USARTDIV × 16 */
    mantissa = usartdiv / 16UL;
    fraction = usartdiv % 16UL;

    uart->BRR = (mantissa << 4) | fraction;

    /*
     * Configure USART:
     * - 8 data bits (M=0, default)
     * - 1 stop bit (CR2 STOP=00, default)
     * - No parity (PCE=0, default)
     * - Enable TX and RX
     * - Enable USART
     */
    uart->CR1 = USART_CR1_TE | USART_CR1_RE | USART_CR1_UE;
}

/*
 * uart_send_char() — Send a single character (blocking)
 *
 * Wait for TXE (TX data register Empty) flag, then write data.
 * The hardware shifts out the byte automatically.
 */
void uart_send_char(USART_TypeDef *uart, char c)
{
    /* Wait until TX data register is empty */
    while (!(uart->SR & USART_SR_TXE))
    {
        /* TXE=0 means previous byte is still being sent */
    }

    /* Write the character to the data register */
    uart->DR = (uint32_t)c;
}

/*
 * uart_send_string() — Send a null-terminated string
 */
void uart_send_string(USART_TypeDef *uart, const char *str)
{
    while (*str != '\0')
    {
        uart_send_char(uart, *str);
        str++;
    }
}

/*
 * uart_receive_char() — Receive a single character (blocking)
 *
 * Wait for RXNE (RX data register Not Empty) flag, then read data.
 */
char uart_receive_char(USART_TypeDef *uart)
{
    /* Wait until RX data register has data */
    while (!(uart->SR & USART_SR_RXNE))
    {
        /* RXNE=0 means no data has been received yet */
    }

    /* Read and return the received character */
    return (char)(uart->DR & 0xFFUL);
}

/*
 * uart_receive_char_nonblocking() — Try to receive without blocking
 *
 * Returns 1 if a character was available, 0 if not.
 * The character is stored in *c if available.
 */
int uart_receive_char_nonblocking(USART_TypeDef *uart, char *c)
{
    int result;

    if (uart->SR & USART_SR_RXNE)
    {
        *c = (char)(uart->DR & 0xFFUL);
        result = 1;
    }
    else
    {
        result = 0;
    }

    return result;
}

/*
 * uart_send_number() — Send a signed integer as ASCII text
 */
void uart_send_number(USART_TypeDef *uart, int32_t num)
{
    char buf[12];   /* Enough for -2147483648\0 */
    int i = 0;
    uint32_t unum;

    if (num < 0)
    {
        uart_send_char(uart, '-');
        unum = (uint32_t)(-num);
    }
    else
    {
        unum = (uint32_t)num;
    }

    /* Convert to ASCII digits (reverse order) */
    do
    {
        buf[i++] = '0' + (char)(unum % 10UL);
        unum /= 10UL;
    } while (unum > 0UL);

    /* Send digits in correct order */
    while (i > 0)
    {
        i--;
        uart_send_char(uart, buf[i]);
    }
}
```

### Using the UART Driver

```c
/******************************************************************************
 * File:    main.c — UART echo example
 * Board:   STM32 Nucleo-F401RE
 * 
 * Connect a serial terminal (PuTTY/Tera Term) to the Nucleo's
 * Virtual COM Port at 115200 baud, 8N1.
 *****************************************************************************/

#include <stdint.h>
#include "uart_driver.h"

int main(void)
{
    /* Initialize UART2 at 115200 baud
     * APB1 clock = 16 MHz (default HSI, no PLL) */
    uart_init(USART2, 16000000UL, 115200UL);

    /* Send a welcome message */
    uart_send_string(USART2, "\r\n=== STM32 UART Driver ===\r\n");
    uart_send_string(USART2, "Type characters — they will be echoed back.\r\n");

    /* Echo loop: receive a character and send it back */
    while (1)
    {
        char c = uart_receive_char(USART2);  /* Wait for input */
        
        uart_send_string(USART2, "Received: ");
        uart_send_char(USART2, c);
        uart_send_string(USART2, "\r\n");
    }

    return 0;
}
```

---

## 7.4 Interrupt-Driven UART

### RX with Interrupt and Ring Buffer

```c
/******************************************************************************
 * File:    uart_interrupt.c
 * Brief:   Interrupt-driven UART RX with ring buffer
 *****************************************************************************/

#define RX_BUFFER_SIZE  128

static volatile uint8_t rx_buffer[RX_BUFFER_SIZE];
static volatile uint16_t rx_head = 0;  /* ISR writes here */
static volatile uint16_t rx_tail = 0;  /* Main loop reads here */

/*
 * Enable UART RX interrupt
 */
void uart_enable_rx_interrupt(USART_TypeDef *uart)
{
    uart->CR1 |= USART_CR1_RXNEIE;   /* Enable RXNE interrupt */

    /* Enable USART2 IRQ in NVIC (IRQ #38) */
    /* ISER1, bit 6 (38 - 32 = 6) */
    (*(volatile uint32_t *)(0xE000E104UL)) = (1UL << 6);
}

/*
 * USART2 ISR — Called when a byte is received
 */
void USART2_IRQHandler(void)
{
    if (USART2->SR & USART_SR_RXNE)
    {
        uint8_t data = (uint8_t)(USART2->DR & 0xFFUL);
        uint16_t next = (rx_head + 1U) % RX_BUFFER_SIZE;

        if (next != rx_tail)   /* Buffer not full */
        {
            rx_buffer[rx_head] = data;
            rx_head = next;
        }
        /* If buffer is full, byte is dropped (overrun) */
    }
}

/*
 * Check if data is available in the buffer
 */
uint16_t uart_rx_available(void)
{
    return (rx_head - rx_tail + RX_BUFFER_SIZE) % RX_BUFFER_SIZE;
}

/*
 * Read a byte from the buffer (call only if available)
 */
uint8_t uart_rx_read(void)
{
    uint8_t data = rx_buffer[rx_tail];
    rx_tail = (rx_tail + 1U) % RX_BUFFER_SIZE;
    return data;
}
```

---

## 7.5 Diagrams

### UART Data Flow

```
  Transmit (TX) Flow:
  
  Your code              USART Hardware              Pin
  ──────────              ──────────────             ────
  uart_send_char()
       │
       ↓
  Wait for TXE flag ←── SR.TXE set when DR is empty
       │
       ↓
  Write byte to DR ───→ DR loaded into shift register
                         │
                         ↓
                    Shift register sends bits
                    one by one at baud rate ───────→ TX Pin
                         │
                    SR.TC set when complete
  
  
  Receive (RX) Flow:
  
  RX Pin ──────→ Shift register samples bits
                  at baud rate
                         │
                         ↓
                  Byte assembled in shift register
                         │
                         ↓
                  DR loaded from shift register
                  SR.RXNE set ─────→ Your code reads DR
                                     (or ISR fires if RXNEIE=1)
```

---

## 7.6 Common Mistakes & Debugging

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Wrong baud rate | Garbage characters on terminal | Verify pclk and BRR calculation |
| TX and RX swapped | No communication | TX→RX, RX→TX (crossover) |
| Wrong COM port in terminal | Nothing appears | Check Device Manager for port number |
| Not enabling USART clock | No output | Set bit 17 in RCC_APB1ENR |
| Wrong AF number | Pin doesn't output UART signal | PA2/PA3 need AF7, not another AF |
| Missing `\r\n` | Text on same line | Use `\r\n` for carriage return + newline |
| Terminal baud mismatch | Random characters | Both sides must use same baud rate |

### Debugging UART with Logic Analyzer

Connect a logic analyzer to PA2 (TX) to see actual signal:
1. Configure the analyzer for async serial protocol
2. Set baud rate to match your code
3. Send a known character (e.g., 'U' = 0x55 gives alternating bits)
4. Verify timing matches expected baud rate

---

## 7.7 Exercises

**Exercise 7.1:** Implement a simple command processor: when the user types "LED ON" + Enter, turn on LD2; when they type "LED OFF" + Enter, turn it off.

**Exercise 7.2:** Write a `printf`-like function: `uart_printf(USART_TypeDef *uart, const char *fmt, ...)` that supports `%d`, `%x`, `%s`, and `%c` format specifiers. (This is a common interview question!)

**Exercise 7.3:** Calculate and program the BRR for 9600 baud when the APB1 clock is 42 MHz (84 MHz SYSCLK with /2 prescaler).

**Exercise 7.4:** Implement the interrupt-driven UART RX from Section 7.4 and build a terminal echo application that can handle rapid typing without losing characters.

**Exercise 7.5:** Add error handling: detect and report framing errors, overrun errors, and noise errors by checking the appropriate SR bits.

---

## 7.8 Industry & Career Notes

- Writing a UART driver from scratch is a **classic firmware interview question** — "Can you configure UART on an STM32 without HAL?"
- Production UART code always uses **DMA** (Chapter 12) for high-throughput data to avoid CPU overhead
- **Log levels** (DEBUG, INFO, WARN, ERROR) are standard practice for UART debug output
- Automotive protocols like **LIN** (Local Interconnect Network) are based on UART

---

## 7.9 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │            ★ MILESTONE: UART DRIVER COMPLETE ★             │
  │                                                            │
  │  ✓ UART sends/receives data over 2 wires (TX, RX)         │
  │                                                            │
  │  ✓ Frame format: Start bit, 8 data bits, Stop bit (8N1)   │
  │                                                            │
  │  ✓ Baud rate set via BRR = f_CK / (16 × baud)            │
  │                                                            │
  │  ✓ Must configure: GPIO (AF7), Clock (RCC), USART regs    │
  │                                                            │
  │  ✓ Polling: wait for TXE/RXNE flags before read/write     │
  │                                                            │
  │  ✓ Interrupt: RXNEIE enables ISR on received data          │
  │    Use ring buffer to store bytes without blocking         │
  │                                                            │
  │  ✓ On Nucleo, USART2 (PA2/PA3) connects to ST-Link VCP    │
  │                                                            │
  │  DRIVER FUNCTIONS BUILT:                                   │
  │    uart_init()           uart_send_char()                  │
  │    uart_send_string()    uart_receive_char()               │
  │    uart_send_number()    uart_rx_available()                │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 8 — Timers & PWM →](./Chapter_08_Timers_PWM.md)*
