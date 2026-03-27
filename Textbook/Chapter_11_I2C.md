# Chapter 11: I2C Protocol
## Two-Wire Multi-Device Bus Communication

**Difficulty Level: ★★★☆☆ Intermediate**
**Estimated Time: 1–2 weeks**

---

## 11.1 Concept Overview

### What Is I2C?

I2C (Inter-Integrated Circuit, pronounced "I-squared-C") is a **synchronous, half-duplex, addressed** two-wire protocol. Unlike SPI, multiple slave devices share the same bus using unique addresses — no additional chip select pins needed.

```
  I2C Bus Topology:
  
       VDD (3.3V)
        │       │
       ┌┴┐    ┌┴┐     Pull-up resistors (4.7kΩ typical)
       │R│    │R│
       └┬┘    └┬┘
        │      │
  ──────┤──────┤─────── SDA (Data)
        │      │
  ──────┤──────┤─────── SCL (Clock)
        │      │
  ┌─────┴──┐ ┌─┴──────┐ ┌───────┐
  │ MASTER │ │ SLAVE 1 │ │SLAVE 2│
  │ STM32  │ │ Accel   │ │ OLED  │
  │        │ │ 0x68    │ │ 0x3C  │
  └────────┘ └─────────┘ └───────┘
  
  Key: Only 2 wires for ANY number of devices (up to 127 addresses)
```

### I2C Signal Protocol

```
  I2C Transaction: Write 0x42 to register 0x10 of slave 0x68
  
  SDA: ─┐  ┌─────┐   ┐   ┐   ┐   ┐   ┐   ┐   ┐   ┐─┐ ...
        └──┤ 1  1 ├ 0 ┤ 1 ┤ 0 ┤ 0 ┤ 0 ┤ 0 ┤ A ┤   │ │
           └─────┘    └───┘   └───┘   └───┘  C  │   │ │
  SCL: ────┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ K┌─┐│ │
           └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘  └─┘
        │S│     Address (7 bits): 0x68      │W│ACK│
        │T│  1   1   0   1   0   0   0     │ │   │  
        │A│                                 │0│   │
        │R│                                 │ │   │
        │T│                                 │ │   │
  
  Sequence:
  1. START condition (SDA goes LOW while SCL is HIGH)
  2. 7-bit slave address + R/W bit (0=Write, 1=Read)
  3. ACK from slave (slave pulls SDA LOW)
  4. Register address byte
  5. ACK from slave
  6. Data byte
  7. ACK from slave
  8. STOP condition (SDA goes HIGH while SCL is HIGH)
```

---

## 11.2 STM32 Hardware Internals

### I2C on STM32F401

```
  I2C1 pins:
    PB6 = SCL (AF4)
    PB7 = SDA (AF4)
  
  OR (alternate):
    PB8 = SCL (AF4)
    PB9 = SDA (AF4)
  
  Important: GPIO must be configured as:
    - Alternate Function mode
    - Open-drain output type (!!)
    - Pull-up (internal or external)
  
  I2C1 Base: 0x40005400 (APB1)
  
  Key Registers:
    CR1   — Control (enable, start, stop, ack)
    CR2   — Clock control (peripheral frequency)
    OAR1  — Own address (for slave mode)
    DR    — Data register
    SR1   — Status (start bit sent, address sent, BTF, etc.)
    SR2   — Status (busy, master/slave mode)
    CCR   — Clock control register (I2C speed)
    TRISE — Rise time configuration
```

### I2C Clock Speed Configuration

```
  Standard Mode (100 kHz):
    CCR = f_PCLK1 / (2 × 100,000)
    For 16 MHz APB1: CCR = 16,000,000 / 200,000 = 80
    
  Fast Mode (400 kHz):
    CCR = f_PCLK1 / (3 × 400,000)  [with duty=0]
    For 16 MHz APB1: CCR = 16,000,000 / 1,200,000 ≈ 14
    
  TRISE = (f_PCLK1 / 1,000,000) + 1
    For 16 MHz: TRISE = 17
```

---

## 11.3 Step-by-Step Implementation

### I2C Master Driver

```c
/******************************************************************************
 * File:    i2c_driver.c
 * Brief:   I2C1 master driver for STM32F401 — register-level
 *
 * I2C1 pins: PB6 (SCL), PB7 (SDA) — both AF4, open-drain
 *****************************************************************************/

#include <stdint.h>

typedef struct {
    volatile uint32_t CR1;
    volatile uint32_t CR2;
    volatile uint32_t OAR1;
    volatile uint32_t OAR2;
    volatile uint32_t DR;
    volatile uint32_t SR1;
    volatile uint32_t SR2;
    volatile uint32_t CCR;
    volatile uint32_t TRISE;
} I2C_TypeDef;

#define I2C1    ((I2C_TypeDef *)0x40005400UL)

/* RCC */
#define RCC_AHB1ENR     (*(volatile uint32_t *)0x40023830UL)
#define RCC_APB1ENR     (*(volatile uint32_t *)0x40023840UL)

/* GPIOB registers */
#define GPIOB_MODER     (*(volatile uint32_t *)0x40020400UL)
#define GPIOB_OTYPER    (*(volatile uint32_t *)0x40020404UL)
#define GPIOB_PUPDR     (*(volatile uint32_t *)0x4002040CUL)
#define GPIOB_AFRL      (*(volatile uint32_t *)0x40020420UL)

/* I2C CR1 bits */
#define I2C_CR1_PE      (1UL << 0)
#define I2C_CR1_START   (1UL << 8)
#define I2C_CR1_STOP    (1UL << 9)
#define I2C_CR1_ACK     (1UL << 10)
#define I2C_CR1_SWRST   (1UL << 15)

/* I2C SR1 bits */
#define I2C_SR1_SB      (1UL << 0)   /* Start bit generated */
#define I2C_SR1_ADDR    (1UL << 1)   /* Address sent/matched */
#define I2C_SR1_BTF     (1UL << 2)   /* Byte transfer finished */
#define I2C_SR1_TXE     (1UL << 7)   /* Data register empty (TX) */
#define I2C_SR1_RXNE    (1UL << 6)   /* Data register not empty (RX) */

void i2c1_init(void)
{
    /* Enable clocks */
    RCC_AHB1ENR |= (1UL << 1);    /* GPIOB */
    RCC_APB1ENR |= (1UL << 21);   /* I2C1 */

    /* PB6 (SCL) and PB7 (SDA) as AF4, open-drain, pull-up */
    GPIOB_MODER &= ~((3UL << 12) | (3UL << 14));
    GPIOB_MODER |=  ((2UL << 12) | (2UL << 14));  /* AF mode */

    GPIOB_OTYPER |= (1UL << 6) | (1UL << 7);      /* Open-drain!! */

    GPIOB_PUPDR &= ~((3UL << 12) | (3UL << 14));
    GPIOB_PUPDR |=  ((1UL << 12) | (1UL << 14));   /* Pull-up */

    GPIOB_AFRL &= ~((0xFUL << 24) | (0xFUL << 28));
    GPIOB_AFRL |=  ((4UL << 24) | (4UL << 28));    /* AF4 */

    /* Reset I2C peripheral */
    I2C1->CR1 |= I2C_CR1_SWRST;
    I2C1->CR1 &= ~I2C_CR1_SWRST;

    /* Configure I2C clock: assume APB1 = 16 MHz */
    I2C1->CR2 = 16UL;              /* APB1 frequency in MHz */
    I2C1->CCR = 80UL;              /* 100 kHz standard mode */
    I2C1->TRISE = 17UL;            /* (16 MHz / 1 MHz) + 1 */

    /* Enable I2C */
    I2C1->CR1 |= I2C_CR1_PE;
}

/*
 * i2c1_write() — Write data to a slave device
 * addr: 7-bit slave address (will be shifted left by hardware)
 * reg:  register address on the slave
 * data: byte to write
 */
void i2c1_write(uint8_t addr, uint8_t reg, uint8_t data)
{
    /* Generate START condition */
    I2C1->CR1 |= I2C_CR1_START;
    while (!(I2C1->SR1 & I2C_SR1_SB));  /* Wait for START */

    /* Send slave address with WRITE (bit 0 = 0) */
    I2C1->DR = (addr << 1) | 0U;
    while (!(I2C1->SR1 & I2C_SR1_ADDR));
    (void)I2C1->SR2;  /* Clear ADDR flag by reading SR2 */

    /* Send register address */
    I2C1->DR = reg;
    while (!(I2C1->SR1 & I2C_SR1_TXE));

    /* Send data byte */
    I2C1->DR = data;
    while (!(I2C1->SR1 & I2C_SR1_BTF));

    /* Generate STOP condition */
    I2C1->CR1 |= I2C_CR1_STOP;
}

/*
 * i2c1_read() — Read data from a slave device
 * Uses repeated start for register read
 */
uint8_t i2c1_read(uint8_t addr, uint8_t reg)
{
    uint8_t data;

    /* --- Phase 1: Send register address --- */
    I2C1->CR1 |= I2C_CR1_START;
    while (!(I2C1->SR1 & I2C_SR1_SB));

    I2C1->DR = (addr << 1) | 0U;  /* Write mode */
    while (!(I2C1->SR1 & I2C_SR1_ADDR));
    (void)I2C1->SR2;

    I2C1->DR = reg;
    while (!(I2C1->SR1 & I2C_SR1_TXE));

    /* --- Phase 2: Repeated start + Read --- */
    I2C1->CR1 |= I2C_CR1_START;
    while (!(I2C1->SR1 & I2C_SR1_SB));

    I2C1->DR = (addr << 1) | 1U;  /* Read mode */
    while (!(I2C1->SR1 & I2C_SR1_ADDR));

    /* Disable ACK (for single byte read) */
    I2C1->CR1 &= ~I2C_CR1_ACK;
    (void)I2C1->SR2;

    /* Wait for data */
    while (!(I2C1->SR1 & I2C_SR1_RXNE));
    data = (uint8_t)I2C1->DR;

    /* Generate STOP */
    I2C1->CR1 |= I2C_CR1_STOP;

    return data;
}
```

---

## 11.4 Common Mistakes & Debugging

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing pull-up resistors | Bus stuck, no communication | Add 4.7kΩ pull-ups to SDA and SCL |
| GPIO not open-drain | Bus contention, garbled data | OTYPER must be open-drain for I2C pins |
| Wrong slave address | ADDR flag never sets | Verify 7-bit address (not 8-bit shifted) |
| Not reading SR2 after ADDR | I2C hangs | Must read SR1 then SR2 to clear ADDR flag |
| Bus lockup | SDA stuck LOW | Reset I2C peripheral, toggle SCL manually |

### Debugging with Logic Analyzer

A logic analyzer is **essential** for I2C debugging. Decode the SDA/SCL signals to see:
- START/STOP conditions
- Slave address and ACK/NACK
- Data bytes being transmitted
- Timing violations

---

## 11.5 Exercises

**Exercise 11.1:** Interface with an MPU6050 accelerometer/gyroscope (address 0x68). Read the WHO_AM_I register (0x75) and verify it returns 0x68.

**Exercise 11.2:** Write a bus scanner that tries all 127 addresses and reports which devices respond with ACK.

**Exercise 11.3:** Display accelerometer data from MPU6050 via UART at 10 Hz.

**Exercise 11.4:** Interface with an SSD1306 OLED display (0x3C) over I2C. Display text on the screen.

---

## 11.6 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 11 SUMMARY                        │
  │                                                            │
  │  ✓ I2C uses 2 wires (SDA, SCL) with pull-up resistors     │
  │  ✓ Addressed protocol: each device has a unique 7-bit ID  │
  │  ✓ Half-duplex: can't send and receive simultaneously     │
  │  ✓ GPIO MUST be open-drain for I2C pins                   │
  │  ✓ The ADDR flag clear sequence (read SR1, then SR2)      │
  │    is a common gotcha in STM32 I2C                         │
  │  ✓ Pull-up resistors are mandatory (4.7kΩ typical)        │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 12 — DMA: Direct Memory Access →](./Chapter_12_DMA.md)*
