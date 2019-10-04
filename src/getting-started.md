# Getting Started

Unlike many other programming fields, software development for embedded systems
requires special hardware. Bare minimum is a target device, for which the
software is developed, and a debug probe that is responsible for programming and
debugging the device. Often for a particular microcontroller unit (MCU), the
vendor offers a development board, which incorporates an MCU, a debug probe,
and some peripherals. But when the development reaches a printed circuit board
(PCB) prototyping stage, an external probe is desirable. There are various
debug probes in the market. A Chinese clone can cost a couple of dollars, while
original probes often cost hundreds. But there is one unique option that is
supported by Drone out-of-the-box - [Black Magic Probe](http://black-magic.org/).

Black Magic Probe, or BMP, is an open-source tool, like Rust or Drone, which is
invaluable when it comes to troubleshooting. Currently it supports Cortex-M and
Cortex-A targets. BMP implements the GDB protocol directly, which is nice,
because there is no need for intermediate software like OpenOCD. Also it embeds
a USB-to-UART converter. The official hardware is sold around $60 and is quite
good. But the firmware supports other hardware options. The most affordable of
which is the [Blue
Pill](https://web.archive.org/web/20190524151648/wiki.stm32duino.com/index.php?title=Blue_Pill).

Blue Pill is an ultra-popular and cheap development board for STM32F103C8T6
microcontroller. It can be bought for around $1.50 from AliExpress and also can
be programmed with Drone. It has 32-bit Cortex-M3 core running at 72 Mhz max, 20
Kb of RAM, and [128
Kb](https://web.archive.org/web/20190524151648/wiki.stm32duino.com/index.php?title=Blue_Pill#128_KB_flash_on_C8_version)
of flash memory. This is good for many applications and is enough to get started
with Drone. So the most affordable start would be with two Blue Pill boards, one
as a debug probe and the other as the host for Drone projects.

But there is another tool needed to flash the BMP firmware to a Blue Pill - a
USB-to-UART adapter. Out of the box a Blue Pill is flashed with a factory boot
loader, which allows programming its flash memory through UART. The cheapest
adapter would be enough for this. CH340G can be bought for around $0.50 from
AliExpress. It will not be needed after initial bootstrap of BMP, because it has
its own USB-to-UART. Though it is convenient to have a spare adapter, as
sometimes there can be multiple UARTs involved.
