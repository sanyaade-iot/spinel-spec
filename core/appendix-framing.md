
# Recommended Framing Protocols {#appendix-framing}

Since this NCP protocol is defined independently of the physical transport or framing, any number of transports and framing protocols could be used successfully. However, in the interests of compatibility, this document provides some recommendations.

## UART Recommendations ###

The recommended default UART settings are:

* Bit rate:     115200
* Start bits:   1
* Data bits:    8
* Stop bits:    1
* Parity:       None
* Flow Control: Hardware

These values may be adjusted depending on the individual needs of the application or product, but some sort of flow control **MUST** be used. Hardware flow control is preferred over software flow control. In the absence of hardware flow control, software flow control (XON/XOFF) **MUST** be used instead.

We also **RECOMMEND** an Arduino-style hardware reset, where the DTR signal is coupled to the `RES` pin through a 0.01 micro-Farad capacitor. This causes the NCP to automatically reset whenever the serial port is opened. At the very least we **RECOMMEND** dedicating one of your AP host pins to controlling the `RES` pin on the NCP, so that you can easily perform a hardware reset if necessary.

### UART Bit Rate Detection ###

When using a UART, the issue of an appropriate bit rate must be considered. 115200 bits/s is the conventional baud rate for many serial peripherals. This rate, however, is slower than the theoretical maximum bitrate of many relevant network interfaces, e.g. the 802.15.4 2.4GHz PHY (250kbit). In most circumstances this mismatch is not significant because the overall bitrate will be much lower than either of these rates, but there are circumstances where a faster UART bitrate is desirable. Thus, this document proposes a simple bitrate detection scheme that can be employed by the AP to detect when the attached NCP is initially running at a higher bitrate.

The algorithm is to send successive NOOP commands to the NCP at increasing bitrates. When a valid `CMD_LAST_STATUS` response has been received, we have identified the correct bitrate.

In order to limit the time spent hunting for the appropriate bitrate, we RECOMMEND that only the following bitrates be checked:

* 115200
* 230400
* 1000000 (1Mbit)

The bitrate **MAY** also be changed programmatically by adjusting `PROP_UART_BITRATE`, if implemented.

(TODO: Define PROP_UART_BITRATE)

### HDLC-Lite {#hdlc-lite}

*HDLC-Lite* is the recommended framing protocol for transmitting Spinel frames over a UART. HDLC-Lite consists of only the framing, escaping, and CRC parts of the larger HDLC protocol---all other parts of HDLC are omitted. This protocol was chosen because it works well with software flow control and is widely implemented.

To transmit a frame with HDLC-lite, the 16-bit CRC must first be appended to the frame. The CRC function is defined to be CRC-16/CCITT, otherwise known as the [KERMIT CRC][].

[KERMIT CRC]: http://reveng.sourceforge.net/crc-catalogue/16.htm#crc.cat.kermit

Individual frames **MUST** terminated with a frame delimiter byte called the 'flag' byte (`0x7E`).

The following bytes values are considered *special* and should be escaped when present in data frames:

Byte Value | Description
:-----------|:----------------------
       0x7E | Frame Delimiter (Flag)
       0x7D | Escape Byte
       0x11 | XON
       0x13 | XOFF
       0xF8 | Vendor-Specific

When present in a data frame, these byte values are escaped by prepending the escape byte (`0x7D`) and XORing the value with `0x20`.

When receiving a frame, the CRC must be verified after the frame is unescaped. If the CRC value does not match what is calculated for the frame data, the frame MUST be discarded. The implementation MAY indicate the failure to higher levels to handle as they see fit, but MUST NOT attempt to process the deceived frame.

Consecutive flag bytes are entirely legal and MUST NOT be treated as a framing error. Consecutive flag bytes MAY be used as a way to wake up a sleeping NCP.

When first establishing a connection to the NCP, it is customary to send one or more flag bytes to ensure that any previously received data is discarded.

#### Reliability Discussion

All Spinel framing layers must make guarantees for reliability and in-order delivery. In this case,
we are assuming that the reliability of HDLC-Lite is adequarely implied by
both of the two usage scenarios we considered most likely:

1. That the NCP and AP are attached to the same physical circuit board, connected via
short traces routed over ground planes, with the AP and NCP being an inch or two apart at most.
2. That the NCP is connected to the AP as a USB dongle, presenting to the AP as
a virtual serial port.

In both of these cases, the high inherent reliability of the underlying hardware implies
that additional reliability guarantees are unnecessary under most circumstances, so we
have ommitted them from this particular framing recommendation. However,
if the UART connection reliability is suspect (due to noise or other factors), then a
more complex framing mechanism may be required.

## SPI Recommendations ##

We **RECOMMEND** the use of the following standard SPI signals:

*   `CS`:   (AP-to-NCP) Chip Select
*   `CLK`:  (AP-to-NCP) Clock
*   `MOSI`: Master-Output/Slave-Input
*   `MISO`: Master-Input/Slave-Output
*   `INT`:  (NCP-to-AP) Host Interrupt
*   `RES`:  (AP-to-NCP) NCP Hardware Reset

The `INT` signal is used by the NCP to indicate to the AP that the NCP has frames pending to send to it. When asserted, the AP SHOULD initiate a SPI transaction in a timely manner.

We RECOMMEND the following SPI properties:

*   `CS` is active low.
*   `CLK` is active high.
*   `CLK` speed is larger than 500 kHz.
*   Data is valid on leading edge of `CLK`.
*   Data is sent in multiples of 8-bits (bytes).
*   Bytes are sent most-significant bit first.

This recommended configuration may be adjusted depending on the individual needs of the application or product.

### SPI Framing Protocol ####

Each SPI frame starts with a 5-byte frame header:

~~~
  0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
|      HDR      |   RECV_LEN (Little Endian)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   DATA_LEN (Little Endian)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
Figure: Diagram of SPI frame header

 `HDR`
 : The first byte is the header byte (defined below)

`RECV_LEN`
: The second and third bytes indicate the largest frame size that that device is ready to receive. If zero, then the other device must not send any data. (Little endian)

`DATA_LEN`
: The fourth and fifth bytes indicate the size of the pending data frame to be sent to the other device. If this value is equal-to or less-than the number of bytes that the other device is willing to receive, then the data of the frame is immediately after the header. (Little Endian)

The `HDR` byte is defined as:

      0   1   2   3   4   5   6   7
    +---+---+---+---+---+---+---+---+
    |RST|CRC|CCF|  RESERVED |PATTERN|
    +---+---+---+---+---+---+---+---+

`RST`
: This bit is set when that device has been reset since the last time `CS` was asserted.

`CRC`
: This bit is set when that device supports writing a 16-bit CRC at the end of the data. The CRC length is **MUST NOT** included in DATA_LEN.

`CCF`
: "CRC Check Failure". Set if the CRC check on the last received frame failed, cleared to zero otherwise. This bit is only used if both sides support CRC.

`RESERVED`
: These bits are all reserved for future use. They **MUST** be cleared to zero and **MUST** be ignored if set.

`PATTERN`
: These bits are set to a fixed value to help distinguish valid SPI frames from garbage (by explicitly making `0xFF` and `0x00` invalid values). Bit 6 **MUST** be set to be one and bit 7 **MUST** be cleared (0). A frame received that has any other values for these bits **MUST** be dropped.

Prior to a sending or receiving a frame, the master **MAY** send a 5-byte frame with zeros for both the max receive frame size and the the contained frame length. This will induce the slave device to indicate the length of the frame it wants to send (if any) and indicate the largest frame it is capable of receiving at the moment. This allows the master to calculate the size of the next transaction. Alternatively, if the master has a frame to send it can just go ahead and send a frame of that length and determine if the frame was accepted by checking that the `RECV_LEN` from the slave frame is larger than the frame the master just tried to send. If the `RECV_LEN` is smaller then the frame wasn't accepted and will need to be transmitted again.

This protocol can be used either unidirectionally or bidirectionally, determined by the behavior of the master and the slave.

If the the master notices `PATTERN` is not set correctly, the master should consider the transaction to have failed and try again after 10 milliseconds, retrying up to 200 times. After unsuccessfully trying 200 times in a row, the master MAY take appropriate remedial action (like a NCP hardware reset, or indicating a communication failure to a user interface).

At the end of the data of a frame is an optional 16-bit CRC, support for which is indicated by the `CRC` bit of the `HDR` byte being set. If these bits are set for both the master and slave frames, then CRC checking is enabled on both sides, effectively requiring that frame sizes be two bytes longer than would be otherwise required. The CRC is calculated using the same mechanism used for the CRC calculation in HDLC-Lite (See (#hdlc-lite)). When both of the `CRC` bits are set, both sides must verify that the `CRC` is valid before accepting the frame. If not enough bytes were clocked out for the CRC to be read, then the frame must be ignored. If enough bytes were clocked out to perform a CRC check, but the CRC check fails, then the frame must be rejected and the `CRC_FAIL` bit on the next frame (and ONLY the next frame) MUST be set.

#### Reliability Discussion

TODO: Write me!

## I2C Recommendations {#i2c-recommendations}

TBD

<!-- RQ
  -- It may make sense to have a look at what Bluetooth HCI is doing
     for native I2C framing and go with that.
  -->

## Native USB Recommendations ###

TBD

<!-- RQ
  -- It may make sense to have a look at what Bluetooth HCI is doing
     for native USB framing and go with that.
  -->
