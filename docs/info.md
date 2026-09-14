## How it works

The project is being developed as a programmable, general-purpose protocol
emulator for UART, SPI, and I2C. The final design will execute a compact
instruction stream that reads pins, drives pins, counts cycles, and provides
deterministic timing for protocol bit-banging.

The current RTL is a bring-up placeholder: it adds the eight dedicated input
bits (`ui_in`) to the eight bidirectional input bits (`uio_in`) and presents
the eight-bit result on `uo_out`. The bidirectional pins are inputs in this
version, and an overflow carry is discarded.

## How to test

For the current placeholder design, hold reset low, then release it. Drive an
eight-bit value on `ui_in` and another on `uio_in`; `uo_out` equals their
eight-bit sum. For example, `ui_in = 20` and `uio_in = 30` produces
`uo_out = 50`.

Run the cocotb test suite with `make -C test` to verify this behavior. The
testbench and this section will be updated with the protocol emulator RTL.

## External hardware

No external hardware is required for the current placeholder design.
