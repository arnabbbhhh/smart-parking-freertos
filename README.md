# Smart Parking System — STM32F446RE + FreeRTOS

> **Course: Advanced Embedded Systems and Designs (AESD) | LNMIIT Jaipur**  
> **Instructor: Dr. Deepak Nair | Author: Arnab Harsh (22UEC024)**

---

## Overview

A **real-time smart parking system** built on the STM32F446RE microcontroller using FreeRTOS. The system monitors 4 parking slots using IR sensors and automatically controls entry/exit gates using PWM-driven servo motors. All system activity is logged over UART to a serial terminal (PuTTY / STM32CubeIDE Serial Monitor).

---

## Hardware

| Component | Pin | Function |
|---|---|---|
| IR Sensor — Slot 1 | PA0 | Slot occupancy detection |
| IR Sensor — Slot 2 | PA1 | Slot occupancy detection |
| IR Sensor — Slot 3 | PA6 | Slot occupancy detection |
| IR Sensor — Slot 4 | PA7 | Slot occupancy detection |
| SG90 Servo — Entry Gate | PB6 (TIM4 CH1) | Entry gate control |
| SG90 Servo — Exit Gate | PB7 (TIM4 CH2) | Exit gate control |
| UART2 | PA2/PA3 | Serial logging at 115200 baud |

### IR Sensor Logic
- `0` → Car Detected (IR beam blocked)
- `1` → Slot Empty (IR beam free)

### Servo PWM Values
| Pulse Width | Gate State |
|---|---|
| 1000 µs | Gate Closed |
| 1500 µs | Mid / Neutral |
| 2000 µs | Gate Open |

> Timer 4 configured for 20 ms period (50 Hz) — standard servo control frequency

---

## FreeRTOS Architecture

```
┌─────────────────┐     parkingQueue     ┌─────────────────┐
│  IRSensorTask   │ ──────────────────►  │   LoggerTask    │
│  (100ms poll)   │                      │  (UART output)  │
└─────────────────┘                      └─────────────────┘
         │                                        ▲
         ▼                                        │
┌─────────────────┐     servoQueue       ┌─────────────────┐
│ DefaultTask     │ ──────────────────►  │  GateServoTask  │
│ (gate logic)    │                      │  (PWM control)  │
└─────────────────┘                      └─────────────────┘

Shared resource protection: uartMutex (prevents concurrent UART writes)
```

### Task Descriptions

| Task | Priority | Period | Function |
|---|---|---|---|
| `IRSensorTask` | Normal | 100 ms | Reads 4 IR sensors, packs into 1 byte, sends to `parkingQueue` |
| `GateServoTask` | Normal | Event-driven | Receives PWM commands from `servoQueue`, drives servo motors |
| `LoggerTask` | Normal | Event-driven | Receives sensor data, decodes, prints status over UART |
| `DefaultTask` | Normal | 200 ms | Core gate logic — opens/closes gates based on slot availability |

### Sensor Data Packing
Four sensor readings are packed into a single 8-bit byte for efficient queue transfer:

```
Bit 3    Bit 2    Bit 1    Bit 0
 PA7      PA6      PA1      PA0
Slot 4   Slot 3   Slot 2   Slot 1
```

### Gate Control Logic
| Condition | Entry Gate | Exit Gate |
|---|---|---|
| Slots available | Open (2000 µs) | Open (2000 µs) |
| All slots full | Closed (1000 µs) | Open (2000 µs) |

---

## UART Output Sample

```
===== PARKING STATUS =====
Slot 1: OCCUPIED
Slot 2: EMPTY
Slot 3: OCCUPIED
Slot 4: EMPTY
Occupied: 2 | Available: 2
Sensor Map: 0101
Entry Gate: OPEN
Exit Gate:  OPEN
==========================
```

---

## Key FreeRTOS Concepts Demonstrated

- **Multitasking** — 4 concurrent tasks with defined priorities
- **Queues** — `parkingQueue` and `servoQueue` for inter-task data transfer
- **Mutex** — `uartMutex` prevents overlapping UART writes from multiple tasks
- **Periodic Tasks** — `vTaskDelayUntil()` for deterministic 100ms sensor polling
- **Event-Driven Tasks** — Gate and logger tasks block on queue until data arrives

---

## How to Run

1. Open **STM32CubeIDE**
2. Create new project for **STM32F446RE Nucleo**
3. Enable **FreeRTOS (CMSIS-RTOS v2)** in `.ioc` config
4. Configure **TIM4** for PWM on CH1 (PB6) and CH2 (PB7)
5. Configure **USART2** at 115200 baud
6. Add source files from `Core/Src/`
7. Build and flash to board
8. Open PuTTY or STM32CubeIDE Serial Monitor at **115200 baud**

---

## Tools Used

- **STM32CubeIDE** with HAL + CMSIS-RTOS v2
- **FreeRTOS** (integrated via STM32Cube middleware)
- **STM32F446RE Nucleo Board**
- **PuTTY** for UART serial monitoring

---

## Future Enhancements

- LCD / OLED display for on-site slot status
- ESP8266 / BLE for wireless IoT dashboard
- ANPR-based automatic vehicle identification
- Cloud-connected monitoring (AWS IoT / MQTT)
- Ultrasonic sensors for more reliable detection
- Solar-powered parking nodes
