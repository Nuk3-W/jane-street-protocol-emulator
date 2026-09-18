# Protocol Emulator Instruction Set Architecture (ISA)

## High-Level Design Document

## 1. Overview

This document defines the high-level design of a custom 16-bit Instruction Set Architecture (ISA) tailored for an open-source, general-purpose protocol emulator Application-Specific Integrated Circuit (ASIC). Unlike traditional CPUs optimized for arithmetic and logic, this architecture is optimized for bit-banging hardware protocols (e.g., UART, SPI, I2C).

A feature has been implemented to embed a **Post-Delay** field directly into instruction opcodes, allowing cycle-accurate pin toggling and sampling without requiring complex timer peripherals or software loop padding.

## 2. Architectural State

The processor operates on a minimal set of internal state registers. There is no general-purpose register file or Arithmetic Logic Unit (ALU).

| Register | Size   | Description                                                                                                                    |
| :------- | :----- | :----------------------------------------------------------------------------------------------------------------------------- |
| **PC**   | 8-bit  | **Program Counter**. Addresses up to 256 instructions in the instruction SRAM.                                                 |
| **OSR**  | 32-bit | **Output Shift Register**. Holds data to be serialized and shifted out to GPIO pins.                                           |
| **ISR**  | 32-bit | **Input Shift Register**. Accumulates serial data sampled from GPIO pins.                                                      |
| **X**    | 8-bit  | **Scratch Register**. Used as a loop counter for repetitive operations.                                                        |
| **CFG**  | 8-bit  | **Configuration Register**. Defines shift directions (MSB-first vs. LSB-first) and pin drive modes (push-pull vs. open-drain). |

## 3. Instruction Format

All instructions are strictly 16 bits wide to ensure single-cycle fetches from the on-chip SRAM. The ISA utilizes a 3-bit Opcode, leaving a 13-bit payload tailored to each instruction's requirements.

**General Layout:**
`[ 15:13 Opcode ] [ 12:0 Payload (Instruction-Specific) ]`

## 4. Instruction Set Reference

### 4.1. Pin Control Instructions

| Opcode | Mnemonic | Format `[15:13]` `[12:10]` `[9:9]` `[8:0]` | Description                                                                                                         |
| :----- | :------- | :----------------------------------------- | :------------------------------------------------------------------------------------------------------------------ |
| `000`  | **SET**  | `000` `Pin(3)` `Val(1)` `Delay(9)`         | Drives the specified `Pin` to `Val` (0 or 1). PC execution stalls for `Delay` clock cycles after the pin is driven. |

### 4.2. Shift and Sample Instructions

| Opcode | Mnemonic | Format `[15:13]` `[12:10]` `[9:5]` `[4:0]` | Description                                                                                                              |
| :----- | :------- | :----------------------------------------- | :----------------------------------------------------------------------------------------------------------------------- |
| `001`  | **OUT**  | `001` `Pin(3)` `Count(5)` `Delay(5)`       | Shifts `Count` bits from the OSR to `Pin` (one bit per cycle). Stalls PC for `Delay` cycles post-execution.              |
| `010`  | **IN**   | `010` `Pin(3)` `Count(5)` `Delay(5)`       | Samples `Pin` into the ISR for `Count` cycles, shifting the ISR each cycle. Stalls PC for `Delay` cycles post-execution. |

### 4.3. Synchronization and Branching

| Opcode | Mnemonic | Format                               | Description                                                                                                                  |
| :----- | :------- | :----------------------------------- | :--------------------------------------------------------------------------------------------------------------------------- |
| `011`  | **WAIT** | `011` `Pin(3)` `Val(1)` `Rsvd(9)`    | Halts execution indefinitely until the state of `Pin` matches `Val`. Used for syncing to external clock edges or start bits. |
| `100`  | **JMP**  | `100` `Cond(3)` `Addr(8)` `Delay(2)` | Jumps to `Addr` if `Cond` is met. Conditions: `000`=Always, `001`=Pin High, `010`=Pin Low, `011`=X Not Zero.                 |

### 4.4. State Management and System I/O

| Opcode | Mnemonic | Format                              | Description                                                                                       |
| :----- | :------- | :---------------------------------- | :------------------------------------------------------------------------------------------------ |
| `101`  | **MOV**  | `101` `Dest(2)` `Imm(8)` `Delay(3)` | Loads an 8-bit immediate `Imm` into a destination (`00`=X, `01`=CFG).                             |
| `110`  | **PULL** | `110` `Rsvd(13)`                    | Loads 32 bits from the external system TX FIFO into the OSR. Stalls if the FIFO is empty.         |
| `111`  | **PUSH** | `111` `Rsvd(13)`                    | Pushes the 32-bit contents of the ISR to the external system RX FIFO. Stalls if the FIFO is full. |

## 5. Timing & Execution Model

Execution is highly deterministic, which is critical for protocol emulation.

- **Base Execution:** Unless otherwise specified, instruction fetch and decode take 1 clock cycle.
- **Hardware Delay Counter:** Whenever an instruction with a `Delay` field is executed, the control unit loads the value into a dedicated hardware down-counter. The Program Counter (PC) increment is disabled until the counter reaches zero.
- **Multi-cycle Operations:** The `IN` and `OUT` instructions consume exactly `Count` clock cycles to perform the shift operations, followed immediately by `Delay` idle cycles.

Total Cycles per instruction = `1 (Fetch/Decode) + Count (if applicable) + Delay`

## 6. System Integration (ASIC Level)

The core interfaces with the broader ASIC infrastructure via three primary boundaries:

1.  **Instruction Memory:** A 256x16 SRAM macro. The core outputs an 8-bit address and receives a 16-bit instruction word asynchronously or on the next clock edge.
2.  **System FIFOs:** 32-bit wide interfaces connecting the emulator to a slower system bus (e.g., a primary microcontroller reading/writing data to the emulator).
3.  **GPIO Matrix:** A routing matrix mapping the logical `Pin(3)` arguments (0-7) to physical chip pads.

## 7. Example Application: 8-N-1 UART Transmitter

This example demonstrates configuring a UART transmission at a calculated baud rate. `Delay` padding ensures exactly 10 cycles per bit (e.g., 1 MHz baud on a 10 MHz system clock).

```assembly
; UART TX on Logical Pin 0
; Frame: 1 Start bit, 8 Data bits, 1 Stop bit
; Cycles per bit: 10

_uart_tx_loop:
    PULL                            ; Block until data is available in system TX FIFO
    SET  Pin=0, Val=0, Delay=9      ; Drive TX Low (Start Bit), hold for 9 cycles
    OUT  Pin=0, Count=8, Delay=1    ; Shift 8 data bits out, hold for 1 cycle post-shift
    SET  Pin=0, Val=1, Delay=9      ; Drive TX High (Stop Bit), hold for 9 cycles
    JMP  Cond=Always, Addr=_uart_tx_loop
```
