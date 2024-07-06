## edid-checked-writer: A utility to read and write a display EDID value

This is a fork of Mark Blakeney's [edid-rw](https://github.com/bulletmark/edid-rw) updated
to perform verification after writing EDID.

The binary name itself kept being `./edid-rw` for commands compatibility.

### Overview

This utility will read and/or write a display's EDID data structure. Use
it with the edid-decode utility to view and check an EDID.
You can also write new EDID data to attempt to fix a corrupt EDID.

*WARNING - THIS UTILITY CAN DESTROY YOUR DISPLAY, MOTHERBOARD, OR OTHER
CONNECTED HARDWARE IF RUN INCORRECTLY. Be very sure you understand what
you are doing. See [this issue](http://github.com/bulletmark/edid-rw/issues/5)
for an example of what can happen.*

You may have to disable output to the display before you can write the
EDID.

### Installation

Requires:
 * python3 smbus module
 * edid-decode, i2cdetect utilities
 * drm-capable graphics driver loaded for the GPU

Install these prerequisites on Debian/Ubuntu:

    sudo apt-get install python3-smbus edid-decode i2c-tools

Or, install these prerequisites on Arch:

    sudo yay -S i2c-tools edid-decode-git

Get this source code:

    curl -L https://github.com/galkinvv/edid-checked-writer/archive/master.tar.gz | tar xvz
    cd edid-checked-writer-master

### Usage

Run without parameters to list available i2c buses and see usage and optional arguments:

    sudo ./edid-rw

All HDMI/DVI/VGA(D-SUB) outputs of AMD/Intel/NVIDIA chips support EDID operations.
DisplayPort lacks EDID, so it is not supported.
There are many i2c buses in a Linux system - 
here is an example output on a sample system having HDMI outputs
available for integrated intel GPU and a pair of PCIe extension cards:
<pre>
Listing available I2C buses via `i2cdetect -l`:
i2c-0   i2c             <b>i915 gmbus dpc                          I2C adapter</b> <i># HDMI of Intel LGA1151 P10S-WS</i>
i2c-1   i2c             i915 gmbus dpb                          I2C adapter
i2c-2   i2c             i915 gmbus dpd                          I2C adapter <i># DVI of Intel LGA1151 P10S-WS</i>
i2c-3   i2c             AUX B/DDI B/PHY B                       I2C adapter
i2c-4   i2c             AUX A/DDI E/PHY E                       I2C adapter
i2c-5   i2c             AMDGPU SMU                              I2C adapter
i2c-6   i2c             AMDGPU DM i2c hw bus 0                  I2C adapter
i2c-7   i2c             AMDGPU DM i2c hw bus 1                  I2C adapter
i2c-8   i2c             AMDGPU DM i2c hw bus 2                  I2C adapter
i2c-9   i2c             <b>AMDGPU DM i2c hw bus 3                  I2C adapter</b> <i># HDMI of AMD RX6700</i>
i2c-10  i2c             AMDGPU DM aux hw bus 0                  I2C adapter
i2c-11  i2c             AMDGPU DM aux hw bus 1                  I2C adapter
i2c-12  i2c             AMDGPU DM aux hw bus 2                  I2C adapter
i2c-13  smbus           SMBus I801 adapter at f040              SMBus adapter
i2c-14  i2c             NVIDIA i2c adapter 1 at 1:00.0          I2C adapter
i2c-15  i2c             NVIDIA i2c adapter 2 at 1:00.0          I2C adapter
i2c-16  i2c             <b>NVIDIA i2c adapter 5 at 1:00.0          I2C adapter</b> <i># HDMI of NVIDIA RTX3060-3090</i>
i2c-17  i2c             NVIDIA i2c adapter 6 at 1:00.0          I2C adapter
i2c-18  i2c             NVIDIA i2c adapter 7 at 1:00.0          I2C adapter
i2c-19  i2c             NVIDIA i2c adapter 8 at 1:00.0          I2C adapter
You have to carefully select correct bus
and pass the number X from the i2c-X bus name as command line "i2c_bus_index" argument

usage: edid-rw [-h] [-w] [-t] [-f] [-s SLEEP] i2c_bus_index
</pre>
Unfortunately they all have very similar names, and lack the handcrafted italic comments like above.

So the wanted i2c-X bus is often found by trial-and-error way with trying to read & decode EDID until expected EDID is found&parsed.

The names teamplates typically corresponding to HDMI ports are marked as bold in the above output.

##### Just viewing EDID
Fetch and decode display attached to i2c-0 EDID data:

    sudo ./edid-rw 0 | edid-decode

##### Replacing with another EDID available as a binary file

Getting exact waned EDID for some monitor can be quite hard, fortunately there is a [huge collection of EDIDs by linuxhw project](https://github.com/linuxhw/EDID).
It has a [monitors model index in a separate file](https://raw.githubusercontent.com/linuxhw/EDID/master/DigitalDisplay.md) that can be used for selecting EDIDs by resolution, size and vendor.

Those EDIDs are in text format and need to be converted to binary before writing. This can be done with grep & xxd as suggested by linuxhw:

    curl https://raw.githubusercontent.com/linuxhw/EDID/master/Digital\
    /Samsung/SAM0E0C/38CD24A55A35 | grep -E '^([a-f0-9]{32}|[a-f0-9 ]{47})$' | xxd -r -p > new-edid.bin

1.Backup a EDID data of a display attached to i2c-16 and decode it to make sure that a correct monitor is attached to a selected bus

    sudo ./edid-rw 16 >original-edid.bin && edid-decode original-edid.bin

2.Write new EDID to the EEPROM (~10 seconds write+verify stage)

*WARNING - Be sure to triple check the EDID bus you are about to write!*

    sudo ./edid-rw -w 16 <new-edid.bin
    
    Provided file is valid EDID, checking original EDID in a EEPROM...
    Writing new EDID to EEPROM...
    OK: New EDID written and verified

Unfortunately, many displays has write-protection on EEPROM chips.
In this case the program will report an error

    Writing new EDID to EEPROM...
    ERROR: Failed writing new EDID, maybe device is write-protected.
    Nothing changed, existing device EDID kept unmodified

Typically, such write protection can't be removed programmatically.
However, there is quite simple non-intrusive hardware workaround:
plug simple device named  "pass-through EDID emulator" between the signal source (GPU)
and monitor cable, corresponding to used HDMI/DVI/VGA plug,
which effectively "replaces" monitor's EEPROM with emulator's EEPROM.
Then any wanted EDID can be written into emulator (most of them has no write protection).

Speaking specifically about HDMI there was success writing report with
cheapest pass-through in metal case with "Source" and "Sink" labels.
It contains 24C02-series 2Kbit serial EEPROM IC with WriteProtect pin pulled to GND.

##### Advanced: using hex editor to make custom EDID modifications
Below is a basic example of reading-modifying-writing EDID for display attached to i2c-9 bus:
use `!Gxxd [-r]` within vim to read, edit, and write binary
file. See `:h xxd` within vim help. You should set the checksum (last)
byte correctly, although edit-rw will calculate and set the checksum
itself if you include the `-f (--fix)` switch. edid-rw will always
validate the checksum and will not write an invalid EDID:

*WARNING - Be sure to triple check the EDID bus you are about to write!*

    sudo ./edid-rw 9 >original-edid.bin && cp original-edid.bin ediatble-edid.bin
    vim -b ediatble-edid.bin # Then use xxd within vim, see ":h xxd" in vim
    sudo ./edid-rw -w 9 <ediatble-edid.bin
    
    Provided file is valid EDID, checking original EDID in a EEPROM...
    Writing new EDID to EEPROM...
    OK: New EDID written and verified

### License

This program is distributed under the terms of the GNU General Public License.

This program is free software: you can redistribute it and/or modify it
under the terms of the GNU General Public License as published by the
Free Software Foundation, either version 3 of the License, or any later
version.

This program is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General
Public License at <http://www.gnu.org/licenses/> for more details.
