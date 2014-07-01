cgminer-scrypt
==============

CGMiner 4.3.5 with GridSeed and Zeus scrypt ASIC support.

This file describes Scrypt-specific settings and options.
For general CGMiner information refer to README.
Scrypt algorithm code was ported from CGMiner version 3.7.2.

## Zeus ##

	./autogen.sh
	./configure --enable-scrypt --enable-zeus
	make

The Zeus driver needs to be configured with two runtime options: the number of
chips per ASIC device with `--zeus-chips` and the desired clock speed in MHz
with `--zeus-clock`. These options are applied to all Zeus miners. To set options
for a specific device use `--zeus-options ID,chips,clock` where ID specifies
the device (see below) and chips and clock set the respective options. Multiple
ID,chips,clock tuples can be joined together separated by semi-colons.

### Device Selection ###

With no `--scan-serial` options the driver will use libusb to autodetect any
connected miners and to perform all device I/O operations. This is the
recommended method if multiple drivers are compiled into cgminer.

If `--scan-serial zeus:auto` is specified, the driver will use libudev to
identify which USB-serial ports (if any) are from a Zeus miner and open those
ports directly. All I/O will be done using direct serial reads and writes
(not through libusb). This method may not work properly if multiple drivers
are enabled. This method is only available on Linux.

Individual devices can be specified manually using `--scan-serial zeus:/dev/ttyX`
(note the "zeus:" is optional if only the Zeus driver has been compiled in). This
disables autodetection (for the Zeus driver only) and all I/O will also be done
using direct serial reads and writes instead of through libusb.

The following three examples are equivalent assuming three miners are connected:

	# Using libusb
	./cgminer --scrypt --zeus-chips 96 --zeus-clock 328
	
	# Direct serial I/O, auto-detect ports (Linux only)
	./cgminer --scrypt --zeus-chips 96 --zeus-clock 328 --scan-serial zeus:auto
	
	# Direct serial I/O, manual port specification
	./cgminer --scrypt --zeus-chips 96 --zeus-clock 328 --scan-serial /dev/ttyUSB0 \
		--scan-serial /dev/ttyUSB1 --scan-serial /dev/ttyUSB2

The different methods are provided to accomodate different systems or preferences
and as fallback.

### Device Identification ###

When using serial I/O the ID for use with `--zeus-options` is the full path name
of the serial port (eg: /dev/ttyUSB0). If using libusb the ID is the miner's
USB serial number, which can be found by running `./cgminer --ndevs` or on Linux
by examining the output of `lsusb`. The ID is also shown on each device's status
line on the main screen of cgminer.

Note: Early hardware versions (at least batch 1) used a different USB-Serial chip
which did not provide a valid serial number. In that case the ID used in CGMiner
is the USB bus and device address in this format: `<bus number>:<device address>`

Chip count for different models: Blizzard: 6, Cyclone: 96

Zeus driver is based on [documentation][zeus] and the official reference implementation.
Many thanks also to sling00 and LinuxETC for providing access to test hardware.

[zeus]: <http://zeusminer.com/user-manual-ver-1-0/>

- - - - - - - -

## Gridseed ##

	./autogen.sh
	./configure --enable-scrypt --enable-gridseed
	make

GC3355-specific options can be specified via `--gridseed-options` or
`"gridseed-options"` in the configuration file as a comma-separated list of
sub-options:

* baud - miner baud rate (default 115200)
* freq - any frequency multiple of 12.5 MHz, non-integer frequencies rounded up (default 600)
* pll_r, pll_f, pll_od - fine-grained frequency tuning; see below
* chips - number of chips per device (default 5)
* per_chip_stats - print per-chip nonce generations and hardware failures
* start_port - first port number for scrypt proxy mode (default 3350); see below
* voltage - switch the voltage to the GC3355 chips; see below
* led_off - turn off the LEDs on the Gridseed miner

When mining scrypt-only this version of cgminer does not initialize the SHA cores so that
power usage is low. On a 5-chip USB miner, power usage is around 10 W.

Gridseed support is based largely on the original [Gridseed CGMiner][] and
[dtbartle][]'s scrypt modifications.

[Gridseed CGMiner]: <https://github.com/gridseed/usb-miner/>
[dtbartle]: <https://github.com/dtbartle/cgminer-gc3355/>

### Frequency Tuning ###

If `pll_r/pll_f/pll_od` are specified, freq is ignored, and calculated as follows:
* Fin = 25
* Fref = int(Fin / (pll_r + 1))
* Fvco = int(Fref * (pll_f + 1))
* Fout = int(Fvco / (1 << pll_od))
* freq = Fout

### Dual Mining ###

When dual-mining `start_port` will set the listening proxy port of the first gridseed
device on the SHA256 instance of cgminer, with additional miners using successive ports.
The scrypt instance of cgminer will attempt to connect starting at this port.

When dual mining, start the SHA mining instance of cgminer first, wait for it to begin
mining and then start the scrypt version. The second instance will detect that the USB
ports are in use and will attempt to connect to the first via UDP.

If everything is working the same devices will appear in both cgminer windows.

### Voltage Modding ###

If `voltage=1` is set the gridseed chips will be switched to an alternate voltage.
Specifically, this flag will cause the MCU to assert the VID0 input to the voltage
regulator. This *requires* a voltmodded miner. On a stock unit this will actually
reduce the regulator's output voltage.

### More Complex Options ###

The options can also be specified for each device individually by serial number via
`--gridseed-freq` or `--gridseed-override` or their configuration file equivalents.
`--gridseed-freq` takes a comma-separated list of serial number to frequency mappings
while `--gridseed-override` takes the same format as `--gridseed-options`:

	--gridseed-freq "<SERIAL>=<FREQ>[,...]"

	--gridseed-override "<SERIAL>:freq=<FREQ>,voltage=<0/1>[,...];<SERIAL>:freq=<FREQ>[,...[;...]]"

