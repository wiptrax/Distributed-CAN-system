# STM32 CAN Bus Distributed System

A distributed system using two STM32F446RE microcontrollers communicating over CAN bus. Node A broadcasts sensor data, Node B monitors thresholds and sends commands back. Built with FreeRTOS and implements a simple ACK protocol.

## Demo


## What It Does

**Node A** simulates an ECU with sensors - broadcasts RPM and temperature data every 500ms over CAN at 500 kbit/s.

**Node B** listens to the data and sends commands back when thresholds are exceeded:
- RPM > 5000 → sends warning command
- Temperature > 80°C → sends warning command

Commands require acknowledgment. If Node A doesn't ACK within 200ms, Node B logs an error.

The Python script plots everything in real-time.

## Hardware

- 2x NUCLEO-F446RE boards ($15 each)
- 2x TJA1050 CAN transceiver modules ($3 each)
- 2x 120Ω resistors (for bus termination)
- Breadboard and jumper wires
- 2x Mini-USB cables

## Protocol

### Message Types

| CAN ID | Type | Purpose |
|--------|------|---------|
| 0x100 | RPM | Engine speed (0-6000) |
| 0x101 | TEMP | Temperature in Celsius |
| 0x102 | Heartbeat | Keep-alive signal |
| 0x200 | Command | Control message from Node B |
| 0x201 | ACK | Acknowledgment from Node A |

### Why Commands Need ACK

Data messages (RPM/TEMP/Heartbeat) are sent continuously every 500ms. If one gets lost, the next one arrives shortly after. No big deal.

Commands are different - they're only sent once when a condition triggers. If that message gets lost, Node A never knows it needs to take action. That's why commands require acknowledgment with a timeout.

This is how real automotive protocols like J1939 and CANopen work.

## Wiring

### TJA1050 to NUCLEO (both boards identical)

| TJA1050 | NUCLEO Pin |
|---------|------------|
| TxD | PA12 (CN10-12) |
| RxD | PA11 (CN10-14) |
| VCC | 5V (CN6-5) |
| GND | GND (CN6-6) |

### CAN Bus Between Modules

Connect CANH to CANH and CANL to CANL between the two TJA1050 modules.

Add a 120Ω resistor between CANH and CANL on each module. This termination is required - without it the bus won't work reliably.

## Building and Flashing

1. Open NodeA project in STM32CubeIDE
2. Build (Ctrl+B)
3. Connect first NUCLEO board
4. Flash (F11)
5. Repeat for NodeB with second board

## Viewing Output

On Mac/Linux:
```bash
ls /dev/tty.*  # find your ports
screen /dev/tty.usbmodem1203 115200  # Node A
screen /dev/tty.usbmodem1303 115200  # Node B
```

Exit screen: Ctrl+A, then K, then Y

## Running the Dashboard
```bash
pip3 install pyserial matplotlib
python3 python/dashboard.py
```

Edit the PORT variable in dashboard.py to match your NodeB port.

## Expected Behavior

### Normal Operation

Node A broadcasts data every 500ms. Node B receives it and logs values.

### When RPM Exceeds 5000

Node B:
```
[CAN_RX] RPM: 5100
[WARNING] RPM threshold exceeded!
[CAN_TX] COMMAND: 1
[CAN_RX] ACK received for command: 1
```

Node A:
```
[CAN_TX] RPM: 5100
[COMMAND] Node B detected HIGH RPM!
[CAN_TX] ACK: 1
```

### When ACK Times Out

Sometimes you'll see:
```
[CAN_TX] COMMAND: 1
[ERROR] Node A did not ACK command!
[CAN_RX] ACK received for command: 1
```

This happens because the ACK arrives after the 200ms timeout window. The system still works - it's just a timing issue from message queue delays. You could increase the timeout to 500ms to reduce these errors.

## Troubleshooting

### UART shows garbage

Wrong baud rate or clock issue. The boards ship with HSE (external crystal) configured, but I found HSI (internal oscillator) more reliable. In the .ioc file:
- System Core → RCC: Disable HSE, Enable HSI
- Clock Configuration: PLL Source = HSI

### Code hangs at startup

Usually CAN_App_Init() waiting for the bus. Make sure:
- Both TJA1050 modules have power
- CANH and CANL are connected between them
- 120Ω resistors are installed on both ends
- Wiring matches the pinout above

### Too many commands being sent

Check NodeB tasks.c - you need static flags to send commands only once per threshold crossing:
```c
static bool rpmWarningActive = false;
static bool tempWarningActive = false;
```

Without these, it sends a command every 500ms while above threshold.

### Dashboard shows nothing

1. Check PORT in dashboard.py matches your actual port
2. Make sure you installed pyserial, not serial: `pip3 uninstall serial && pip3 install pyserial`

## Performance

Measured on actual hardware running for 2+ hours:

- CAN bus: 500 kbit/s
- Message rate: 2 Hz (every 500ms)
- Command latency: 15-45ms from threshold to ACK
- ACK success rate: 98-99%
- CPU usage: ~10% on both nodes
- RAM: ~12KB including FreeRTOS
- Flash: ~45KB

No crashes or watchdog resets observed.

## Project Structure
```
NodeA/Core/
├── Inc/
│   ├── can_app.h       # CAN protocol definitions
│   ├── uart_log.h      # Logging functions
│   └── tasks.h         # FreeRTOS task declarations
└── Src/
    ├── can_app.c       # CAN transmit/receive
    ├── uart_log.c      # UART wrapper
    ├── tasks.c         # Sensor node tasks
    └── main.c          # Initialization

NodeB/                  # Same structure, different task logic

python/
└── dashboard.py        # Live plotting script
```

## What I Learned

**Protocol design:** Understanding when to use ACKs vs. letting data be lossy. Commands need confirmation, periodic data doesn't.

**Debugging embedded systems:** Used UART logging heavily to isolate where code was hanging. Found that CAN_App_Init() blocks waiting for valid bus signals - can't test without hardware connected.

**Clock configuration:** HSE didn't work reliably on my boards, switched to HSI. This is apparently common with NUCLEO boards.

**FreeRTOS patterns:** ISR puts CAN messages into a queue, task reads them. Keeps interrupt handler fast.

**Real-time constraints:** Had to add "send once" logic because sending commands every 500ms while above threshold flooded the bus and caused UART buffer overflows.

## Future Work

- Add retry logic when ACK times out
- Implement actual actions (not just logging) when commands received
- Add a third node to test multi-node scenarios
- SD card logging of all CAN traffic
- Bootloader over CAN for firmware updates

## References

- [STM32F446RE Reference Manual](https://www.st.com/resource/en/reference_manual/dm00135183.pdf)
- [TJA1050 Datasheet](https://www.nxp.com/docs/en/data-sheet/TJA1050.pdf)
- [FreeRTOS Documentation](https://www.freertos.org/Documentation/RTOS_book.html)
- [CAN Specification (ISO 11898)](https://www.iso.org/standard/63648.html)

---
