# SPI Gate

An SPI controller for homebrew computers done entirely in discrete logic

## What is it?

SPI Gate is a discrete SPI controller intended to be used with homebrew (or real brew) 8- and 16-bit computers.
Typically, retro computers "bit-bang" an SPI interface, manually controlling GPIO pins, usually with something like a 6522 VIA, but this is slow and cumbersome.
Instead, it would be nice to have a programming model similar to an AVR: some memory-mapped registers and set-and-forget data transfer.

## Why is it?

In the end, the real purpose of this device is to enable fast and easy SD card access for a wide range of retro computers, but staying true to a through-hole component aesthetic.
Could this be done in a microcontroller, like an AVR?
Yes.
Could this be done in an FPGA/CPLD?
Yes.
Can many of the basic gates be combined in a GAL?
Again, yes.
And maybe some day I will add those as options.
But the purpose of this particular project is to create a fast, pleasant to use device using *discrete components* to learn how it was done in ye olden dayes.

## Where does it fit?

While the intention of this device is adaptability to a number of retrocomputers, the schematic and PCB is for a 6502-based computer.
Specifically, my homebrew [n8 Bit Special computer](https://github.com/nrivard/n8-Bit).
Adapting it for your 6502 (and hopefully 68000-based as I have a homebrew design) computer should be relatively simple.
The external signal requirements are intentionally pretty light, but this means you may have to change timing if your CPU samples data differently than a 6502.

## How is it supposed to work?

As stated in a previous section, I want it to look very similar to an AVR's SPI function:

* a memory-mapped control register
* a memory-mapped data register
* polling or interrupt when all 8 bits have been transferred
* programmable clock speed

But this is going in a retrocomputer, so the device also needs:
* device-select lines (to free up the computer's GPIO lines or remove the need for them entirely)

And lastly, intended purpose is for an SD card, and so to reduce complexity, it will be:
* Mode 0 only
* shift MSB only
* no write-collision detection (you should be careful!)

### External Signals

The following are the external signals that interface with the computer:

| Signal | I/O? | Description |
| --- | --- | --- |
| D0..7 | In-out | 8 bit data bus |
| A0 | Input | 1 bit register select |
| CLK | Input | System clock |
| RW | Input | Read when high, write when low |
| /CE | Input | Device selected when low |
| /RES | Input | Reset when low |
| /IRQ | Output | Transmission complete when low and interrupts enabled |

The following are external signals to SPI devices:

| Signal | I/O? | Description |
| --- | --- | --- |
| SCLK | Output | SPI Clock |
| MOSI | Output | Serial output to device |
| MISO | Input | Serial input from device |
| /CS0..3 | Output | Device select lines, low is assertion |

### Control Register

The control register is an 8-bit register that looks like:


| Bit 7 | 	Bit 6 |	Bit 5 |	Bit 4 |	Bit 3 |	Bit 2 |	Bit 1 |	Bit 0 |
| ---   | ---     | ---   | ---   | ---   | ---   | ---   | --- |
| ITC | BUSY | IEN | DEN | SEL1 | SEL0 | DIV1 | DIV0 |

DIV0 and DIV1 form a 2 bit clock divider select with the following possible values:

| SPI1 | SPI0 | Clock Divide |
| - | - | - |
| 0 | 0 | CLK / 2 |
| 0 | 1 | CLK / 8 |
| 1 | 0 | CLK / 16 |
| 1 | 1 | CLK / 32 |

There is no way to run this device at full CPU speed, it will always be divided in some way.
Taking the default clock rate of the n8 Bit computer (~3.6Mhz) you can select from speeds of 1.8Mhz, 450khz, 225khz, and 112khz.
This is important because certain devices will not operate at too high a clock speed.
SD cards, for example, require a clock speed between 100khz and 400khz to enter SPI mode.

SEL0 and SEL1 form a 2 bit device select.
`00` will select device 1, `01` will select device 2, and so on.

DEN is a Device Enable bit.
When on, whatever device is selected via SEL0 and SEL1 will be enabled. When off, no SPI device's CS line will be asserted.

IEN is an Interrupt Enable bit. When on, the /IRQ line will be asserted when a transfer has completed.

BUSY is a status flag.
When on, it means a transfer is in progress.
You can use this to avoid write-collisions.

ITC is the Interrupt/Transfer Complete flag.
When a transfer is complete, this bit will be asserted and can be polled in a loop or tested in an interrupt handler.
To clear the ITC flag, a read or a write must be made to the data register.

### Data Register

The data register is an 8 bit register.
Data transfer is initiated on a write and is performed independently from the system.
The result of the transfer from the SPI device is stored in the register.
Reading or writing this register will reset the ITC flag in the control register.

## What does it look like?

The design uses 12 74HC series chips:

| Chip | Count | Description |
| --- | --- | --- |
| 74HC273 | 1 | 8-bit register |
| 74HC74 | 1 | Dual flip-flop |
| 74HC245 | 1 | 8 bit bus transceiver |
| 74HC165 | 1 | Parallel-in, serial-out shift register |
| 74HC595 | 1 | Serial-in, parallel-out shift register |
| 74HC393 | 2 | Dual 4-bit binary counter |
| 74HC153 | 1 | Dual 4 input multiplexer |
| 74HC139 | 1 | Dual 2-to-4 decoder |
| 74HC00 | 1 | Quad-NAND gate |
| 74HC04 | 1 | Hex-NOT gate |
| 74HC08 | 1 | Quad-AND gate |

The design is relatively efficient, with only a single AND gate, a single NOT gate, and 1 of the counters being un-utilized.

### Schematic

