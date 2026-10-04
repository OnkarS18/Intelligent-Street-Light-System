# Intelligent Street Lighting System (STM32F410RB + BH1750)

An embedded system that measures ambient light and switches a street lamp automatically. It classifies the surroundings as **Day**, **Dusk** or **Night**, drives the lamp, and reports every reading over UART. It was built for the Embedded Edge AI lab and is now used as a QA and project-management case study in the Project Management course (MIT Academy of Engineering, E&TC).

![status](https://img.shields.io/badge/platform-STM32F410RB-blue) ![ide](https://img.shields.io/badge/IDE-STM32CubeIDE%201.17.0-green) ![lang](https://img.shields.io/badge/language-C%20(HAL)-lightgrey)

## Features

- BH1750 (GY-30) light sensor read over I2C1 at 100 kHz
- TIM5 interrupt at 1 Hz; the ISR only sets a flag and the main loop does the I2C work
- Three-state classification with 10% hysteresis, so the lamp does not flicker near the limits
- Fail-safe behaviour: if the sensor fails, the lamp turns ON and the monitor shows `SENSOR FAULT`
- Bounded I2C timeout (50 ms) and automatic bus recovery after 3 failed reads
- Live status over USART2 at 115200 baud through the ST-Link virtual COM port

## Hardware

| Item | Details |
|---|---|
| Board | STM32 Nucleo-F410RB (STM32F410RBT6, Cortex-M4, 80 MHz) |
| Sensor | GY-30 module with BH1750, I2C address `0x23` (ADDR tied to GND) |
| Lamp | Onboard LED LD2 on PA5 (stands in for the street light) |
| Cable | USB Type-A to Mini-B (power, programming, UART) |
| Wires | 4 male-to-female jumpers |

### Pin connections

| BH1750 pin | STM32 pin | Signal |
|---|---|---|
| VCC | 3.3V | Power |
| GND | GND | Ground |
| SDA | PB9 | I2C1_SDA |
| SCL | PB8 | I2C1_SCL |
| ADDR | GND | Address 0x23 |
| - | PA5 (LD2) | Street light output |
| - | PA2 / PA3 | USART2 TX / RX (ST-Link VCP) |

Circuit diagram: [`docs/circuit_diagram.jpg`](docs/circuit_diagram.jpg)

## How it works

1. At start-up the clock runs at 80 MHz (HSI through the PLL) and GPIO, USART2, TIM5 and I2C1 are initialised.
2. The BH1750 is powered on in continuous high-resolution mode (1 lux resolution).
3. TIM5 fires every second (80 MHz / 10000 / 8000 = 1 Hz) and sets `timer_flag`.
4. The main loop reads the sensor and converts the raw value with `lux = raw / 1.2`.
5. `BH1750_Classify()` picks the state, using the previous state for hysteresis.
6. `StreetLight_Control()` sets the lamp and the result is printed over UART.

### Behaviour

| State | Rule (with hysteresis) | Lamp |
|---|---|---|
| Day | lux >= 1000 (leaves Day below 900, enters Day at 1100 or more) | OFF |
| Dusk | between about 100 and 1000 lux | ON |
| Night | lux < 100 (leaves Night at 110 or more, enters Night below 90) | ON |
| Fault | I2C read fails | ON (fail-safe) |

Notes:
- The largest reading the code can produce is about 54,612 lux (65535 / 1.2), not 65535 lux.
- Dusk currently switches the lamp fully ON. There is no dimming, because PA5 is a plain ON/OFF output. PWM dimming is tracked as a future enhancement.
- The blue button B1 is configured as an interrupt input but is reserved and unused.

### Sample serial output

```text
=== Intelligent Street Lighting System ===
STM32F410RB + BH1750 | UART 115200 baud

[OK] BH1750 initialized (Continuous High-Res mode)
[OK] TIM5 started - 1s periodic interrupt active

Lux:  112.50  |  Condition: DUSK  [Light ON ]
Lux: 1250.83  |  Condition: DAY   [Light OFF]
Lux:    8.33  |  Condition: NIGHT [Light ON ]
Lux:   -     |  Condition: SENSOR FAULT [Light ON ]
```

## Project structure

```text
intelligent-street-light-stm32/
├── Core/
│   ├── Inc/            main.h, bh1750.h
│   └── Src/            main.c, bh1750.c
├── docs/               circuit_diagram.jpg, QA_checklist.md, test_log.md
├── .github/            ISSUE_TEMPLATE/bug_report.md, pull_request_template.md
├── StreetLight.ioc     STM32CubeMX configuration
├── .gitignore
└── README.md
```

## Build and run

1. Install **STM32CubeIDE v1.17.0** (or newer).
2. Clone the repository:
```bash
   git clone https://github.com/<OnkarS18>/Intelligent-Street-Light-System.git
```
3. In CubeIDE choose **File > Import > Existing Projects into Workspace** and select the cloned folder.
4. In project properties enable **Use float with printf from newlib-nano** (`-u _printf_float`), otherwise lux values will not print.
5. Connect the Nucleo board by USB, then **Build** and **Run/Debug**.
6. Open a serial terminal on the ST-Link virtual COM port at **115200 baud, 8N1**.
7. Cover the sensor for Night, use room light for Dusk, and shine a torch for Day.

## Peripheral configuration

| Peripheral | Setting |
|---|---|
| I2C1 | Standard mode 100 kHz, PB8 (SCL), PB9 (SDA), AF4 |
| USART2 | 115200, 8 data bits, no parity, 1 stop bit |
| TIM5 | Prescaler 9999, period 7999, update interrupt enabled |
| PA5 | GPIO output push-pull, no pull, low speed, label `STREET_LED` |
| Clock | HSI 16 MHz, PLLM 8, PLLN 80, PLLP 2, SYSCLK 80 MHz, APB1 /2 |

## Test results (bench)

| Condition | Observed lux |
|---|---|
| Night (sensor covered) | 0.83 to 32.50 |
| Dusk (normal indoor light) | 97.50 to 166.67 |
| Day (torch on sensor) | 1035.83 to 2618.33 |

More cases (fault injection, boundary sequence) are logged in [`docs/test_log.md`](docs/test_log.md).

## QA and collaboration workflow

This repository is also a GitHub-based QA exercise.

- Every problem is raised as an **Issue** using the bug template, with severity and label.
- Every fix is made on its own branch (`fix/issue-<n>-<short-name>`) and merged through a **Pull Request**.
- Commit messages end with `Fixes #<n>` so the issue closes automatically.
- Progress is tracked on the **Projects board** and four **Milestones** (M1 to M4).

| Issue | Problem | Severity |
|---|---|---|
| #1 | Sensor fault reported as NIGHT; failed start-up leaves lamp OFF | High |
| #2 | I2C calls used `HAL_MAX_DELAY` and could block forever | High |
| #3 | No hysteresis around the lux thresholds | Medium |
| #4 | Dusk printed "Light DIM" but the lamp was fully ON | Medium |
| #5 | Comma-operator in switch, dead code | Low |
| #6 | Report and code disagreed on range and thresholds | Low |

## Roadmap

- [ ] PWM dimming at dusk (confirm a timer channel on the pinout first)
- [ ] Independent watchdog (IWDG) for full lock-up recovery
- [ ] Manual override using button B1
- [ ] Telemetry (for example LoRa) for multi-pole deployments

## Author

**Onkar Sorde**, PRN 202301070054
BTech E&TC, Semester VII, Division C, MIT Academy of Engineering, Pune
Course: Project Management (2307476T), Course Teacher: Dr. Ashish Mulajkar

## License

Educational project. Generated STM32 HAL and CubeMX files remain under STMicroelectronics' licence terms.
