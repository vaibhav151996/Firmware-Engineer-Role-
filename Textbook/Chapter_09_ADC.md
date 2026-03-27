# Chapter 9: ADC — Analog to Digital Conversion
## Reading Sensor Voltages and Converting to Digital Values

**Difficulty Level: ★★★☆☆ Intermediate**
**Estimated Time: 2 weeks**

---

## 9.1 Concept Overview

### What Is an ADC?

An ADC (Analog-to-Digital Converter) translates a **continuous analog voltage** into a **discrete digital number** that your firmware can process.

```
  The Real World Is Analog:
  
  Temperature, light, sound, pressure, current — all analog signals.
  
  Analog Signal          ADC              Digital Value
  (Continuous)           (Converter)      (Discrete)
  
  3.3V ┈ ┈ ┈ ┈ ┈┈┈┈    ┌─────────┐
            ╱    ╲       │         │      4095 (0xFFF)
           ╱      ╲      │  12-bit │      
          ╱        ╲     │   ADC   │ ──→  2048
         ╱          ╲    │         │      
  0V ───╱            ╲── │         │      0 (0x000)
                         └─────────┘
  
  Resolution: 12-bit = 4096 levels (0–4095)
  
  Voltage per step = V_ref / 2^n = 3.3V / 4096 = 0.806 mV
```

### ADC Parameters

| Parameter | STM32F401 Value | Meaning |
|-----------|-----------------|---------|
| Resolution | 6, 8, 10, or 12 bits | Number of discrete levels |
| Reference Voltage | 3.3V (V_DDA) | Maximum measurable voltage |
| Channels | 16 external + 3 internal | Number of inputs |
| Sample Rate | Up to 2.4 MSPS | Conversions per second |
| Input Range | 0V to V_REF (3.3V) | Measurable voltage range |

---

## 9.2 STM32 Hardware Internals

### ADC Block Diagram

```
  ┌──────────────────────────────────────────────────────────┐
  │                        ADC1                               │
  │                                                          │
  │  External Inputs:          ┌──────────────────────┐      │
  │  PA0 (CH0) ──┐            │  Sample & Hold       │      │
  │  PA1 (CH1) ──┤            │  Circuit              │      │
  │  PA2 (CH2) ──┤  ┌──────┐ │  ┌────────┐           │      │
  │  PA3 (CH3) ──┼──│ MUX  │─┤  │ S/H    │──→ SAR ──┼──→ DR│
  │  PA4 (CH4) ──┤  │      │ │  │Capacitor│   Logic  │      │
  │  PA5 (CH5) ──┤  └──────┘ │  └────────┘           │      │
  │  PA6 (CH6) ──┤            └──────────────────────┘      │
  │  PA7 (CH7) ──┤                                          │
  │  PB0 (CH8) ──┤  Internal:                               │
  │  PB1 (CH9) ──┤  CH16 ── Temperature Sensor              │
  │  PC0-PC5 ────┘  CH17 ── V_REFINT (1.21V)                │
  │                  CH18 ── V_BAT (battery voltage)         │
  │                                                          │
  │  Key Registers:                                          │
  │    ADC_SR   ── Status (EOC flag)                         │
  │    ADC_CR1  ── Control (resolution, scan mode)           │
  │    ADC_CR2  ── Control (enable, start, alignment)        │
  │    ADC_SMPR ── Sample time per channel                   │
  │    ADC_SQR  ── Sequence (which channels, what order)     │
  │    ADC_DR   ── Data register (conversion result)         │
  └──────────────────────────────────────────────────────────┘
```

### ADC Conversion Process

```
  ADC Conversion Steps:
  
  1. SAMPLING                      2. CONVERSION (SAR)
  ┌─────────────────────┐          ┌─────────────────────┐
  │ Switch connects pin │          │ SAR algorithm:      │
  │ to S/H capacitor    │          │ 12 iterations for   │
  │                     │          │ 12-bit result       │
  │ Capacitor charges   │          │                     │
  │ to input voltage    │          │ Bit 11: Is V > 1.65?│
  │                     │          │ Bit 10: Is V > ...? │
  │ Duration: SMPR bits │          │ ...                 │
  │ (3-480 cycles)      │          │ Bit 0: final bit    │
  └─────────┬───────────┘          └─────────┬───────────┘
            │                                │
            └───────── Total: ──────────────→│
              Sampling + 12 cycles            ↓
              = Conversion Time         Result in ADC_DR
  
  At 12-bit, f_ADC = 30 MHz:
  Fastest conversion = (3 + 12) cycles / 30 MHz = 0.5 µs = 2 MSPS
```

---

## 9.3 Step-by-Step Implementation

### Single-Channel ADC Reading

```c
/******************************************************************************
 * File:    adc_driver.c
 * Brief:   ADC driver for STM32F401 — single conversion, polling
 * Board:   Nucleo-F401RE
 *
 * Read analog voltage on PA0 (ADC1_CH0)
 * Connect a potentiometer or sensor to PA0
 *****************************************************************************/

#include <stdint.h>

/* RCC */
#define RCC_AHB1ENR     (*(volatile uint32_t *)0x40023830UL)
#define RCC_APB2ENR     (*(volatile uint32_t *)0x40023844UL)

/* GPIO */
#define GPIOA_MODER     (*(volatile uint32_t *)0x40020000UL)

/* ADC1 */
#define ADC1_BASE       0x40012000UL
#define ADC1_SR         (*(volatile uint32_t *)(ADC1_BASE + 0x00UL))
#define ADC1_CR1        (*(volatile uint32_t *)(ADC1_BASE + 0x04UL))
#define ADC1_CR2        (*(volatile uint32_t *)(ADC1_BASE + 0x08UL))
#define ADC1_SMPR2      (*(volatile uint32_t *)(ADC1_BASE + 0x10UL))
#define ADC1_SQR3       (*(volatile uint32_t *)(ADC1_BASE + 0x34UL))
#define ADC1_DR         (*(volatile uint32_t *)(ADC1_BASE + 0x4CUL))

/* Bits */
#define ADC_SR_EOC      (1UL << 1)   /* End of conversion */
#define ADC_CR2_ADON    (1UL << 0)   /* ADC enable */
#define ADC_CR2_SWSTART (1UL << 30)  /* Start conversion */
#define ADC_CR2_CONT    (1UL << 1)   /* Continuous mode */

void adc_init(void)
{
    /* Enable clocks */
    RCC_AHB1ENR |= (1UL << 0);    /* GPIOA */
    RCC_APB2ENR |= (1UL << 8);    /* ADC1 */

    /* Configure PA0 as analog input */
    GPIOA_MODER |= (3UL << 0);    /* Mode = 11 (analog) */

    /* ADC Configuration:
     * - 12-bit resolution (default)
     * - Right alignment (default)
     * - Single conversion mode
     * - 84 cycles sample time for CH0 (SMPR2[2:0] = 100)
     */
    ADC1_SMPR2 |= (4UL << 0);     /* 84 cycles sample time for CH0 */
    ADC1_SQR3  = 0UL;             /* First (and only) conversion = CH0 */

    /* Enable ADC */
    ADC1_CR2 |= ADC_CR2_ADON;
}

uint16_t adc_read(void)
{
    /* Start conversion */
    ADC1_CR2 |= ADC_CR2_SWSTART;

    /* Wait for End of Conversion */
    while (!(ADC1_SR & ADC_SR_EOC))
    {
        /* Polling — wait */
    }

    /* Read and return the result (12 bits, 0-4095) */
    return (uint16_t)(ADC1_DR & 0xFFFUL);
}

/*
 * Convert ADC value to millivolts
 * V = (ADC_value / 4095) × 3300 mV
 * Using integer math to avoid floating point:
 * V_mV = ADC_value × 3300 / 4095
 */
uint32_t adc_to_millivolts(uint16_t adc_value)
{
    return ((uint32_t)adc_value * 3300UL) / 4095UL;
}

/*
 * Convert millivolts to temperature (for LM35 sensor)
 * LM35: 10 mV per °C
 * Temperature = V_mV / 10
 */
int32_t millivolts_to_temp_c(uint32_t mv)
{
    return (int32_t)(mv / 10UL);
}
```

### Main Application: Read and Display via UART

```c
#include <stdint.h>
#include "uart_driver.h"   /* From Chapter 7 */

extern void adc_init(void);
extern uint16_t adc_read(void);
extern uint32_t adc_to_millivolts(uint16_t adc_value);
extern void delay_ms(uint32_t ms);

int main(void)
{
    uart_init(USART2, 16000000UL, 115200UL);
    adc_init();

    uart_send_string(USART2, "\r\n=== ADC Temperature Monitor ===\r\n");

    while (1)
    {
        uint16_t raw = adc_read();
        uint32_t mv  = adc_to_millivolts(raw);

        uart_send_string(USART2, "ADC: ");
        uart_send_number(USART2, (int32_t)raw);
        uart_send_string(USART2, "  Voltage: ");
        uart_send_number(USART2, (int32_t)mv);
        uart_send_string(USART2, " mV\r\n");

        delay_ms(500);
    }
}
```

---

## 9.4 Common Mistakes & Debugging

| Mistake | Symptom | Fix |
|---------|---------|-----|
| GPIO not in analog mode | ADC reads noise or 0 | Set MODER to 11 (analog) |
| ADC clock not enabled | Reads always 0 | Enable bit 8 in RCC_APB2ENR |
| Wrong channel in SQR3 | Reading wrong pin | Verify channel number matches pin |
| Integer overflow in conversion | Wrong millivolt values | Use uint32_t for intermediate calculations |
| No sample time configured | Inaccurate readings | Set appropriate SMPR value for your impedance |

---

## 9.5 Exercises

**Exercise 9.1:** Read the internal temperature sensor (CH16) and display the die temperature via UART. The formula is: T(°C) = ((V_sense - V25) / Avg_Slope) + 25.

**Exercise 9.2:** Implement oversampling: take 16 ADC samples, average them, and display the result. Compare noise levels with and without averaging.

**Exercise 9.3:** Use the ADC to read a potentiometer and control LED brightness via PWM (combine Chapters 8 and 9).

**Exercise 9.4:** Implement a threshold alarm: if the voltage exceeds a preset limit, toggle an LED and send a UART warning message.

---

## 9.6 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │                  CHAPTER 9 SUMMARY                         │
  │                                                            │
  │  ✓ ADC converts analog voltage (0–3.3V) to digital        │
  │    value (0–4095 at 12-bit resolution)                     │
  │                                                            │
  │  ✓ GPIO pin MUST be in analog mode (MODER = 11)            │
  │                                                            │
  │  ✓ Conversion: Start → Wait for EOC → Read DR              │
  │                                                            │
  │  ✓ V_mV = (ADC_value × 3300) / 4095                       │
  │                                                            │
  │  ✓ Sample time affects accuracy — longer for high           │
  │    impedance sources                                       │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 10 — SPI Protocol →](./Chapter_10_SPI.md)*
