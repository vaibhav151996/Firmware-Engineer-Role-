# Chapter 17: FreeRTOS on STM32
## ★ MILESTONE: Multi-Task RTOS Application

**Difficulty Level: ★★★★☆ Advanced**  
**Estimated Time: 3 weeks**

---

## 17.1 Concept Overview

### What Is an RTOS?

An RTOS (Real-Time Operating System) is a lightweight operating system that manages **multiple tasks** running "simultaneously" on a single CPU. Unlike a desktop OS, an RTOS is designed for **deterministic, time-critical** embedded applications.

```
  Bare-Metal (Super Loop):              RTOS (Multi-Tasking):
  
  while (1) {                           ┌─────────┐
      read_sensors();     ← Sequential  │ Task 1  │ Read sensors
      process_data();                    │ Task 2  │ Process data  
      update_display();   ← If one      │ Task 3  │ Update display
      check_uart();          blocks,     │ Task 4  │ Check UART
  }                          ALL stop!   └─────────┘
                                         ↕ Scheduler switches
                                         between tasks rapidly
  
  Problem: If read_sensors() takes         Each task runs
  100ms, UART is unresponsive for          independently!
  100ms!                                   Blocked task doesn't
                                           affect others.
```

### Why FreeRTOS?

FreeRTOS is the most widely used RTOS in embedded systems:
- **Free and open-source** (MIT license)
- Tiny footprint: ~5-10 KB Flash, ~1 KB RAM overhead
- Supports all Cortex-M devices
- Used in billions of devices (Amazon acquired it)
- Excellent documentation and community

### RTOS Core Concepts

```
  ┌──────────────────────────────────────────────────────────┐
  │              FREERTOS CONCEPTS                            │
  │                                                          │
  │  TASK:                                                   │
  │    An independent function with its own stack.           │
  │    Each task thinks it "owns" the CPU.                   │
  │    The scheduler switches between tasks.                 │
  │                                                          │
  │  SCHEDULER:                                              │
  │    Decides which task runs next.                         │
  │    Uses priority-based preemptive scheduling.            │
  │    Highest priority ready task always runs.              │
  │                                                          │
  │  QUEUE:                                                  │
  │    Thread-safe FIFO buffer for passing data              │
  │    between tasks (producer → consumer).                  │
  │                                                          │
  │  SEMAPHORE:                                              │
  │    Signaling mechanism for synchronization.              │
  │    Binary semaphore: signal an event occurred.           │
  │    Counting semaphore: track available resources.        │
  │                                                          │
  │  MUTEX:                                                  │
  │    Mutual exclusion — protects shared resources.         │
  │    Only one task can "hold" the mutex at a time.         │
  │                                                          │
  │  TASK STATES:                                            │
  │  ┌─────────┐     ┌─────────┐     ┌──────────┐           │
  │  │ READY   │────→│ RUNNING │────→│ BLOCKED  │           │
  │  │(waiting │←────│(on CPU) │     │(waiting  │           │
  │  │ to run) │     └─────────┘     │ for event│           │
  │  └─────────┘          ↑          │ /timeout)│           │
  │       ↑               │          └──────┬───┘           │
  │       └───────────────┴─────────────────┘               │
  │       (event occurred / timeout expired)                │
  └──────────────────────────────────────────────────────────┘
```

### Task Scheduling Visualization

```
  Time →
  
  Priority 3  Task_LED:    ██    ██    ██    ██    ██    ██
  (Highest)                (toggles LED every 500ms)

  Priority 2  Task_UART:        ████            ████
  (Medium)                     (sends data when ready)

  Priority 1  Task_Idle:  ██  ██    ████  ████  ██    ████
  (Lowest)                (runs when nothing else needs CPU)
  
  The scheduler preempts lower-priority tasks when a
  higher-priority task becomes READY (e.g., after a delay expires).
```

---

## 17.2 STM32 Hardware Internals

### How FreeRTOS Uses Cortex-M Hardware

```
  FreeRTOS relies on these Cortex-M hardware features:
  
  ┌────────────────────────────────────────────────────┐
  │  SysTick Timer:                                    │
  │    Generates periodic interrupt (the "tick")        │
  │    Typically 1 ms period                           │
  │    Drives the scheduler — vTaskDelay() counts       │
  │    ticks, not real time                            │
  │                                                    │
  │  PendSV (Pendable Service Call):                   │
  │    Software-triggered exception at LOWEST priority  │
  │    Used for context switching                      │
  │    Switches the stack pointer and saved registers   │
  │    to the next task's stack                         │
  │                                                    │
  │  SVCall (Supervisor Call):                          │
  │    Used to start the first task                    │
  │                                                    │
  │  Each task gets its own STACK:                     │
  │  ┌────────────┐  ┌────────────┐  ┌────────────┐   │
  │  │ Task1 Stack│  │ Task2 Stack│  │ Task3 Stack│   │
  │  │ (256 words)│  │ (128 words)│  │ (256 words)│   │
  │  │ R0-R12, LR │  │ R0-R12, LR │  │ R0-R12, LR │  │
  │  │ PC, xPSR   │  │ PC, xPSR   │  │ PC, xPSR   │  │
  │  │ Local vars │  │ Local vars │  │ Local vars │   │
  │  └────────────┘  └────────────┘  └────────────┘   │
  │        ↑               ↑               ↑          │
  │        └───────────────┴───────────────┘          │
  │         Allocated from the FreeRTOS heap           │
  │         (configTOTAL_HEAP_SIZE in FreeRTOSConfig.h)│
  └────────────────────────────────────────────────────┘
```

---

## 17.3 Step-by-Step Implementation

### Adding FreeRTOS to an STM32 Project

```
  Method 1: STM32CubeIDE with CubeMX (recommended for beginners)
  1. Create new project → Select Nucleo-F401RE
  2. Middleware → FREERTOS → Interface: CMSIS_V2
  3. Add tasks via CubeMX GUI
  4. Generate code
  
  Method 2: Manual integration (recommended for understanding)
  1. Download FreeRTOS source from freertos.org
  2. Add to your project:
     - Source/tasks.c, queue.c, list.c, timers.c
     - Source/portable/GCC/ARM_CM4F/port.c
     - Source/portable/MemMang/heap_4.c
     - Include/FreeRTOS.h, task.h, queue.h, semphr.h
  3. Create FreeRTOSConfig.h
  4. Configure SysTick and PendSV handlers
```

### FreeRTOSConfig.h — Key Settings

```c
/******************************************************************************
 * File:    FreeRTOSConfig.h
 * Brief:   FreeRTOS configuration for STM32F401RE
 *****************************************************************************/
#ifndef FREERTOS_CONFIG_H
#define FREERTOS_CONFIG_H

/* Core configuration */
#define configUSE_PREEMPTION            1    /* Preemptive scheduling */
#define configUSE_IDLE_HOOK             0
#define configUSE_TICK_HOOK             0
#define configCPU_CLOCK_HZ             (84000000UL)  /* 84 MHz */
#define configTICK_RATE_HZ             ((TickType_t)1000)  /* 1 kHz = 1 ms tick */
#define configMAX_PRIORITIES           (5)   /* Priority levels 0-4 */
#define configMINIMAL_STACK_SIZE       ((uint16_t)128)  /* Words (512 bytes) */
#define configTOTAL_HEAP_SIZE          ((size_t)16384)  /* 16 KB for FreeRTOS */
#define configMAX_TASK_NAME_LEN        (16)

/* Feature enables */
#define configUSE_MUTEXES               1
#define configUSE_COUNTING_SEMAPHORES   1
#define configUSE_QUEUE_SETS            0
#define configUSE_TRACE_FACILITY        1    /* Enable for debugging */
#define configUSE_16_BIT_TICKS          0    /* 32-bit tick counter */

/* Hook functions */
#define configCHECK_FOR_STACK_OVERFLOW  2   /* Stack overflow detection! */

/* Cortex-M specific */
#define configPRIO_BITS                 4   /* STM32F4 has 4 priority bits */
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY     15
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY 5

/* Map FreeRTOS handlers to Cortex-M handlers */
#define vPortSVCHandler     SVC_Handler
#define xPortPendSVHandler  PendSV_Handler
#define xPortSysTickHandler SysTick_Handler

/* Include optional functions */
#define INCLUDE_vTaskDelay              1
#define INCLUDE_vTaskDelayUntil         1
#define INCLUDE_vTaskDelete             1
#define INCLUDE_xTaskGetSchedulerState  1
#define INCLUDE_uxTaskGetStackHighWaterMark 1

#endif /* FREERTOS_CONFIG_H */
```

### Complete FreeRTOS Application

```c
/******************************************************************************
 * File:    main.c
 * Brief:   FreeRTOS multi-task application on STM32 Nucleo-F401RE
 *
 * ★ MILESTONE: FreeRTOS Application
 *
 * Tasks:
 *   1. LED Task (Priority 2) — Blinks LD2 at 1 Hz
 *   2. UART Task (Priority 3) — Sends sensor data every second
 *   3. Button Task (Priority 3) — Responds to button press
 *
 * Inter-Task Communication:
 *   Queue: Button task → UART task (send messages)
 *   Semaphore: Button ISR → Button task (signal press)
 *****************************************************************************/

#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include "semphr.h"
#include <stdint.h>
#include <string.h>

/* Hardware drivers (from previous chapters) */
extern void clock_init_84mhz(void);
extern void gpio_clock_enable(void *port);
extern void uart_init(void *uart, uint32_t pclk, uint32_t baud);
extern void uart_send_string(void *uart, const char *str);
extern void uart_send_number(void *uart, int32_t num);

/* GPIO/peripheral definitions */
#define GPIOA_BASE      0x40020000UL
#define GPIOA           ((void *)GPIOA_BASE)
#define GPIOA_ODR       (*(volatile uint32_t *)(GPIOA_BASE + 0x14UL))
#define GPIOA_MODER     (*(volatile uint32_t *)(GPIOA_BASE + 0x00UL))

#define GPIOC_BASE      0x40020800UL
#define GPIOC_IDR       (*(volatile uint32_t *)(GPIOC_BASE + 0x10UL))

#define USART2          ((void *)0x40004400UL)

#define RCC_AHB1ENR     (*(volatile uint32_t *)0x40023830UL)

/* FreeRTOS objects */
static QueueHandle_t  uart_queue;      /* Queue for messages to UART */
static SemaphoreHandle_t btn_semaphore; /* Semaphore for button press */

/* Message type for the queue */
typedef struct {
    char text[32];
    int32_t value;
} UartMessage_t;

/* ========================================================================= */
/*                           TASK FUNCTIONS                                  */
/* ========================================================================= */

/*
 * Task 1: LED Blink
 * Toggles LD2 every 500ms using vTaskDelay
 * This demonstrates the simplest RTOS task.
 */
void task_led_blink(void *params)
{
    (void)params;  /* Unused parameter */

    while (1)
    {
        GPIOA_ODR ^= (1UL << 5);       /* Toggle PA5 (LD2) */
        vTaskDelay(pdMS_TO_TICKS(500)); /* Delay 500ms (non-blocking!) */
        /* vTaskDelay puts this task to BLOCKED state.
         * Other tasks can run during this time! */
    }
}

/*
 * Task 2: UART Logger
 * Waits for messages on the queue and sends them via UART.
 * This demonstrates queue-based inter-task communication.
 */
void task_uart_logger(void *params)
{
    (void)params;
    UartMessage_t msg;

    uart_send_string(USART2, "\r\n=== FreeRTOS Started ===\r\n");

    while (1)
    {
        /* Block until a message arrives in the queue */
        if (xQueueReceive(uart_queue, &msg, pdMS_TO_TICKS(1000)) == pdTRUE)
        {
            uart_send_string(USART2, msg.text);
            if (msg.value != 0)
            {
                uart_send_string(USART2, " Value: ");
                uart_send_number(USART2, msg.value);
            }
            uart_send_string(USART2, "\r\n");
        }
        else
        {
            /* Timeout — no message for 1 second */
            uart_send_string(USART2, "[Heartbeat] System running...\r\n");
        }
    }
}

/*
 * Task 3: Button Handler
 * Waits for button semaphore, then sends a message to the queue.
 * This demonstrates semaphore-based event signaling.
 */
void task_button_handler(void *params)
{
    (void)params;
    uint32_t press_count = 0;
    UartMessage_t msg;

    while (1)
    {
        /* Wait for the button semaphore (given by ISR or polling) */
        if (xSemaphoreTake(btn_semaphore, pdMS_TO_TICKS(50)) == pdTRUE)
        {
            press_count++;

            /* Send a message to UART task via queue */
            strncpy(msg.text, "[BUTTON] Press #", sizeof(msg.text));
            msg.value = (int32_t)press_count;

            xQueueSend(uart_queue, &msg, pdMS_TO_TICKS(100));
        }

        /* Simple polling for button (PC13, active LOW) */
        if (!(GPIOC_IDR & (1UL << 13)))
        {
            vTaskDelay(pdMS_TO_TICKS(200));  /* Debounce */
            if (!(GPIOC_IDR & (1UL << 13)))
            {
                xSemaphoreGive(btn_semaphore);
                /* Wait for button release */
                while (!(GPIOC_IDR & (1UL << 13)))
                {
                    vTaskDelay(pdMS_TO_TICKS(10));
                }
            }
        }
    }
}

/* ========================================================================= */
/*                     STACK OVERFLOW HOOK                                    */
/* ========================================================================= */

/*
 * Called by FreeRTOS when a stack overflow is detected.
 * This is a CRITICAL error — in production, log it and reset.
 */
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    (void)xTask;
    (void)pcTaskName;

    /* Infinite loop with LED rapid blink to indicate error */
    while (1)
    {
        GPIOA_ODR ^= (1UL << 5);
        for (volatile uint32_t i = 0; i < 100000; i++);
    }
}

/* ========================================================================= */
/*                              MAIN                                         */
/* ========================================================================= */

int main(void)
{
    /* Initialize hardware */
    clock_init_84mhz();

    /* Enable GPIO clocks */
    RCC_AHB1ENR |= (1UL << 0) | (1UL << 2);  /* GPIOA + GPIOC */

    /* Configure PA5 as output (LED) */
    GPIOA_MODER &= ~(3UL << 10);
    GPIOA_MODER |=  (1UL << 10);

    /* Initialize UART */
    uart_init(USART2, 42000000UL, 115200UL);  /* APB1 = 42 MHz */

    /* ===================================================================
     * Create FreeRTOS objects
     * =================================================================== */

    /* Create a queue for 5 messages */
    uart_queue = xQueueCreate(5, sizeof(UartMessage_t));

    /* Create a binary semaphore for button signaling */
    btn_semaphore = xSemaphoreCreateBinary();

    /* ===================================================================
     * Create tasks
     *
     * xTaskCreate(function, name, stack_size, parameters, priority, handle)
     *
     * Stack size is in WORDS (4 bytes each on 32-bit ARM)
     * 256 words = 1024 bytes = 1 KB per task
     * =================================================================== */

    xTaskCreate(task_led_blink,     "LED",    128, NULL, 2, NULL);
    xTaskCreate(task_uart_logger,   "UART",   256, NULL, 3, NULL);
    xTaskCreate(task_button_handler,"Button", 256, NULL, 3, NULL);

    /* ===================================================================
     * Start the scheduler — this function NEVER returns!
     * =================================================================== */
    vTaskStartScheduler();

    /* If we get here, there's not enough RAM for the idle task */
    while (1);

    return 0;
}
```

---

## 17.4 Diagrams

### Task Communication Flow

```
  ┌──────────────┐         ┌──────────────┐
  │  Button Task │         │  UART Task   │
  │              │         │              │
  │  Detects     │   Queue │  Waits for   │
  │  button      │────────→│  messages    │
  │  press       │         │  from queue  │
  │              │         │              │
  │  Sends msg   │         │  Formats and │
  │  to queue    │         │  sends via   │
  │              │         │  UART        │
  └──────────────┘         └──────────────┘
  
  ┌──────────────┐
  │  LED Task    │         (Independent — no communication)
  │              │
  │  Toggles LED │
  │  every 500ms │
  └──────────────┘
```

### Memory Usage with FreeRTOS

```
  SRAM Usage (96 KB total):
  
  0x2000_0000  ┌──────────────────────┐
               │  .data + .bss        │  Global variables
               │  (~1 KB)             │
               ├──────────────────────┤
               │  FreeRTOS Heap       │  configTOTAL_HEAP_SIZE
               │  (16 KB)             │  Task stacks allocated here
               │  ┌────────────────┐  │  Queue buffers allocated here
               │  │ LED Task Stack │  │
               │  │ (512 bytes)    │  │
               │  ├────────────────┤  │
               │  │ UART Task Stack│  │
               │  │ (1024 bytes)   │  │
               │  ├────────────────┤  │
               │  │ Button Task    │  │
               │  │ Stack(1024 B)  │  │
               │  ├────────────────┤  │
               │  │ Idle Task Stack│  │
               │  │ (512 bytes)    │  │
               │  ├────────────────┤  │
               │  │ Free heap      │  │
               │  └────────────────┘  │
               ├──────────────────────┤
               │  Main Stack (MSP)    │  Used during startup
               │  (1 KB)             │  and interrupts
  0x2001_8000  └──────────────────────┘
```

---

## 17.5 Common Mistakes & Debugging

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Stack too small | Random crashes, corrupted data | Use `uxTaskGetStackHighWaterMark()` to check |
| Missing `volatile` on shared vars | Task sees stale values | Use queues/semaphores instead of bare variables |
| Calling blocking APIs from ISR | System freeze | Use `xSemaphoreGiveFromISR()` in ISRs |
| Priority inversion | Low-priority task blocks high | Use mutexes with priority inheritance |
| Heap too small | `xTaskCreate` returns NULL | Increase `configTOTAL_HEAP_SIZE` |
| vTaskDelay(0) | Task doesn't yield | Use `taskYIELD()` or `vTaskDelay(1)` |

### Debugging FreeRTOS in STM32CubeIDE

1. **View running tasks:** Use RTOS-aware debugging plugins
2. **Check stack usage:** Call `uxTaskGetStackHighWaterMark(NULL)` in each task
3. **Check heap remaining:** Call `xPortGetFreeHeapSize()`
4. **Enable `configCHECK_FOR_STACK_OVERFLOW`:** Detects overflow at runtime
5. **Use Tracealyzer:** Professional RTOS visualization tool (free for FreeRTOS)

---

## 17.6 Exercises

**Exercise 17.1:** Add a fourth task that reads the ADC every 100ms and sends the value to the UART task via the queue.

**Exercise 17.2:** Implement a mutex to protect the UART resource when multiple tasks try to print simultaneously.

**Exercise 17.3:** Create a task that uses `vTaskDelayUntil()` for precise 10ms periodic execution. Measure jitter with an oscilloscope.

**Exercise 17.4:** Implement a watchdog task: if any other task doesn't "check in" within 5 seconds, reset the system.

**Exercise 17.5:** Use FreeRTOS software timers to implement a state machine with timeout transitions.

---

## 17.7 Industry & Career Notes

- FreeRTOS is used in **billions** of devices — Amazon IoT, automotive, medical, industrial
- Knowing FreeRTOS is a **key differentiator** on firmware resumes
- Modern alternatives: Zephyr RTOS (Linux Foundation), ThreadX (Microsoft Azure RTOS)
- Safety-certified version: SafeRTOS (IEC 61508, ISO 26262)
- Interview tip: Be prepared to explain **priority inversion** and **deadlock**, and how to prevent them

---

## 17.8 Chapter Summary

```
  ┌────────────────────────────────────────────────────────────┐
  │         ★ MILESTONE: FreeRTOS APPLICATION COMPLETE ★       │
  │                                                            │
  │  ✓ RTOS allows multiple tasks to run "concurrently"        │
  │  ✓ Priority-based preemptive scheduling on Cortex-M        │
  │  ✓ Queues for thread-safe data passing between tasks       │
  │  ✓ Semaphores for event signaling (ISR → Task)             │
  │  ✓ Mutexes for protecting shared resources                 │
  │  ✓ vTaskDelay() is non-blocking (unlike software delay)    │
  │  ✓ Each task has its own stack allocated from FreeRTOS heap │
  │  ✓ FreeRTOS uses SysTick + PendSV for context switching    │
  │                                                            │
  │  OBJECTS CREATED:                                          │
  │    3 Tasks: LED, UART Logger, Button Handler               │
  │    1 Queue: Button → UART message passing                  │
  │    1 Semaphore: Button event signaling                     │
  └────────────────────────────────────────────────────────────┘
```

---

*Next: [Chapter 18 — Bootloader Design →](./Chapter_18_Bootloader.md)*
