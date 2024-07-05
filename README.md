## edid-checked-writer: A utility to read and write a display EDID value

This is a fork of Mark Blakeney's https://github.com/bulletmark/edid-rw updated
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

    $ sudo apt-get install python3-smbus edid-decode i2c-tools

Or, install these prerequisites on Arch:

    $ sudo yay -S i2c-tools edid-decode-git

Get this source code:

    $ curl -L https://github.com/galkinvv/edid-checked-writer/archive/master.tar.gz | tar xvz
    $ cd edid-checked-writer-master

### Usage

Run without parameters to list available i2c buses and see usage and optional arguments:

    $ sudo ./edid-rw

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

Fetch and decode display attached to i2c-0 EDID data:

    $ sudo ./edid-rw 0 | edid-decode

Fetch and decode display attached to i2c-16 EDID data:

    $ sudo ./edid-rw 16 | edid-decode

Below is basic example of reading-modifying-writing EDID for disaply attached to i2c-9 bus:
use `!Gxxd [-r]` within vim to read, edit, and write binary
file. See `:h xxd` within vim help. You should set the checksum (last)
byte correctly although edit-rw will calculate and set the checksum
itself if you include the `-f (--fix)` switch. edid-rw will always
validate the checksum and will not write an invalid EDID:

*WARNING - Be sure to triple check the EDID bus you are about to
write!*

    $ sudo ./edid-rw 9 >edid.bin
    $ vim -b edid.bin # Then use xxd within vim, see ":h xxd" in vim
    $ sudo ./edid-rw -w 9 <edid.bin # ~10seconds write+verify stage
    Provided file is valid EDID, checking original EDID in a EEPROM...
    Writing new EDID to EEPROM...
    OK: New EDID written and verified

If you are unsure in EDID editing, you may try replacing your EDID with variant
from another monitor - there is a great collection at https://github.com/linuxhw/EDID
Note that if external files are in text format, like the link above - they should be converted to binary 256-byte form before writing them to EEPROM.
Typically, this is done by running reversed-hex-dumping on a part of a text file containing the hex representation.

### License

Copyright (C) 2012 Mark Blakeney. This program is distributed under the
terms of the GNU General Public License.

This program is free software: you can redistribute it and/or modify it
under the terms of the GNU General Public License as published by the
Free Software Foundation, either version 3 of the License, or any later
version.

This program is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General
Public License at <http://www.gnu.org/licenses/> for more details.
