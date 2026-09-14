# STM32 Interrupt-Driven LED Controller

An embedded systems learning project built on an STM32 Nucleo board using STM32CubeIDE and the STM32 HAL.

This project began as a simple LED blinking exercise and has gradually evolved into an interrupt-driven application featuring:

- Timer-based system timing
- GPIO interrupts and button debouncing
- A finite state machine for LED behavior
- UART command input
- Interrupt-driven UART reception
- RX ring buffering
- Remote terminal echo
- Interrupt-driven UART transmission
- TX ring buffering
- UART overrun tracking

The primary goal of this project is not simply to make an LED blink, but to progressively learn and implement common embedded systems concepts while building toward a more realistic application architecture.

---

## Features

### LED State Machine

The onboard LED operates using a simple finite state machine:

```text
LED_OFF ────> LED_ON
   ▲            │
   └────────────┘
```

The LED transitions between `LED_ON` and `LED_OFF` according to the currently selected pulse speed.

Available speeds:

| Speed | Period |
|---|---:|
| `SLOW` | 1000 ms |
| `MID` | 500 ms |
| `FAST` | 100 ms |

---

## Hardware Button Control

A hardware button interrupt cycles through the available LED speeds:

```text
SLOW → MID → FAST → SLOW
```

The button is handled through a GPIO external interrupt rather than polling.

A software debounce window prevents multiple button actions from being registered during a single physical press.

---

## Hardware Timer

TIM6 generates periodic interrupts used to maintain a millisecond counter:

```c
volatile uint32_t ms_counter;
```

The timer callback increments the counter:

```text
TIM6 Interrupt
      ↓
ms_counter++
```

The main application uses this counter for:

- LED timing
- Button debounce timing
- General non-blocking timing behavior

This avoids using blocking delays inside the main application loop.

---

# UART Command Interface

The application accepts commands through a UART terminal.

Currently supported commands:

```text
fast
mid
slow
```

Example:

```text
fast
```

sets the LED blinking speed to:

```text
FAST = 100 ms
```

---

## Remote Echo

Characters received from the UART are echoed back to the terminal by the STM32.

This allows the terminal to run without local echo enabled.

The basic flow is:

```text
Keyboard
   ↓
UART RX
   ↓
STM32
   ↓
UART TX
   ↓
Terminal Display
```

The STM32 is responsible for echoing characters rather than relying on the terminal emulator.

Basic backspace handling is also implemented.

---

# Interrupt-Driven UART Reception

UART reception is handled asynchronously using:

```c
HAL_UART_Receive_IT()
```

The UART receive completion callback performs minimal work:

```text
UART RX Interrupt
        ↓
Receive byte
        ↓
Place byte into RX ring buffer
        ↓
Immediately re-arm UART reception
```

The callback does not perform command parsing or other expensive application logic.

Instead, received data is passed to the main application through a ring buffer.

---

# RX Ring Buffer

The project uses a single-producer/single-consumer ring buffer for UART reception.

```text
              PRODUCER
          UART Interrupt
                │
                ▼
        ┌───────────────┐
        │   RX BUFFER   │
        └───────────────┘
                │
                ▼
             CONSUMER
             Main Loop
```

The UART interrupt acts as the producer.

The main loop acts as the consumer.

### RX Buffer Behavior

The buffer tracks:

```c
rx_head
rx_tail
```

If the buffer becomes full, incoming bytes are dropped and an overrun counter is incremented:

```c
rx_overruns
```

This provides visibility into whether incoming data is arriving faster than the application can process it.

---

# TX Ring Buffer

UART transmission is also handled asynchronously.

Instead of blocking the CPU while characters are transmitted, output is placed into a transmit queue.

The basic architecture is:

```text
printf()
   ↓
__io_putchar()
   ↓
TX Ring Buffer
   ↓
UART Transmit Interrupt
   ↓
UART Hardware
   ↓
Terminal
```

The transmit buffer allows application code to continue running while the UART hardware sends data in the background.

---

## Interrupt-Driven UART Transmission

Transmission begins when data is added to an idle TX queue.

Each UART transmission completion interrupt advances the queue and begins transmitting the next byte.

Conceptually:

```text
Add byte to TX queue
        │
        ▼
Is UART busy?
     │       │
    No      Yes
     │        │
     ▼        │
Start TX ◄────┘
     │
     ▼
TX Complete Interrupt
     │
     ▼
More bytes waiting?
     │       │
    Yes      No
     │        │
     ▼        ▼
Send next   Clear
 byte       tx_busy
```

The system tracks whether transmission is currently active using:

```c
tx_busy
```

---

## TX Overrun Tracking

The TX queue also detects when it becomes full.

If the application attempts to add data when no buffer space is available:

```c
tx_overruns++;
```

This makes dropped output detectable during debugging.

---

# Project Architecture

The application is structured around a simple interrupt-driven design.

```text
                    ┌─────────────────┐
                    │    Main Loop    │
                    │                 │
                    │ • LED FSM       │
                    │ • Commands      │
                    │ • RX Processing │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼

        GPIO Interrupt   TIM6 Interrupt   UART Interrupt
        Button Press       1 ms Tick       RX / TX

              │              │              │
              ▼              ▼              ▼

         buttonPressed    ms_counter    Ring Buffers
```

The general design principle is:

> **Keep interrupt handlers short and move application-level processing into the main loop whenever possible.**

---

# Non-Blocking Design

A major goal of this project is to progressively reduce blocking behavior.

The main loop does not wait for:

- Button presses
- UART input
- UART output
- LED timing intervals

Instead, these events are handled asynchronously using interrupts, timers, and state machines.

This allows multiple subsystems to operate concurrently without requiring an RTOS.

---

# Technologies Used

- STM32
- STM32CubeIDE
- STM32 HAL
- C
- GPIO
- External interrupts
- Hardware timers
- UART
- Interrupt-driven I/O
- Ring buffers
- Finite state machines

---

# Current Project Progress

### Completed

- [x] Basic GPIO LED control
- [x] LED finite state machine
- [x] Multiple blinking speeds
- [x] GPIO button interrupt
- [x] Software button debouncing
- [x] Hardware timer-based millisecond counter
- [x] Non-blocking LED timing
- [x] UART initialization
- [x] UART command interface
- [x] Remote terminal echo
- [x] Basic backspace handling
- [x] Interrupt-driven UART RX
- [x] RX ring buffer
- [x] RX overrun tracking
- [x] Interrupt-driven UART TX
- [x] TX ring buffer
- [x] TX overrun tracking

---

# Future Improvements

Potential next steps for the project include:

- [ ] Improve command parsing
- [ ] Add an `invalid command` response
- [ ] Add a `help` command
- [ ] Add status reporting
- [ ] Display RX/TX overrun statistics
- [ ] Support longer or more complex commands
- [ ] Improve terminal editing behavior
- [ ] Add command history
- [ ] Transmit multiple bytes per UART interrupt
- [ ] Implement DMA-based UART transmission
- [ ] Implement DMA-based UART reception
- [ ] Add a structured command parser
- [ ] Separate UART functionality into reusable driver modules
- [ ] Separate application logic into multiple source files

---

# Learning Goals

This repository documents my progression into embedded systems development.

Rather than immediately jumping into a large project, I am building functionality incrementally and focusing on understanding the underlying architecture of each system.

The progression so far has been roughly:

```text
GPIO
 ↓
LED State Machines
 ↓
Hardware Timers
 ↓
GPIO Interrupts
 ↓
UART Communication
 ↓
Interrupt-Driven UART RX
 ↓
RX Ring Buffers
 ↓
Interrupt-Driven UART TX
 ↓
TX Ring Buffers
 ↓
More Advanced Embedded Systems Features
```

The eventual goal is to apply these concepts to a larger embedded project involving real sensors, peripherals, and hardware control.

---

## Notes

This project is primarily a learning project, and some implementations are intentionally kept simple while concepts are being explored.

The goal is to understand how the underlying embedded systems mechanisms work rather than immediately using the most abstract or optimized solution available.

As the project develops, components will be refactored and improved.