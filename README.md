# VoTonic

This project plans to reverse engineer and document the protocol used by the inverters, solar charge controllers, tank sensors, displays etc. produced by the German company [Votronic](https://votronic.de/).
These products can be found in recreational and expedition vehicles, boats and so on.

Note:
There is also the [S-BC Bluetooth Connector](https://www.votronic.de/index.php/en/products/measuring-display-systems/energy-monitor-app) available from Votronic.
If all you want to do is interface your setup with a smartphone, this might be your way to go.
When I started working on this project, this component didn’t exist yet.
And now, I kind of want to finish it anyway.

## Status

Work in progress.

I’ve figured out the physical/electrical setup (RS485 with RJ45 connectors) and am currently trying to make sense of the serial protocol that’s spoken on the wire.

## Hardware I’m Using

### Bus System VBS2 Master Box

This is the heart of the system, with one wire screw connection bar each at the top and bottom, interfacing to non-bus components.

It also has three RJ45 connections labeled “Bus”, which are (in my setup) connected to

* the Display / control unit
* the Equipment Adapter and
* the Solar/Battery Charger

### [Battery Charger Triple VBCS 45/30/350](https://www.votronic.de/index.php/en/products2/battery-charger-series-vbcs-triple/standard-version/vbcs-45-30-350)

Solar Charge Controller and Battery Charger.
Connects to both the ignition and RV batteries.
Also has a bus connection.

I don’t mess with this thing, it has big wires.

### [MobilPOWER Inverter SMI 1700 Sinus ST-NVS](https://www.votronic.de/index.php/en/products2/sine-inverters/standard-version/smi-1700-st-nvs)

Does the 230V shizzle.
Isn’t directly connected to the bus, but via the EQ-BCI adapter described below.

### Equipment Adapter EQ-BCI

This is a matchbox-sized device that has two RJ45 jacks labeled “Bus” and one smaller jack (RJ12? have to check) labeled “B2B/Charger/Inverter”.
It apparently translates between whatever protocol my inverter is using and the bus protocol analyzed in this project.

Since it has two bus connections, of which only one was used when I got my camper van, it was the perfect candidate to start reverse engineering and tapping into the bus.
Looking at the chips inside the Equipment Adapter made me realize that the bus is using RS485, and looking at the datasheets of the ICs allowed me to reverse engineer the RJ45 pin mapping.

#### Board Contents

The top of the board contains:

* `6N137` optocoupler
* 25V 100µF capacitor
* ZTT 8.00MT 8MHz ceramic resonator
* diode labeled “Z 15”

The bottom of the board contains:

* ATMEL ATMEGA8L 8AU 1710D
* 7LB176 68M AKY5 [RS-485 transciever](http://www.ti.com/lit/ds/slls067h/slls067h.pdf)
  * pin 1 is at the bottom left (holding it so that the label on the IC is upright)
* 2× 690Y 2951 CMC [voltage regulators](http://ww1.microchip.com/downloads/en/DeviceDoc/mic2950.pdf), one for each bus port, apparently for power supply
* TCLT1002 V 727 68 [optocoupler](https://www.vishay.com/docs/83515/tclt1000.pdf)

## Physical Connection

Most of the products I’m using connect through an 8-wire RJ45 jack.
You can use cheap (flat) phone cable or twisted-pair (Ethernet) cables to connect to them.

When using a cable with T568B colors, pin 1 uses the white/orange wire; T568A use white/green on pin 1.

The pinout looks like this:

* **1:** not sure, seems not connected or reserved
* **2 & 3:** Vin (power supply)
* **4:** RS485 A wire
* **5:** RS485 B wire
* **6, 7 &8:** ground

## RS485 / UART properties

The wire protocol has a transfer rate of 19,200 bits per second, 8 bit, with even parity and 1 stop bit.

Linux users can use the following command to set up their RS485 interface:

```sh
stty -F /dev/ttyWHATEVER 19200 cs8 -cstopb parenb -parodd raw
```

I’m using [this RS485 USB adapter](https://www.amazon.de/USB-RS485-Adapter-mit-Gehäuse/dp/B00I9H5J02).

If you want to look at the raw data using `hexump`, I’d use something like

```sh
stdbuf -i0 -o0 -e0 hexdump -Cv /dev/ttyWHATEVER
```

## Protocol

Most of this is speculation.
Don’t rely on it yet.
All byte values are written in hexadecimal.

There seems to be no collision avoidance on the bus.
Devices seem to talk whenever they want, even while another transmission is already running.
This leads to something like 5 to 10 % of packets looking wrong:
They’re longer than expected because they’re two packets mushed together, where one starts somewhere in the middle of the other.

Every packet on the bus I’ve seen seems to be 9 bytes long and consists of the following parts:

* A start byte, which is always `aa`.
* Three bytes of header data which could contain a “from” address, “to” address, type, mode or a combination thereof. Not yet understood.
* Probably the length of the payload; this was `03` in all non-collided packets I saw.
* The payload; this was three bytes long in all non-collided packets I saw.
* A one-byte checksum, but I don’t know the algorithm yet. It’s not a nonce or timestamp, because if the other bytes in the packet stay the same, this value is identical, too.

## FAQ

### Why are you doing this?

Because I own a camper van containing some of their products, and I’d like to be able to interface with them in order to read power and water levels etc.

### There’s a typo in the project name!

No there’s not.
For trademark reasons, this project is named after [vodka tonic](https://en.wikipedia.org/wiki/Vodka_tonic).
Because why not!
