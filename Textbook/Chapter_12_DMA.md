# Chapter 12: DMA — Direct Memory Access
## Offloading Data Transfers from the CPU

**Difficulty Level: ★★★☆☆ Intermediate**  
**Estimated Time: 1–2 weeks**

---

## 12.1 Concept Overview

### What Is DMA?

DMA (Direct Memory Access) is a hardware engine that can **transfer data between memory and peripherals without CPU intervention**. Instead of the CPU reading from a peripheral register and writing to a buffer byte-by-byte, DMA does it automatically in the background.

```
  WITHOUT DMA (CPU copies data):        WITH DMA (hardware copies data):
  
  ┌──────┐ Read  ┌──────────┐           ┌──────┐         ┌──────────┐
  │ CPU  │←──────│ UART DR  │           │ CPU  │         │ UART DR  │
  │      │──────→│          │           │(FREE)│         │          │
  │      │ Write ┌──────────┐           │ Does │  ┌────┐ ┌──────────┐
  │      │──────→│ Buffer[] │           │ other│  │DMA │→│ Buffer[] │
  └──────┘       └──────────┘           │ work!│  └────┘ └──────────┘
                                        └──────┘
  CPU utilization: HIGH                 CPU utilization: ~0%
  
  DMA is like hiring a delivery person — you tell them where to
  pick up and where to deliver, then go back to your own work.
```

### DMA Transfer Types

```
  ┌─────────────────────────────────────────────────────────┐
  │              DMA TRANSFER DIRECTIONS                     │
  │                                                         │
  │  Peripheral → Memory  (e.g., UART RX → buffer)         │
  │  Memory → Peripheral  (e.g., buffer → UART TX)         │
  │  Memory → Memory      (e.g., memcpy replacement)       │
  │                                                         │
  │  DMA Modes:                                             │
  │  ┌────────────┐  Transfer N items, then STOP            │
  │  │  Normal    │  CPU gets interrupt at completion        │
  │  └────────────┘                                         │
  │  ┌────────────┐  Automatically restart after             │
  │  │  Circular  │  completion — continuous transfers       │
  │  └────────────┘  Great for ADC sampling!                 │
  └─────────────────────────────────────────────────────────┘
```

---

## 12.2 STM32 Hardware Internals

### DMA Architecture on STM32F401

```
  STM32F401 has 2 DMA controllers, each with multiple streams:
  
  DMA1 (8 streams) — Mainly APB1 peripherals
  DMA2 (8 streams) — APB2 peripherals + Memory-to-Memory
  
  Each stream has 8 channels (selectable via CHSEL bits).
  
  Example mappings:
  DMA1, Stream 5, Channel 4 = USART2_RX
  DMA1, Stream 6, Channel 4 = USART2_TX
  DMA2, Stream 0, Channel 0 = ADC1
```

### Key DMA Registers

```
  Per-Stream Registers:
  
  DMA_SxCR    — Configuration (enable, direction, channel, etc.)
  DMA_SxNDTR  — Number of Data items To transfer (countdown)
  DMA_SxPAR   — Peripheral Address (e.g., &USART2->DR)
  DMA_SxM0AR  — Memory Address 0 (your buffer address)
  DMA_SxFCR   — FIFO Control
  
  Global Registers:
  DMA_LISR/HISR  — Interrupt Status (transfer complete, error)
  DMA_LIFCR/HIFCR — Interrupt Flag Clear
```

---

## 12.3 Step-by-Step Implementation

### DMA-Driven UART TX

```c
/******************************************************************************
 * File:    dma_uart_tx.c
 * Brief:   Send a string via UART using DMA (zero CPU involvement)
 *
 * Uses DMA1 Stream 6 Channel 4 for USART2_TX
 *****************************************************************************/

#include <stdint.h>

/* Peripheral definitions (abbreviated) */
#define RCC_AHB1ENR     (*(volatile uint32_t *)0x40023830UL)
#define RCC_APB1ENR     (*(volatile uint32_t *)0x40023840UL)

/* USART2 */
#define USART2_SR       (*(volatile uint32_t *)0x40004400UL)
#define USART2_DR       (*(volatile uint32_t *)0x40004404UL)
#define USART2_BRR      (*(volatile uint32_t *)0x40004408UL)
#define USART2_CR1      (*(volatile uint32_t *)0x4000440CUL)
#define USART2_CR3      (*(volatile uint32_t *)0x40004414UL)

/* DMA1 Stream 6 (USART2_TX) */
#define DMA1_BASE       0x40026000UL
#define DMA1_HISR       (*(volatile uint32_t *)(DMA1_BASE + 0x04UL))
#define DMA1_HIFCR      (*(volatile uint32_t *)(DMA1_BASE + 0x0CUL))
#define DMA1_S6CR       (*(volatile uint32_t *)(DMA1_BASE + 0xA0UL))
#define DMA1_S6NDTR     (*(volatile uint32_t *)(DMA1_BASE + 0xA4UL))
#define DMA1_S6PAR      (*(volatile uint32_t *)(DMA1_BASE + 0xA8UL))
#define DMA1_S6M0AR     (*(volatile uint32_t *)(DMA1_BASE + 0xACUL))
#define DMA1_S6FCR      (*(volatile uint32_t *)(DMA1_BASE + 0xB4UL))

/* Initialize UART2 with DMA TX support */
void uart_dma_init(void)
{
    /* Enable clocks: GPIOA, USART2, DMA1 */
    RCC_AHB1ENR |= (1UL << 0) | (1UL << 21);  /* GPIOA + DMA1 */
    RCC_APB1ENR |= (1UL << 17);                /* USART2 */

    /* Configure PA2 as USART2_TX (AF7) — same as Chapter 7 */
    (*(volatile uint32_t *)0x40020000UL) &= ~(3UL << 4);
    (*(volatile uint32_t *)0x40020000UL) |=  (2UL << 4);
    (*(volatile uint32_t *)0x40020020UL) &= ~(0xFUL << 8);
    (*(volatile uint32_t *)0x40020020UL) |=  (7UL << 8);

    /* USART2 configuration: 115200 baud at 16 MHz */
    USART2_BRR = 0x008BUL;   /* 16 MHz / 115200 ≈ 138.9 → 0x8B */
    USART2_CR1 = (1UL << 13) | (1UL << 3);  /* UE + TE */
    USART2_CR3 = (1UL << 7);  /* DMAT: DMA enable transmitter */
}

/* Send a buffer using DMA */
void uart_dma_send(const uint8_t *data, uint16_t length)
{
    /* Disable stream first (required before reconfiguring) */
    DMA1_S6CR &= ~(1UL << 0);
    while (DMA1_S6CR & (1UL << 0));  /* Wait until disabled */

    /* Clear any pending flags for Stream 6 */
    DMA1_HIFCR = (0x3DUL << 16);  /* Clear all Stream 6 flags */

    /* Configure DMA Stream 6 */
    DMA1_S6PAR  = 0x40004404UL;    /* Peripheral address = USART2_DR */
    DMA1_S6M0AR = (uint32_t)data;  /* Memory address = our buffer */
    DMA1_S6NDTR = length;          /* Number of bytes to transfer */

    DMA1_S6CR = (4UL << 25)        /* Channel 4 (USART2_TX) */
              | (1UL << 6)         /* Direction: Memory → Peripheral */
              | (1UL << 10)        /* Memory increment mode */
              | (0UL << 9)         /* Peripheral address fixed */
              | (1UL << 4);        /* Transfer complete interrupt enable */

    /* Enable the DMA stream */
    DMA1_S6CR |= (1UL << 0);

    /* DMA now automatically sends the data! */
    /* Wait for transfer complete (or use interrupt) */
    while (!(DMA1_HISR & (1UL << 21)));  /* TCIF6 */
    DMA1_HIFCR = (1UL << 21);            /* Clear flag */
}

/* Example usage */
int main(void)
{
    uart_dma_init();

    const char msg[] = "Hello from DMA!\r\n";
    uart_dma_send((const uint8_t *)msg, sizeof(msg) - 1);

    while (1)
    {
        /* CPU is free while DMA handles UART transmission */
    }
    return 0;
}
```

---

## 12.4 Common Mistakes

| Mistake | Fix |
|---------|-----|
| Stream not disabled before reconfiguring | Always disable and wait for EN=0 first |
| Wrong channel number | Check reference manual's DMA request mapping table |
| Forgetting to enable DMA in peripheral | Set DMAT/DMAR bit in USART_CR3 |
| Wrong transfer direction | Memory→Peripheral for TX, Peripheral→Memory for RX |
| Not clearing interrupt flags | Use LIFCR/HIFCR to clear flags before starting |

---

## 12.5 Exercises

**Exercise 12.1:** Implement DMA-based UART RX in circular mode to continuously receive data without CPU intervention.

**Exercise 12.2:** Use DMA to transfer ADC conversion results directly to a buffer (ADC + DMA).

**Exercise 12.3:** Compare CPU utilization: measure how much time the CPU spends in polling-mode UART TX vs DMA TX for a 1000-byte message.

---

## 12.6 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 12 SUMMARY                        │
  │                                                            │
  │  ✓ DMA transfers data without CPU involvement              │
  │  ✓ Normal mode: transfer once, then stop                   │
  │  ✓ Circular mode: continuously loop the transfer           │
  │  ✓ Must configure: source, destination, count, direction   │
  │  ✓ Enable DMA request in the peripheral (DMAT, DMAR bits)  │
  │  ✓ Check the DMA channel mapping table in the ref manual   │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 13 — CAN Bus →](./Chapter_13_CAN_Bus.md)*
