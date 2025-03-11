# PERIPHERAL UART CTRL (PUC)

This example is based on the [Peripheral UART Sample](https://docs.nordicsemi.com/bundle/ncs-2.9.0/page/nrf/samples/bluetooth/peripheral_uart/README.html), but using a packet type called PUC as an arbiter to determine if you want the message to be bridged between the UART peripheral and the BLE service, or if you want to do something on the device itself.

This is an rudimentary example illustrating using work queues (as well as transferring data using work containers) adding some system control with a simple protocol.

![image](https://github.com/user-attachments/assets/1b7915b0-2443-4717-8383-b4164fa0a200)


# PUC Packet
The PUC Packet is a simple preamble, command, length, data, CRC byte structure.
- **Preamble** : `'P' 'U' 'C'` or `0x50 0x55 0x43` (PUC is short for Peripheral UART control)
- **Command** : A single byte designating a command field.
- **Length** : The length of the data field. (It's a constant overhead protocol, but your messages might be shorter than what you ultimately decide to make in the end. You don't have to follow it exactly, this is just a fast way to get something going quickly.)
- **Data** : The data field, for example if you wanted to make a command to write to a log file, the contents of that write command would go here. In this sample, I did not break out that functionality, but it's certainly something you could add.
- **CRC** : A one-byte CRC that just performs a non-inclusive cumulative XOR check on the buffer. Below is a diagram.
  
  ![image](https://github.com/user-attachments/assets/fcf9e21e-18de-45d4-8e51-e4bda685d6de)

## Example packets for your testing
|Preamble|Command|Length|Data|CRC|Result|
|---|---|---|---|---|---|
|`0x50 0x55 0x43`|`0x01`|`0x04`|`0x0D 0x0E 0x0A 0x0D`|`0x47`|BLE on (advstart)|
|`0x50 0x55 0x43`|`0x02`|`0x04`|`0x0D 0x0E 0x0A 0x0D`|`0x44`|BLE off (discon/advstop)|
|`0x50 0x55 0x43`|`0x03`|`0x04`|`0x0D 0x0E 0x0A 0x0D`|`0x45`|LED4 on|
|`0x50 0x55 0x43`|`0x04`|`0x04`|`0x0D 0x0E 0x0A 0x0D`|`0x42`|LED4 off|
> Copy-pastable versions for something like Realterm
```
0x50 0x55 0x43 0x01 0x04 0x0D 0x0E 0x0A 0x0D 0x47 
0x50 0x55 0x43 0x02 0x04 0x0D 0x0E 0x0A 0x0D 0x44
0x50 0x55 0x43 0x03 0x04 0x0D 0x0E 0x0A 0x0D 0x45
0x50 0x55 0x43 0x04 0x04 0x0D 0x0E 0x0A 0x0D 0x42
```

If not PUC, it's the peripheral_uart serial-BLE bridge, utilizing the following characteristics.
![image](https://github.com/user-attachments/assets/98976f04-9683-4ef5-9382-502401d08d0d)


# Requirements
Pretty much anything that supports BLE and NUS if you remove the dk library abstraction.
If you want to test out-of-the-box, a Nordic DK is best.
| Compatible devices|
|---|
| nRF54L15DK |
| nRF52832DK |
| nRF52840DK|
| nRF5340DK|
> Frankly, any Nordic DK should do.

# Example usage
Connect via COM Port to a DK or UART pins.
You can send strings via uart, terminating in `\r\f` or `\n`, and it will send it over BLE, vice versa.
OR you can control the device itself via the few PUC commands implemented.

## PUC
Logs are reported over RTT, so you can use j-link RTT Viewer or logger to see the `LOG_...` outputs of the application.

![image](https://github.com/user-attachments/assets/04f6df47-3389-4bcd-9ae2-8d961a787e33)
## BLE-Serial bridge
![image](https://github.com/user-attachments/assets/250f15c2-a56f-462b-b4fa-bb894181437d)

```
*** Booting My Application v2.9.0-e0e712fbafb6 ***
*** Using nRF Connect SDK v2.9.0-7787b2649840 ***
*** Using Zephyr OS v3.7.99-1f8f3dc29142 ***
Starting Nordic Peripheral UART CTRL service example
Hello from phone!
```

# Notes
Future steps: you could expand from here, create a separate .c/.h software module to be more flexible, and move the arbitration from the uart callbacks and BT Receive callback. This is a quick illustration for a common need for bolting on an nRF device as a serial wire replacement, while also wanting some other processing commands for the chip itself.
For example, your arbiter could query the Nordic SoC from the Bluetooth side if you were to add it to the receive cb.
_(i.e. a full realization of this diagram)_

![image](https://github.com/user-attachments/assets/f5a61b61-96cb-45f4-9b03-22d9c3030655)

For testing, if you want to just generate some CRCs, here's a copy+pasteable code snippet.
```c
static int gen_xorc(uint8_t *buf, uint16_t len)
{
    int xorc = 0;
    for (int i = 0; i < len; i++)
		xorc ^= buf[i];
	return xorc;
}
```

I don't use the len arg in my puc_crc fn, it's there if you want it.

Furthermore, you'd have to decide what to pad empty data bytes with, that is not illustrated here as that's up to your implementation. You don't have to use PUC at all.
