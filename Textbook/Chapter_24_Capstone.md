# Chapter 24: Capstone Project — IoT Environmental Monitor

**Difficulty Level: ★★★★★ Expert**  
**Estimated Time: 4–6 weeks**

---

## 24.1 Project Overview

This capstone integrates nearly every concept from this textbook into a single, portfolio-worthy project: a **professional IoT environmental monitoring system**.

### System Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                   CAPSTONE PROJECT                          │
  │            IoT Environmental Monitor v1.0                   │
  │                                                             │
  │  ┌──────────┐   I2C    ┌──────────────┐                    │
  │  │ Temp/Hum │ ──────── │              │   UART             │
  │  │ Sensor   │          │   STM32F401  │ ──────── Terminal  │
  │  │(BME280)  │          │   Nucleo     │                    │
  │  └──────────┘          │              │   USB              │
  │                        │  FreeRTOS    │ ──────── PC App    │
  │  ┌──────────┐   ADC    │  3 Tasks     │                    │
  │  │ Light    │ ──────── │  Bootloader  │   Debug            │
  │  │ Sensor   │          │  Watchdog    │ ──────── SWD/GDB   │
  │  │(LDR)     │          │              │                    │
  │  └──────────┘          └──────────────┘                    │
  │                              │                              │
  │                         ┌────┴────┐                         │
  │                         │  LED    │ Status indicator        │
  │                         │  (PA5)  │ (blink patterns)        │
  │                         └─────────┘                         │
  └─────────────────────────────────────────────────────────────┘
  
  Features:
  ✓ Multi-sensor data acquisition (I2C + ADC)
  ✓ FreeRTOS with 3 tasks + queues + semaphores
  ✓ UART CLI for configuration and monitoring
  ✓ Non-volatile configuration storage in Flash
  ✓ Custom bootloader for firmware updates
  ✓ Watchdog timer for reliability
  ✓ Low-power sleep between measurements
  ✓ Professional code structure with HAL
  ✓ Makefile + automated build
  ✓ Python test suite + data visualization
```

---

## 24.2 Project Structure

```
  env-monitor/
  ├── bootloader/
  │   ├── src/
  │   │   ├── main.c                 # Bootloader entry (Ch18)
  │   │   └── flash_driver.c         # Flash programming
  │   ├── include/
  │   │   └── flash_driver.h
  │   ├── STM32F401_BL.ld           # Bootloader linker (0x08000000)
  │   └── Makefile
  │
  ├── application/
  │   ├── src/
  │   │   ├── main.c                 # Application entry
  │   │   ├── app_tasks.c            # FreeRTOS tasks (Ch17)
  │   │   ├── cli.c                  # UART command line (Ch7)
  │   │   └── config_store.c         # Flash config storage (Ch16)
  │   ├── drivers/
  │   │   ├── gpio_driver.c/h        # GPIO (Ch4)
  │   │   ├── uart_driver.c/h        # UART with DMA (Ch7, Ch12)
  │   │   ├── i2c_driver.c/h         # I2C (Ch11)
  │   │   ├── adc_driver.c/h         # ADC (Ch9)
  │   │   ├── timer_driver.c/h       # SysTick + Timers (Ch5, Ch8)
  │   │   └── bme280.c/h             # BME280 sensor driver
  │   ├── include/
  │   │   ├── app_config.h            # System configuration
  │   │   ├── FreeRTOSConfig.h        # RTOS config (Ch17)
  │   │   └── error_handler.h         # Fault handling (Ch19)
  │   ├── startup/
  │   │   ├── startup_stm32f401.s     # Startup assembly (Ch21)
  │   │   └── system_init.c           # Clock setup (Ch5)
  │   ├── STM32F401_APP.ld           # App linker (0x08010000)
  │   ├── Makefile                    # Build system (Ch15)
  │   └── CMakeLists.txt
  │
  ├── tools/
  │   ├── upload_firmware.py          # Bootloader client (Ch18, Ch22)
  │   ├── serial_monitor.py          # Data logging (Ch22)
  │   ├── live_plot.py               # Real-time visualization (Ch22)
  │   └── test_firmware.py           # HIL tests (Ch22)
  │
  └── README.md
```

---

## 24.3 Application Main

```c
/******************************************************************************
 * File:    main.c
 * Brief:   Capstone application entry point
 *
 * Initializes hardware, creates FreeRTOS tasks, starts scheduler
 *****************************************************************************/

#include <stdint.h>
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include "semphr.h"

#include "gpio_driver.h"
#include "uart_driver.h"
#include "i2c_driver.h"
#include "adc_driver.h"
#include "timer_driver.h"
#include "bme280.h"
#include "cli.h"
#include "config_store.h"
#include "error_handler.h"
#include "app_config.h"

/* ========================================================================= */
/*                       GLOBAL RESOURCES                                    */
/* ========================================================================= */

/* Queue for sensor data → logger task */
QueueHandle_t sensor_data_queue;

/* Mutex for I2C bus (shared between tasks) */
SemaphoreHandle_t i2c_mutex;

/* Sensor data structure */
typedef struct {
    float    temperature;   /* Celsius */
    float    humidity;      /* % RH */
    float    pressure;      /* hPa */
    uint16_t light_raw;     /* ADC counts (0-4095) */
    uint32_t timestamp_ms;  /* Tick count at measurement */
} SensorData_t;

/* System configuration (loaded from Flash) */
AppConfig_t app_config;

/* ========================================================================= */
/*                     TASK: Sensor Acquisition                              */
/* ========================================================================= */

void task_sensor(void *params)
{
    (void)params;
    SensorData_t data;
    BME280_Data_t bme_data;

    uart_print("[SENSOR] Task started\r\n");

    /* Initialize BME280 */
    if (xSemaphoreTake(i2c_mutex, pdMS_TO_TICKS(100)))
    {
        bme280_init();
        xSemaphoreGive(i2c_mutex);
    }

    while (1)
    {
        /* Read BME280 (temperature, humidity, pressure) via I2C */
        if (xSemaphoreTake(i2c_mutex, pdMS_TO_TICKS(100)))
        {
            bme280_read(&bme_data);
            xSemaphoreGive(i2c_mutex);

            data.temperature = bme_data.temperature;
            data.humidity    = bme_data.humidity;
            data.pressure    = bme_data.pressure;
        }

        /* Read light sensor via ADC */
        data.light_raw   = adc_read();
        data.timestamp_ms = xTaskGetTickCount();

        /* Send data to logger task via queue */
        xQueueSend(sensor_data_queue, &data, pdMS_TO_TICKS(10));

        /* Blink LED to indicate measurement */
        gpio_toggle(GPIOA, 5);

        /* Sleep for configured interval */
        vTaskDelay(pdMS_TO_TICKS(app_config.sample_interval_ms));
    }
}

/* ========================================================================= */
/*                     TASK: Data Logger                                     */
/* ========================================================================= */

void task_logger(void *params)
{
    (void)params;
    SensorData_t data;
    char buf[128];

    uart_print("[LOGGER] Task started\r\n");

    while (1)
    {
        if (xQueueReceive(sensor_data_queue, &data, portMAX_DELAY))
        {
            /* Format and send via UART */
            int len = snprintf(buf, sizeof(buf),
                "[%lu] T=%.1f C, H=%.1f %%, P=%.1f hPa, L=%u\r\n",
                data.timestamp_ms,
                (double)data.temperature,
                (double)data.humidity,
                (double)data.pressure,
                data.light_raw);

            uart_send_string(buf);

            /* Check thresholds and alert */
            if (data.temperature > app_config.temp_alert_high)
            {
                uart_print("!!! ALERT: Temperature HIGH !!!\r\n");
            }
            if (data.humidity > app_config.humidity_alert_high)
            {
                uart_print("!!! ALERT: Humidity HIGH !!!\r\n");
            }
        }
    }
}

/* ========================================================================= */
/*                     TASK: CLI (Command Line Interface)                    */
/* ========================================================================= */

void task_cli(void *params)
{
    (void)params;

    uart_print("[CLI] Task started\r\n");
    uart_print("Type 'help' for commands\r\n> ");

    while (1)
    {
        cli_process();  /* Handle incoming UART commands */
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

/* ========================================================================= */
/*                              MAIN                                         */
/* ========================================================================= */

int main(void)
{
    /* Relocate vector table (we're running from bootloader offset) */
    SCB->VTOR = 0x08010000UL;

    /* Initialize hardware */
    clock_init_84mhz();      /* Ch5: PLL to 84 MHz */
    systick_init();           /* Ch5: 1 ms tick */
    gpio_init_all();          /* Ch4: LED, debug pins */
    uart_init(115200);        /* Ch7: UART2 for CLI/logging */
    i2c1_init();              /* Ch11: I2C for BME280 */
    adc_init();               /* Ch9: ADC for light sensor */

    /* Load configuration from Flash */
    config_load(&app_config);

    /* Print boot banner */
    uart_print("\r\n");
    uart_print("╔═══════════════════════════════════════╗\r\n");
    uart_print("║   IoT Environmental Monitor v1.0      ║\r\n");
    uart_print("║   STM32F401RE — Capstone Project      ║\r\n");
    uart_print("╚═══════════════════════════════════════╝\r\n");

    /* Enable Watchdog (8 second timeout) */
    /* iwdg_init(8000); */

    /* Create RTOS resources */
    sensor_data_queue = xQueueCreate(16, sizeof(SensorData_t));
    i2c_mutex = xSemaphoreCreateMutex();

    configASSERT(sensor_data_queue != NULL);
    configASSERT(i2c_mutex != NULL);

    /* Create tasks */
    xTaskCreate(task_sensor, "Sensor", 256, NULL, 2, NULL);
    xTaskCreate(task_logger, "Logger", 256, NULL, 1, NULL);
    xTaskCreate(task_cli,    "CLI",    256, NULL, 1, NULL);

    /* Start scheduler — never returns */
    vTaskStartScheduler();

    /* Should never reach here */
    error_handler("Scheduler failed");
    while (1);
}
```

---

## 24.4 CLI Module

```c
/******************************************************************************
 * File:    cli.c
 * Brief:   UART command line interface
 *
 * Commands:
 *   help                  Show available commands
 *   status                Show system status
 *   config show           Show current configuration
 *   config set <key> <val> Set configuration parameter
 *   config save           Save config to Flash
 *   reset                 Software reset
 *   version               Show firmware version
 *****************************************************************************/

#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include "cli.h"
#include "uart_driver.h"
#include "config_store.h"
#include "app_config.h"

#define CLI_MAX_CMD_LEN  64
#define CLI_MAX_ARGS      4

extern AppConfig_t app_config;

static char cmd_buffer[CLI_MAX_CMD_LEN];
static uint8_t cmd_pos = 0;

/* Command table */
typedef struct {
    const char *name;
    const char *help;
    void (*handler)(int argc, char *argv[]);
} CliCommand_t;

static void cmd_help(int argc, char *argv[]);
static void cmd_status(int argc, char *argv[]);
static void cmd_config(int argc, char *argv[]);
static void cmd_version(int argc, char *argv[]);
static void cmd_reset(int argc, char *argv[]);

static const CliCommand_t commands[] = {
    {"help",    "Show this help message",  cmd_help},
    {"status",  "Show system status",      cmd_status},
    {"config",  "Configuration management", cmd_config},
    {"version", "Show firmware version",   cmd_version},
    {"reset",   "Software reset",          cmd_reset},
};

#define NUM_COMMANDS (sizeof(commands) / sizeof(commands[0]))

static void cmd_help(int argc, char *argv[])
{
    (void)argc; (void)argv;
    uart_print("Available commands:\r\n");
    for (uint32_t i = 0; i < NUM_COMMANDS; i++)
    {
        char buf[64];
        snprintf(buf, sizeof(buf), "  %-10s %s\r\n",
                 commands[i].name, commands[i].help);
        uart_print(buf);
    }
}

static void cmd_status(int argc, char *argv[])
{
    (void)argc; (void)argv;
    char buf[128];
    snprintf(buf, sizeof(buf),
             "Uptime: %lu ms\r\n"
             "Free heap: %u bytes\r\n"
             "Tasks: %lu\r\n",
             (unsigned long)xTaskGetTickCount(),
             (unsigned)xPortGetFreeHeapSize(),
             (unsigned long)uxTaskGetNumberOfTasks());
    uart_print(buf);
}

static void cmd_config(int argc, char *argv[])
{
    if (argc < 2) {
        uart_print("Usage: config show|set|save\r\n");
        return;
    }
    
    if (strcmp(argv[1], "show") == 0) {
        char buf[128];
        snprintf(buf, sizeof(buf),
                 "Sample interval: %lu ms\r\n"
                 "Temp alert high: %.1f C\r\n"
                 "Humidity alert:  %.1f %%\r\n",
                 app_config.sample_interval_ms,
                 (double)app_config.temp_alert_high,
                 (double)app_config.humidity_alert_high);
        uart_print(buf);
    }
    else if (strcmp(argv[1], "set") == 0 && argc >= 4) {
        if (strcmp(argv[2], "interval") == 0)
            app_config.sample_interval_ms = (uint32_t)atoi(argv[3]);
        else if (strcmp(argv[2], "temp_high") == 0)
            app_config.temp_alert_high = (float)atof(argv[3]);
        else if (strcmp(argv[2], "hum_high") == 0)
            app_config.humidity_alert_high = (float)atof(argv[3]);
        else
            uart_print("Unknown key\r\n");
    }
    else if (strcmp(argv[1], "save") == 0) {
        config_save(&app_config);
        uart_print("Config saved to Flash\r\n");
    }
}

static void cmd_version(int argc, char *argv[])
{
    (void)argc; (void)argv;
    uart_print("Firmware: IoT-EnvMon v1.0.0\r\n");
    uart_print("Build: " __DATE__ " " __TIME__ "\r\n");
    uart_print("MCU: STM32F401RE\r\n");
}

static void cmd_reset(int argc, char *argv[])
{
    (void)argc; (void)argv;
    uart_print("Resetting...\r\n");
    /* Wait for UART to flush */
    for (volatile uint32_t i = 0; i < 160000; i++);
    NVIC_SystemReset();
}

/* Parse and execute a command line */
static void execute_command(const char *line)
{
    char buf[CLI_MAX_CMD_LEN];
    strncpy(buf, line, sizeof(buf) - 1);
    buf[sizeof(buf) - 1] = '\0';

    /* Tokenize */
    char *argv[CLI_MAX_ARGS];
    int argc = 0;
    char *token = strtok(buf, " ");
    while (token && argc < CLI_MAX_ARGS)
    {
        argv[argc++] = token;
        token = strtok(NULL, " ");
    }

    if (argc == 0) return;

    /* Find and execute command */
    for (uint32_t i = 0; i < NUM_COMMANDS; i++)
    {
        if (strcmp(argv[0], commands[i].name) == 0)
        {
            commands[i].handler(argc, argv);
            return;
        }
    }

    uart_print("Unknown command. Type 'help'\r\n");
}

/* Called periodically from CLI task */
void cli_process(void)
{
    uint8_t c;
    if (uart_receive_char(&c))
    {
        if (c == '\r' || c == '\n')
        {
            uart_print("\r\n");
            if (cmd_pos > 0)
            {
                cmd_buffer[cmd_pos] = '\0';
                execute_command(cmd_buffer);
                cmd_pos = 0;
            }
            uart_print("> ");
        }
        else if (c == 0x7F || c == '\b')  /* Backspace */
        {
            if (cmd_pos > 0)
            {
                cmd_pos--;
                uart_print("\b \b");
            }
        }
        else if (cmd_pos < CLI_MAX_CMD_LEN - 1)
        {
            cmd_buffer[cmd_pos++] = (char)c;
            uart_send_char(c);  /* Echo */
        }
    }
}
```

---

## 24.5 Configuration Storage

```c
/******************************************************************************
 * File:    config_store.c
 * Brief:   Store/load configuration in last Flash sector
 *
 * Uses the last sector of Flash to store configuration.
 * Sector 7 (128 KB) is used — overkill, but sectors are the smallest
 * erasable unit on STM32F401.
 *****************************************************************************/

#include "config_store.h"
#include <string.h>

#define CONFIG_FLASH_ADDR   0x08060000UL  /* Sector 7 start */
#define CONFIG_MAGIC        0xC0FFEE42UL

typedef struct {
    uint32_t    magic;
    AppConfig_t config;
    uint32_t    crc;      /* Simple checksum */
} StoredConfig_t;

/* Default configuration */
static const AppConfig_t default_config = {
    .sample_interval_ms  = 1000,
    .temp_alert_high     = 35.0f,
    .humidity_alert_high = 80.0f,
};

static uint32_t calc_crc(const AppConfig_t *cfg)
{
    const uint8_t *data = (const uint8_t *)cfg;
    uint32_t crc = 0;
    for (uint32_t i = 0; i < sizeof(AppConfig_t); i++)
    {
        crc += data[i];
        crc = (crc << 1) | (crc >> 31);  /* Simple rotate + accumulate */
    }
    return crc;
}

void config_load(AppConfig_t *cfg)
{
    const StoredConfig_t *stored = (const StoredConfig_t *)CONFIG_FLASH_ADDR;

    if (stored->magic == CONFIG_MAGIC &&
        stored->crc == calc_crc(&stored->config))
    {
        memcpy(cfg, &stored->config, sizeof(AppConfig_t));
        uart_print("[CONFIG] Loaded from Flash\r\n");
    }
    else
    {
        memcpy(cfg, &default_config, sizeof(AppConfig_t));
        uart_print("[CONFIG] Using defaults\r\n");
    }
}

void config_save(const AppConfig_t *cfg)
{
    StoredConfig_t new_config;
    new_config.magic = CONFIG_MAGIC;
    memcpy(&new_config.config, cfg, sizeof(AppConfig_t));
    new_config.crc = calc_crc(cfg);

    /* Erase sector 7 */
    flash_unlock();
    flash_erase_sector(7);

    /* Write config byte-by-byte */
    const uint8_t *src = (const uint8_t *)&new_config;
    for (uint32_t i = 0; i < sizeof(StoredConfig_t); i++)
    {
        flash_program_byte(CONFIG_FLASH_ADDR + i, src[i]);
    }

    flash_lock();
}
```

---

## 24.6 Chapters Used in This Project

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Chapter Cross-Reference                                     │
  ├──────────────────────────────────────────────────────────────┤
  │  Ch 1:  LED Blink         → Status LED on PA5                │
  │  Ch 2:  Embedded C        → Coding style, volatile, structs  │
  │  Ch 3:  Dev Environment   → Toolchain, debugging setup       │
  │  Ch 4:  GPIO              → LED, button, debug pins          │
  │  Ch 5:  Clock System      → 84 MHz PLL, SysTick              │
  │  Ch 6:  Interrupts        → EXTI button, NVIC priorities     │
  │  Ch 7:  UART              → CLI, data logging, printf        │
  │  Ch 8:  Timers            → Measurement timing, LED patterns  │
  │  Ch 9:  ADC               → Light sensor reading             │
  │  Ch 10: SPI               → (Optional: SPI sensor variant)   │
  │  Ch 11: I2C               → BME280 sensor communication      │
  │  Ch 12: DMA               → (Optional: UART DMA for logging) │
  │  Ch 13: CAN Bus           → (Optional: multi-node network)   │
  │  Ch 15: Build Systems     → Makefile, automated build        │
  │  Ch 16: Memory/Linker     → App linker script, config Flash  │
  │  Ch 17: FreeRTOS          → 3 tasks, queues, semaphores      │
  │  Ch 18: Bootloader        → UART firmware update             │
  │  Ch 19: Debugging         → Hard Fault handler, assertions   │
  │  Ch 22: Python            → Upload, test, plot tools         │
  └──────────────────────────────────────────────────────────────┘
```

---

## 24.7 Testing Your Capstone

### Acceptance Test Checklist

```
  ┌──┬────────────────────────────────────────────┬──────┐
  │# │ Test                                       │ Pass │
  ├──┼────────────────────────────────────────────┼──────┤
  │1 │ Bootloader boots to app within 3 seconds   │ [ ]  │
  │2 │ Firmware update via UART works              │ [ ]  │
  │3 │ Temperature reading within ±1°C of known   │ [ ]  │
  │4 │ Humidity reading within ±3% of known        │ [ ]  │
  │5 │ Light sensor responds to covering/uncovering│ [ ]  │
  │6 │ UART CLI responds to all commands           │ [ ]  │
  │7 │ Config save/load survives power cycle       │ [ ]  │
  │8 │ LED blinks during normal operation          │ [ ]  │
  │9 │ Temperature alert triggers at threshold     │ [ ]  │
  │10│ System runs 24 hours without crash          │ [ ]  │
  │11│ Python test suite passes all tests          │ [ ]  │
  │12│ Python live plot shows real-time data        │ [ ]  │
  │13│ Hard Fault handler catches forced faults     │ [ ]  │
  │14│ Code compiles with -Wall -Wextra -Werror    │ [ ]  │
  │15│ Flash usage below 50%                       │ [ ]  │
  └──┴────────────────────────────────────────────┴──────┘
```

---

## 24.8 Making It Portfolio-Worthy

### What Hiring Managers Look For

```
  Your GitHub Repository Should Have:
  
  ✓ Clear README with:
    - Project description and photo
    - Hardware diagram / schematic
    - How to build and flash
    - Feature list with completion status
  
  ✓ Clean code:
    - Consistent style (MISRA-inspired)
    - Function comments (doxygen-style)
    - No dead code or commented-out blocks
    - Clear separation of drivers and application
  
  ✓ Professional practices:
    - Makefile that works out of the box
    - .gitignore for build artifacts
    - Version tags (v1.0, v1.1, etc.)
    - Meaningful commit messages
  
  ✓ Documentation:
    - Block diagram of the system
    - Pin assignment table
    - CLI command reference
    - Known limitations
  
  ✓ Testing:
    - Python test scripts that run automatically
    - Test results log
  
  BONUS (Stand Out):
    - CI/CD with GitHub Actions (build on push)
    - PCB design (KiCad) for a custom board
    - Power analysis (oscilloscope current measurement)
    - OTA update over WiFi (add ESP32 as WiFi bridge)
```

---

## 24.9 Extensions and Next Steps

```
  Level Up Your Project:
  
  Extension 1: Add an OLED Display (SSD1306, I2C)
    → Display sensor values on a 128x64 OLED
    → New task: Display refresh at 10 Hz
    → Chapter bonus: framebuffer, fonts, I2C shared access
  
  Extension 2: SD Card Data Logging (SPI)
    → Log CSV data to microSD card
    → Use FatFS file system library
    → Circular logging with file rotation
  
  Extension 3: WiFi Bridge (ESP32 via UART)
    → Connect ESP32 to STM32 UART
    → ESP32 posts data to cloud (MQTT / HTTP)
    → Web dashboard for remote monitoring
  
  Extension 4: Custom PCB
    → Design PCB in KiCad
    → Order from JLCPCB / PCBWay
    → Solder and test your own board
  
  Extension 5: Battery + Solar
    → Li-Po battery with charging IC
    → Solar panel input
    → Ultra-low-power sleep modes
    → Power budget analysis
```

---

## 24.10 Chapter Summary

```
  ╔═══════════════════════════════════════════════════════════╗
  ║             ★ CAPSTONE PROJECT COMPLETE ★                 ║
  ║                                                           ║
  ║  You have built a complete embedded system:               ║
  ║                                                           ║
  ║  ✓ Custom bootloader with UART update                     ║
  ║  ✓ Multi-sensor acquisition (I2C + ADC)                   ║
  ║  ✓ FreeRTOS multitasking (3 tasks + IPC)                 ║
  ║  ✓ UART command-line interface                            ║
  ║  ✓ Non-volatile configuration storage                     ║
  ║  ✓ Professional code structure                            ║
  ║  ✓ Python toolchain (test, plot, upload)                  ║
  ║  ✓ Automated build system                                 ║
  ║                                                           ║
  ║  This project demonstrates:                               ║
  ║    Peripheral drivers, RTOS, memory management,           ║
  ║    error handling, build systems, and testing —            ║
  ║    everything a firmware engineer needs.                   ║
  ║                                                           ║
  ║  PUT THIS ON YOUR RESUME AND GITHUB.                      ║
  ╚═══════════════════════════════════════════════════════════╝
```

---

*Next: [Appendices — Reference Tables, Glossary & Resources →](./Appendices.md)*
