# ECE319K Lab 8 — Distributed UART Data Acquisition System

## Overview
This project implements a full-duplex serial communication system 
between two MSPM0G3507 microcontrollers. Each board samples a slide 
potentiometer position using a 12-bit ADC, encodes the result as a 
4-byte ASCII message, and transmits it to the other board over UART 
at 2000 bps. The receiving board decodes the message and displays the 
position in centimeters on an ST7735 LCD.

The system runs continuously at 30 Hz with under 25ms end-to-end 
latency, verified using oscilloscope triple-toggle timing measurements.

## How It Works

### Transmit Side (Producer)
A TimerG12 hardware interrupt fires every 33.3ms (30 Hz). Inside the 
ISR, the ADC samples the slide potentiometer on PB18, the raw 12-bit 
result gets converted to a fixed-point distance (0.000 to 2.000 cm), 
and the result is encoded as a 4-byte message:
0x3C  →  start byte ('<')
0x3X  →  ones digit (ASCII)
0x3X  →  tenths digit (ASCII)
0x3X  →  hundredths digit (ASCII)

For example, 1.540 cm transmits as: `0x3C 0x31 0x35 0x34`

The four bytes get written to UART1 (PA8) using blind synchronization. 
This works without data loss because the UART hardware has a 4-element 
transmit FIFO, and the 20ms transmission window fits comfortably inside 
the 33.3ms sampling period.

### Receive Side (Consumer)
UART2 (PA22) receives the incoming bytes and triggers a receiver timeout 
interrupt after the line goes idle for 4 bit times (~2ms). The ISR 
immediately drains the hardware receive FIFO into a software circular 
FIFO queue, then returns. The main loop reads from the software FIFO, 
searches for the 0x3C start byte to stay synchronized, decodes the 
three digit bytes into a fixed-point position value, and updates the LCD.

This interrupt plus FIFO architecture keeps the two threads fully 
decoupled — the ISR never waits on main, and main never misses data 
even during slow LCD updates.

## Software FIFO Implementation
The software FIFO is a statically allocated circular buffer with 
two-index tracking (PutI and GetI). Wrap-around is handled with modulo 
arithmetic. The empty condition is PutI == GetI, and the full condition 
is (PutI+1) % SIZE == GetI, which intentionally wastes one slot to 
avoid needing a separate counter. The FIFO holds up to 19 bytes — 
enough to buffer multiple complete messages if the main loop is 
temporarily occupied with LCD output.

## Hardware Configuration
| Component | Detail |
|---|---|
| Microcontroller | TI MSPM0G3507 (80 MHz) |
| ADC | 12-bit, channel 5, PB18 |
| UART TX | UART1, PA8, 2000 bps |
| UART RX | UART2, PA22, 2000 bps |
| Display | ST7735 LCD via SPI |
| Baud rate divisor | IBRD=1250, FBRD=0 |

## Baud Rate Calculation
Bus clock:        80 MHz
UART clock:       80 MHz / 2 = 40 MHz
Baud clock base:  40 MHz / 16 = 2,500,000
Baud rate:        2,500,000 / 1250 = 2000 bps exactly
Bit time:         1 / 2000 = 500 µs
Frame time:       10 bits × 500 µs = 5 ms
Message time:     4 frames × 5 ms = 20 ms

## Interrupt Configuration
- **TimerG12**: period = 2,666,667 cycles → 80 MHz / 2,666,667 = 30 Hz
- **UART2 IRQ**: interrupt 14, priority 2, receiver timeout only (IMASK = 0x0001)
- **NVIC**: ICPR[0] and ISER[0] bit 14, priority bits 23-22 in IP[3]
- **IFLS**: 0x0422, receiver timeout select = 4 bit times of idle

## Performance Measurements
Verified with oscilloscope triple-toggle debugging on PB27 (TimerG12) 
and PB22 (UART2 ISR):

| Metric | Value |
|---|---|
| Sampling rate | 30 Hz |
| Message transmission time | 20 ms |
| Network latency (ADC to FIFO) | ~22 ms |
| TimerG12 ISR execution time | ~8 µs |
| UART2 ISR execution time | ~10 µs |
| Transmit task CPU overhead | ~0.024% |
| Receive task CPU overhead | ~0.030% |

## Files
| File | Description |
|---|---|
| `FIFO1.c` | Circular software FIFO implementation |
| `UART1.c` | Blind transmitter init and output function |
| `UART2.c` | Interrupt receiver init, ISR, and InChar |
| `Lab8Main.c` | TimerG12 ISR, main loop, LCD display logic |
| `ADC1.c` | 12-bit ADC init, sampling, and Convert function |

## What I Learned
Working on this project gave me a much deeper understanding of how 
producer-consumer systems work at the hardware level. The hardest part 
was getting the receiver timeout interrupt configured correctly and 
understanding why the hardware FIFO alone is not enough. On top of the Hardware FIFO, you also need the 
software FIFO to bridge the gap between the faster ISR and the slower 
main loop. Debugging with the oscilloscope triple toggle was also a 
new experience and made timing issues immediately visible in a way that 
printf debugging never could.
