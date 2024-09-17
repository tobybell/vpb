# VISORS Propulsion Bootloader

The VISORS propulsion bootloader is a small program stored on the ATmega128 chip
of the VISORS propulsion system in addition to the VISORS propulsion
application software. The purpose of the bootloader is to enable updating the
propulsion application software via telecommand.

The bootloader is stored in the upper-most 1024 bytes in flash memory of the
ATmega128 chip, starting at address 130,048/0x1fc00, and the application
software is free to use the lower 130,048 bytes starting at address 0x00000.

The bootloader is based closely on a widely used bootloader implementation for
AVR chips called Optiboot, modified to be able to communicate via the packet
format used by the existing VISORS propulsion application software and VISORS
BCT bus.

The source code is available in the [`VISORS-FLT` branch of the `cgtc-software` repository](
https://github-research.gatech.edu/LRG-GitHub/cgtc-software/tree/VISORS-FLT/visors-bootloader).

## User Guide

<details>
<summary><b>Startup</b></summary>

### Startup

The ATmega128 should be configured to run the bootloader on boot/power-on. The
bootloader will wait for 2 seconds to receive a valid bootloader command. If it
does not receive a valid bootloader command for 2 seconds, it will exit and
start the most recently loaded application.

The bootloader assumes the application is stored at address 0x0000 in memory.

> **IMPORTANT**
> When flashing the bootloader to the chip, make sure to set the
> ATmega128's fuses to run the 1024-B bootloader on power-on. This means
> making sure that `HFUSE` bits 0 and 1 are LOW, and bit 2 is HIGH.
</details>

<details>
<summary><b>Writing pages</b></summary>

### Writing Pages

The ATmega128 has 128 kiB of flash memory, addressable as 256-byte pages. There
are 512 pages (512 × 256 B = 128 kiB). The bootloader allows writing to flash
pages one 256-byte page at a time.

For VISORS, the command to write a page is split into two commands,
WRITE_PAGE_LO and WRITE_PAGE_HI, because BCT only supports a maximum command
size of 195 B, so writing a full 256 B in one command is not possible.
PAGE_WRITE_LO and PAGE_WRITE_HI write the lower and upper half (128 B) of a
page, respectively. **Note: these commands operate as a pair.** Pages can
only be written a full 256 B at a time, so the PAGE_WRITE_LO command merely
writes the first 128 B to a temporary buffer, and the PAGE_WRITE_HI command
actually writes the full page to flash. If either command is dropped or
mistransmitted, the page will not be written.

Both PAGE_WRITE_LO and PAGE_WRITE_HI commands specify the page number. During
the PAGE_WRITE_LO command, the bootloader will save the page number from the LO
command, and during a PAGE_WRITE_HI command, it will check to ensure that the
page number matches the saved page number from the LO command. This avoids
corruption due to dropped half-page commands.

> **CAUTION**
> The bootloader itself occupies the last 4 pages of flash (last 1 kiB of
> memory). The application starts at page 0 and grows upwards. So applications
> can be up to 508 pages (127 kiB) large. The bootloader should not be used to
> modify the bootloader pages (509-511), as doing so might render the device
> unusable.
</details>

<details>
<summary><b>Reading pages</b></summary>

### Reading pages

The ATmega128 has 128 kiB of flash memory, addressable as 256-byte pages. There
are 512 pages (512 × 256 B = 128 kiB). The bootloader can read one page at a
time using the PAGE_READ command. Pages are identified by index, from 0 to 511.
Upon reading a page, the bootloader will respond with PAGE_READ telemetry
containing 256 B of data. Each data response is also tagged with the page
number, to avoid treating an out-of-sync response for an old request as
containing the data for a newer request.

> **IMPORTANT**
> Use the page number included in the response when interpreting PAGE_READ
> telemetry. Do not assume that the next PAGE_READ telemetry received is for
> the most recent PAGE_READ command, especially if reading many pages in quick
> succession.
</details>

<details>
<summary><b>Watchdog timer</b></summary>

### Watchdog Timer

The bootloader includes a watchdog timer that is set to 2 seconds when the
bootloader starts. If the watchdog timer expires, the chip will exit the
bootloader and start the VISORS propulsion application software. This means that
interaction with the bootloader must begin within 2 seconds of powering on the
chip.

The watchdog timer is reset to 2 seconds every time the bootloader
receives a valid bootloader command (it will not be reset by VISORS
propulsion application commands, which follow a different format). This means
that communication with the bootloader can take longer than 2 seconds if needed,
as long as each command is separated by less than 2 seconds.

The watchdog timer is a safe and normal way to exit the bootloader into the
application, in addition to the EXIT command. If the bootloader is
used to program the chip, the application can be started after programming is
done either by sending the EXIT command or by waiting 2 seconds.

The watchdog timer can be disabled completely using the TIMEOUT_DISABLE command,
which may be useful for testing or manual operation. If the watchdog timer is
disabled, the bootloader can be exited into the application using the EXIT
command or by power-cycling the chip.
</details>

<details>
<summary>
<b>Packet framing</b>
</summary>

### Command framing

```
[sync header: 35 2E F8 53]
[unused: xx xx]
[payload]
[crc16]
```

`crc16` is the BCT CRC16 of a all bytes after the sync header (so, starting from
`xx xx`) to the end of `payload`.

### Telemetry framing

```
[sync header: 35 2E F8 53]
[ccsds header: 0b 32 00 00]
[length]
[payload]
[crc16]
```

`length` is the length of `payload` + 1, as a big-endian 16-bit integer.

`crc16` is the BCT CRC16 of a all bytes after the sync header (so, starting from
`0b 32`) up to the end of `payload`.
</details>

## Commands Reference

<details>
<summary>
<code>0x30</code>
<code>PAGE_WRITE_LO [page] [data]</code>
<i>Start writing the first 128-B half of a 256-B flash page</i>
</summary>
<p>

**Note:** this command does not write to flash memory, it only saves the data to
a temporary buffer. This command should be followed immediately by a
`PAGE_WRITE_HI` command to actually save the data to flash.

#### Parameters

> |name|type|description|
> |-|-|-|
> |`page`|`0...511`|Flash memory page number|
> |`data`|`[128] u8`|First half of page contents|

#### Encoding

`0x30 | high(page)` `page & 0xFF` `data`

> |size|value|description|
> |-|-|-|
> |1|`Ox30 + (page > 256 ? 0x80 : 0)`|Opcode + page MSB|
> |1|`page % 256`|Page LSB|
> |128|`data`|Page contents lower half|

#### Responses

> |opcode|description|
> |-|-|
> |`PAGE_WRITE_LO`|Successfully saved lower 128 B of `page` to temporary buffer|

#### Example encoding

```
// PAGE_WRITE_LO page=13 data=[01 02 03 04 05 06 07 08 01 02...]
30 0d 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01
02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04
05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07
08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02
03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08
```

```
// PAGE_WRITE_LO page=300 data=[ff fe fd fc fb fa f9 f8 ff fe...]
b0 2c ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff
fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc
fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9
f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe
fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8
```
</details>

<details>
<summary>
<code>0x31</code>
<code>PAGE_WRITE_HI [page] [data]</code>
<i>Finish writing the second 128-B half of a 256-B flash page</i>
</summary>
<p>

**Note:** this command should immediately follow a matching `PAGE_WRITE_LO`
command. If it does not, it will return an error.

#### Parameters

> |name|type|description|
> |-|-|-|
> |`page`|`0...511`|Flash memory page number|
> |`data`|`[128] u8`|Second half of page contents|

#### Encoding

`0x31 | high(page)` `page & 0xFF` `data`

> |size|value|description|
> |-|-|-|
> |1|`Ox31 + (page > 256 ? 0x80 : 0)`|Opcode + page MSB|
> |1|`page % 256`|Page LSB|
> |128|`data`|Page contents upper half|

#### Responses

> |opcode|description|
> |-|-|
> |`PAGE_WRITE_HI`|Successfully wrote 256 B of `page` to flash memory|
> |`PAGE_WRITE_MISMATCH`|`page` did not match the most recent `PAGE_WRITE_LO` command|

#### Example encoding

```
// PAGE_WRITE_HI page=13 data=[01 02 03 04 05 06 07 08 01 02...]
31 0d 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01
02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04
05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07
08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02
03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08
```

```
// PAGE_WRITE_HI page=300 data=[ff fe fd fc fb fa f9 f8 ff fe...]
b1 2c ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff
fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc
fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9
f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe
fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8
```
</details>

<details>
<summary>
<code>0x32</code>
<code>PAGE_READ [page]</code>
<i>Read a 256-B flash page</i>
</summary>
<p>

This can be used to verify flash contents if needed.

#### Parameters

> |name|type|description|
> |-|-|-|
> |`page`|`0...511`|Flash memory page number|

#### Encoding

`0x32 | high(page)` `page & 0xFF`

> |size|value|description|
> |-|-|-|
> |1|`Ox32 + (page > 256 ? 0x80 : 0)`|Opcode + page MSB|
> |1|`page % 256`|Page LSB|

#### Responses

> |opcode|description|
> |-|-|
> |`PAGE_READ`|Read-back of the 256-B contents of `page`|

#### Example encoding

```
// PAGE_READ page=15
32 0f
```

```
// PAGE_READ page=482
b2 e2
```
</details>

<details>
<summary>
<code>0x33</code>
<code>EXIT</code>
<i>Exit bootloader and start application</i>
</summary>
<p>

This command can be used after programming to explicitly exit the bootloader and
start the application. It is usually optional, since the bootloader will exit
automatically after 2 seconds of inactivity. But if `TIMEOUT_DISABLE` is used,
then `EXIT` is necessary to exit the bootloader when done (if you don't want to
power-cycle).

#### Parameters

> None

#### Encoding

`0x33`

> |size|value|description|
> |-|-|-|
> |1|`Ox33`|Opcode|

#### Responses

> |opcode|description|
> |-|-|
> |`EXIT`|Bootloader exiting and jumping to application|

#### Example encoding

```
// EXIT
33
```
</details>

<details>
<summary>
<code>0x34</code>
<code>TIMEOUT_DISABLE</code>
<i>Disable the 2-second watchdog timer</i>
</summary>
<p>

Normally, the bootloader will automatically exit and start the application after
2 seconds of inactivity. If this is undesirable (e.g., for interactive use by a
human operator), send `TIMEOUT_DISABLE` within 2 seconds after power-on. After
the timeout is disabled, the bootloader will only exit via `EXIT` command or via
power-cycle.

#### Parameters

> None

#### Encoding

`0x34`

> |size|value|description|
> |-|-|-|
> |1|`Ox34`|Opcode|

#### Responses

> |opcode|description|
> |-|-|
> |`TIMEOUT_DISABLE`|Disabled watchdog timer|

#### Example encoding

```
// TIMEOUT_DISABLE
34
```
</details>

## Telemetry Reference

<details>
<summary>
<code>0x40</code>
<code>PAGE_WRITE_LO [page]</code>
<i>Saved page lower half to temporary buffer</i>
</summary>

#### Parameters

> |name|type|description|
> |-|-|-|
> |`page`|`0...511`|Flash memory page number|

#### Encoding

`0x40 | high(page)` `page & 0xFF`

> |size|value|description|
> |-|-|-|
> |1|`Ox40 + (page > 256 ? 0x80 : 0)`|Opcode + page MSB|
> |1|`page % 256`|Page LSB|

#### Example encoding

```
// TLM_PAGE_WRITE_LO page=13
40 0d
```

```
// TLM_PAGE_WRITE_LO page=300
b0 2c
```
</details>

<details>
<summary>
<code>0x41</code>
<code>PAGE_WRITE_HI [page]</code>
<i>Saved full page to flash memory (combined with previous `PAGE_WRITE_LO`)</i>
</summary>

#### Parameters

> |name|type|description|
> |-|-|-|
> |`page`|`0...511`|Flash memory page number|

#### Encoding

`0x41 | high(page)` `page & 0xFF`

> |size|value|description|
> |-|-|-|
> |1|`Ox41 + (page > 256 ? 0x80 : 0)`|Opcode + page MSB|
> |1|`page % 256`|Page LSB|

#### Example encoding

```
// TLM_PAGE_WRITE_HI page=0
41 00
```

```
// TLM_PAGE_WRITE_HI page=256
c1 00
```
</details>

<details>
<summary>
<code>0x42</code>
<code>PAGE_READ [page] [data]</code>
<i>Got data from reading back a 256-B flash page</i>
</summary>

#### Parameters

> |name|type|description|
> |-|-|-|
> |`page`|`0...511`|Flash memory page number|
> |`data`|`[256] u8`|Full page contents|

#### Encoding

`0x42 | high(page)` `page & 0xFF` `data`

> |size|value|description|
> |-|-|-|
> |1|`Ox42 + (page > 256 ? 0x80 : 0)`|Opcode + page MSB|
> |1|`page % 256`|Page LSB|
> |256|`data`|Page contents|

#### Example encoding

```
// TLM_PAGE_READ page=6 data=[01 02 03 04 05 06 07 08 01 02...]
42 06 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01
02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04
05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07
08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02
03 04 05 06 07 08 01 02 03 04 05 06 07 08 01 02 03 04 05 06 07 08
```

```
// TLM_PAGE_READ page=270 data=[ff fe fd fc fb fa f9 f8 ff fe...]
c2 0e ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff
fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc
fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9
f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe
fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8 ff fe fd fc fb fa f9 f8
```
</details>

<details>
<summary>
<code>0x43</code>
<code>EXIT</code>
<i>Bootloader exiting and jumping to application</i>
</summary>

#### Parameters

> None

#### Encoding

`0x43`

> |size|value|description|
> |-|-|-|
> |1|`Ox43`|Opcode|

#### Example encoding

```
// TLM_EXIT
43
```
</details>

<details>
<summary>
<code>0x44</code>
<code>TIMEOUT_DISABLE</code>
<i>Watchdog timer disabled</i>
</summary>

#### Parameters

> None

#### Encoding

`0x44`

> |size|value|description|
> |-|-|-|
> |1|`Ox44`|Opcode|

#### Example encoding

```
// TIMEOUT_DISABLE
44
```
</details>

<details>
<summary>
<code>0x45</code>
<code>START</code>
<i>Bootloader started</i>
</summary>
<p>

> This message is emitted on power-on. It indicates the bootloader started
> running. At this point, the 2-second watchdog timer is active.

#### Parameters

> None

#### Encoding

`0x45`

> |size|value|description|
> |-|-|-|
> |1|`Ox45`|Opcode|

#### Example encoding

```
// START
45
```
</details>

<details>
<summary>
<code>0x46</code>
<code>TIMEOUT</code>
<i>2-second watchdog timer timed out</i>
</summary>
<p>

> This message is emitted if a reset is triggered by the 2-second watchdog
> timer, and the bootloader is about to start the application.

#### Parameters

> None

#### Encoding

`0x46`

> |size|value|description|
> |-|-|-|
> |1|`Ox46`|Opcode|

#### Example encoding

```
// TIMEOUT
46
```
</details>

<details>
<summary>
<code>0x47</code>
<code>BAD_CRC</code>
<i>Received command with an invalid CRC</i>
</summary>
<p>

> May be sent in response to any attempted command if the CRC was found to be
> invalid.

#### Parameters

> None

#### Example encoding

```
// BAD_CRC
47
```
</details>

<details>
<summary>
<code>0x48</code>
<code>BAD_OPCODE [opcode]</code>
<i>Received command with invalid opcode</i>
</summary>
<p>

> May be sent in response to any attempted command if the opcode was not
> recognized. If the opcode was not recognized, that means the packet length was
> also unknown, so the bootloader will wait until it sees the next 4-byte
> command sync header.

#### Parameters

|name|type|description|
|-|-|-|
|`opcode`|`u8`|The received opcode that was not recognized|

#### Encoding

`0x48` `opcode`

> |size|value|description|
> |-|-|-|
> |1|`Ox48`|Opcode|
> |1|`opcode`|Invalid received command opcode|

#### Example encoding

```
// BAD_OPCODE opcode=b5
48 b5
```
</details>

<details>
<summary>
<code>0x49</code>
<code>PAGE_WRITE_MISMATCH [hi_page] [lo_page]</code>
<i>Received mismatched `PAGE_WRITE_LO and `PAGE_WRITE_HI` pages</i>
</summary>
<p>

> **Note:** only the LSB of the mismatched pages are included in this message,
> so it's possible, though unlikely, that `hi_page` and `lo_page` would actually
> be equal. This would indicate that one of them was for a high page number (>=
> 256) and the other was not.

#### Parameters

> |name|type|description|
> |-|-|-|
> |`lo_page`|`u8`|LSB of the page from the `PAGE_WRITE_LO` command|
> |`hi_page`|`u8`|LSB of the page from the `PAGE_WRITE_HI` command|

#### Encoding

`0x49` `hi_page` `lo_page`

> |size|value|description|
> |-|-|-|
> |1|`Ox49`|Opcode|
> |1|`hi_page`|`PAGE_WRITE_HI` page number|
> |1|`lo_page`|Mismatched `PAGE_WRITE_LO` page number|

#### Example encoding

```
// PAGE_WRITE_MISMATCH hi_page=19 lo_page=18
49 13 12
```
</details>

## Implementation

The VISORS propulsion bootloader is based on a widely used bootloader for AVR
chips called Optiboot (also used by Arduino), which was modified to be able to
communicate with the VISORS BCT bus. Optiboot originally implemented the STK500
communication protocol in order to interface with widely available AVR chip
programming hardware. Since the VISORS propulsion system instead needs to
communicate with the BCT bus, the STK500 communication protocol was scrapped and
replaced with a simple custom protocol (documented above) that could run on top
of the command packet format sent to propulsion by the BCT bus. This included
adding BCT's CRC16 validity check, which improves the bootloader's robustness
to communication errors.

#### References

[Optiboot Flash](https://github.com/MCUdude/optiboot_flash) – variant of Optiboot on which VISORS propulsion bootloader is most closely based

[Optiboot](https://github.com/Optiboot/optiboot) – original Optiboot, widespread bootloader for AVR chips

[Optiboot for Arduino](https://github.com/arduino/ArduinoCore-avr/tree/master/bootloaders/optiboot) – Optiboot customized for Arduino, the widespread hobbyist platform

## Testing

Testing was performed by Toby Bell and Jonathan Vollrath on Monday, 16 Sep 2024.
The test procedure was authored by Toby Bell and carried out successfully by
Jonathan Vollrath at Georgia Tech, with Toby monitoring remotely via Zoom at
Stanford University.

The [test procedure and record can be found here](testing-2024-09-16.pdf).

## Acknowledgement

The author, Toby Bell, would like to express his utmost gratitude to the
creator Michael Pollet and other maintainers of
[SimAVR](https://github.com/buserror/simavr). The VISORS propulsion bootloader
was developed in a very short timeframe in order to be ready for propulsion
system integration—about 3 days. Since no ATmega128 hardware was available to
Toby Bell during this time, SimAVR, an open-source full-chip simulator for AVR
chips, was used to develop and test the bootloader. Ultimately, the bootloader
performed exactly as expected when tested on real hardware for the first time.
This was an invaluable resource without which developing the VISORS bootloader
so quickly would not have been possible.
