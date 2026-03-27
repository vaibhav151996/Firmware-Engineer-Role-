# Chapter 13: CAN Bus
## Controller Area Network for Automotive and Industrial Systems

**Difficulty Level: ★★★★☆ Advanced**  
**Estimated Time: 2 weeks**

---

## 13.1 Concept Overview

### What Is CAN?

CAN (Controller Area Network) is a robust, multi-master communication protocol designed for **noisy environments** like cars and factories. Unlike UART (point-to-point) or I2C (master-slave), CAN allows **any node** to transmit to **all other nodes** on the bus.

```
  CAN Bus Topology:
  
  120Ω                                              120Ω
  ┌─┤├──────────────────────────────────────────────┤├─┐
  │                     CAN_H                           │
  ├─────┬──────────┬──────────┬──────────┬─────────────┤
  │     │          │          │          │              │
  │  ┌──┴──┐   ┌──┴──┐   ┌──┴──┐   ┌──┴──┐           │
  │  │ECU 1│   │ECU 2│   │ECU 3│   │ECU 4│           │
  │  │(Engine)│ │(ABS)│  │(Dash)│  │(STM32)│          │
  │  └──┬──┘   └──┬──┘   └──┬──┘   └──┬──┘           │
  │     │          │          │          │              │
  ├─────┴──────────┴──────────┴──────────┴─────────────┤
  │                     CAN_L                           │
  └─────────────────────────────────────────────────────┘
  
  Two-wire differential bus with 120Ω termination at each end.
  Maximum: 1 Mbit/s at up to 40m, 125 kbit/s at up to 500m.
```

### CAN Frame Format

```
  Standard CAN Frame (CAN 2.0A):
  
  ┌───┬───────────┬───┬──┬────────────────────────┬─────┬───┬───┬───────┐
  │SOF│ 11-bit ID │RTR│r │ DLC (4 bits)           │DATA │CRC│ACK│  EOF  │
  │ 1 │ (Arbitr.) │   │  │ 0-8 bytes              │0-8B │15b│   │ 7bits │
  └───┴───────────┴───┴──┴────────────────────────┴─────┴───┴───┴───────┘
  
  SOF: Start of Frame (dominant bit)
  ID:  Message identifier (lower ID = higher priority)
  RTR: Remote Transmission Request
  DLC: Data Length Code (0-8 bytes)
  DATA: The actual payload (0 to 8 bytes)
  CRC: Error checking (15-bit)
  ACK: Acknowledgment (receiver confirms receipt)
  EOF: End of Frame
  
  Key Concept: ARBITRATION
  If two nodes transmit simultaneously, the one with the
  LOWER ID wins (dominant bit = 0 wins over recessive = 1).
  The loser automatically retries. No data is lost!
```

---

## 13.2 STM32 Hardware Internals

### CAN Peripheral on STM32F401

```
  STM32F401 includes bxCAN (Basic Extended CAN):
  
  CAN1 pins:
    PA11 = CAN_RX (AF9)   — Needs external CAN transceiver
    PA12 = CAN_TX (AF9)   — (e.g., MCP2551, SN65HVD230)
  
  OR:
    PB8 = CAN_RX (AF9)
    PB9 = CAN_TX (AF9)
  
  ┌───────────┐      ┌────────────┐      ┌──────────┐
  │  STM32    │      │   CAN      │      │  CAN     │
  │  bxCAN    │──TX──│Transceiver │──────│  BUS     │
  │  (digital)│──RX──│(MCP2551)   │──────│          │
  └───────────┘      └────────────┘      └──────────┘
  
  The transceiver converts between the STM32's digital
  CAN_TX/CAN_RX and the differential CAN_H/CAN_L bus.
```

### CAN Mailbox System

```
  bxCAN has 3 TX mailboxes and 2 RX FIFOs:
  
  ┌────────────────────────────────────────┐
  │              bxCAN                      │
  │                                        │
  │  TX:  ┌─────┐ ┌─────┐ ┌─────┐         │
  │       │ MB0 │ │ MB1 │ │ MB2 │         │
  │       └──┬──┘ └──┬──┘ └──┬──┘         │
  │          └───────┼───────┘             │
  │                  ↓                     │
  │            TX Scheduler                │
  │         (lowest ID first)              │
  │                  ↓                     │
  │               CAN TX ──→ Bus           │
  │                                        │
  │  RX:      Bus ──→ CAN RX              │
  │                  ↓                     │
  │          Acceptance Filters            │
  │          (28 filter banks)             │
  │            ↓           ↓               │
  │       ┌───────┐   ┌───────┐           │
  │       │ FIFO0 │   │ FIFO1 │           │
  │       │(3 deep)│  │(3 deep)│          │
  │       └───────┘   └───────┘           │
  └────────────────────────────────────────┘
```

---

## 13.3 Step-by-Step Implementation

### Basic CAN TX/RX (Loopback Mode for Testing)

```c
/******************************************************************************
 * File:    can_driver.c
 * Brief:   Basic CAN driver for STM32F401 — loopback mode
 *
 * Loopback mode allows testing WITHOUT external transceiver or bus.
 * The CAN peripheral sends messages to itself.
 *****************************************************************************/

#include <stdint.h>

/* CAN1 register base */
#define CAN1_BASE       0x40006400UL

/* CAN registers (simplified — full struct in production) */
#define CAN_MCR         (*(volatile uint32_t *)(CAN1_BASE + 0x000UL))
#define CAN_MSR         (*(volatile uint32_t *)(CAN1_BASE + 0x004UL))
#define CAN_BTR         (*(volatile uint32_t *)(CAN1_BASE + 0x01CUL))
#define CAN_TSR         (*(volatile uint32_t *)(CAN1_BASE + 0x008UL))

/* TX Mailbox 0 */
#define CAN_TI0R        (*(volatile uint32_t *)(CAN1_BASE + 0x180UL))
#define CAN_TDT0R       (*(volatile uint32_t *)(CAN1_BASE + 0x184UL))
#define CAN_TDL0R       (*(volatile uint32_t *)(CAN1_BASE + 0x188UL))
#define CAN_TDH0R       (*(volatile uint32_t *)(CAN1_BASE + 0x18CUL))

/* RX FIFO 0 */
#define CAN_RF0R        (*(volatile uint32_t *)(CAN1_BASE + 0x00CUL))
#define CAN_RI0R        (*(volatile uint32_t *)(CAN1_BASE + 0x1B0UL))
#define CAN_RDT0R       (*(volatile uint32_t *)(CAN1_BASE + 0x1B4UL))
#define CAN_RDL0R       (*(volatile uint32_t *)(CAN1_BASE + 0x1B8UL))
#define CAN_RDH0R       (*(volatile uint32_t *)(CAN1_BASE + 0x1BCUL))

/* Filter registers */
#define CAN_FMR         (*(volatile uint32_t *)(CAN1_BASE + 0x200UL))
#define CAN_FA1R        (*(volatile uint32_t *)(CAN1_BASE + 0x21CUL))
#define CAN_FS1R        (*(volatile uint32_t *)(CAN1_BASE + 0x20CUL))
#define CAN_FFA1R       (*(volatile uint32_t *)(CAN1_BASE + 0x214UL))
#define CAN_F0R1        (*(volatile uint32_t *)(CAN1_BASE + 0x240UL))
#define CAN_F0R2        (*(volatile uint32_t *)(CAN1_BASE + 0x244UL))

/* RCC */
#define RCC_APB1ENR     (*(volatile uint32_t *)0x40023840UL)

void can_init_loopback(void)
{
    /* Enable CAN1 clock */
    RCC_APB1ENR |= (1UL << 25);

    /* Enter Initialization mode */
    CAN_MCR |= (1UL << 0);   /* INRQ: Request init mode */
    while (!(CAN_MSR & (1UL << 0)));  /* Wait for INAK */

    /* Configure bit timing for 500 kbit/s at 16 MHz APB1
     * BRP = 1 (prescaler = 2)
     * TS1 = 12, TS2 = 3
     * => tq = 2/16MHz = 125ns
     * => Bit time = (1 + 13 + 3) * 125ns = 2.125µs ≈ 470 kbit/s
     * Adjust for exact baudrate as needed
     */
    CAN_BTR = (1UL << 30)    /* LBKM: Loopback mode */
            | (1UL << 0)     /* BRP = 1 (prescaler = 2) */
            | (12UL << 16)   /* TS1 = 12 */
            | (2UL << 20);   /* TS2 = 2 */

    /* Configure acceptance filter: accept ALL messages */
    CAN_FMR  |= (1UL << 0);  /* Enter filter init mode */
    CAN_FS1R |= (1UL << 0);  /* Filter 0: 32-bit scale */
    CAN_F0R1  = 0x00000000UL; /* ID = match all */
    CAN_F0R2  = 0x00000000UL; /* Mask = don't care */
    CAN_FA1R |= (1UL << 0);  /* Activate filter 0 */
    CAN_FMR  &= ~(1UL << 0); /* Leave filter init mode */

    /* Exit Initialization mode → Normal mode */
    CAN_MCR &= ~(1UL << 0);
    while (CAN_MSR & (1UL << 0));  /* Wait for INAK clear */
}

/* Send a CAN message */
void can_send(uint16_t id, uint8_t *data, uint8_t len)
{
    /* Wait for TX mailbox 0 to be empty */
    while (!(CAN_TSR & (1UL << 26)));  /* TME0 */

    /* Set ID (standard 11-bit, shift left by 21) */
    CAN_TI0R = ((uint32_t)id << 21);

    /* Set data length */
    CAN_TDT0R = len & 0x0FUL;

    /* Load data bytes */
    CAN_TDL0R = ((uint32_t)data[3] << 24) | ((uint32_t)data[2] << 16)
              | ((uint32_t)data[1] << 8)  | ((uint32_t)data[0]);
    if (len > 4U)
    {
        CAN_TDH0R = ((uint32_t)data[7] << 24) | ((uint32_t)data[6] << 16)
                   | ((uint32_t)data[5] << 8)  | ((uint32_t)data[4]);
    }

    /* Request transmission */
    CAN_TI0R |= (1UL << 0);  /* TXRQ */
}

/* Receive a CAN message from FIFO 0 */
int can_receive(uint16_t *id, uint8_t *data, uint8_t *len)
{
    /* Check if FIFO 0 has messages */
    if ((CAN_RF0R & 0x03UL) == 0U)
    {
        return 0;  /* No message */
    }

    /* Read ID */
    *id = (uint16_t)((CAN_RI0R >> 21) & 0x7FFUL);

    /* Read length */
    *len = (uint8_t)(CAN_RDT0R & 0x0FUL);

    /* Read data */
    uint32_t low = CAN_RDL0R;
    uint32_t high = CAN_RDH0R;
    data[0] = (uint8_t)(low);
    data[1] = (uint8_t)(low >> 8);
    data[2] = (uint8_t)(low >> 16);
    data[3] = (uint8_t)(low >> 24);
    data[4] = (uint8_t)(high);
    data[5] = (uint8_t)(high >> 8);
    data[6] = (uint8_t)(high >> 16);
    data[7] = (uint8_t)(high >> 24);

    /* Release FIFO 0 */
    CAN_RF0R |= (1UL << 5);  /* RFOM0 */

    return 1;  /* Message received */
}
```

---

## 13.4 Exercises

**Exercise 13.1:** Use loopback mode to send and receive a CAN message. Display the received data via UART.

**Exercise 13.2:** Connect two Nucleo boards with CAN transceivers. Send button state from one board, control LED on the other.

**Exercise 13.3:** Implement a CAN message filter that only accepts messages with ID 0x100–0x1FF.

**Exercise 13.4:** Build a simple CAN bus monitor that logs all messages to UART with timestamp, ID, and data.

---

## 13.5 Industry & Career Notes

- CAN is **mandatory knowledge** for automotive firmware engineers
- Modern cars use CAN-FD (Flexible Data-rate) with up to 64-byte payloads and faster data phase
- J1939 (trucks/heavy equipment) and OBD-II (diagnostics) are higher-level protocols built on CAN
- CAN bus debugging requires a dedicated CAN analyzer tool (PCAN, CANalyzer, etc.)

---

## 13.6 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 13 SUMMARY                        │
  │                                                            │
  │  ✓ CAN is a robust, multi-master differential bus          │
  │  ✓ Message-based (not address-based): ID determines        │
  │    priority (lower ID = higher priority)                   │
  │  ✓ Requires external transceiver (MCP2551, SN65HVD230)    │
  │  ✓ Loopback mode enables testing without hardware          │
  │  ✓ Acceptance filters select which messages to receive     │
  │  ✓ Critical protocol for automotive and industrial         │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 14 — USB Communication →](./Chapter_14_USB.md)*
