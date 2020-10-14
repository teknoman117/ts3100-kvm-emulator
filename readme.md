KVM Emulator for the Technologic Systems TS-3100
================================================

Disclaimer: I have absolutely no connection with this company, this project is purely as a hobby project.

The TS-3100 is an (at the time) low cost single board computer from late 2001 made by Technologic Systems (https://www.embeddedarm.com/).

Usage
-----
<pre>
# Fetch and build emulator
git clone --recurse-submodules https://github.com/teknoman117/ts31000-kvm-emulator
cd ts3100-kvm-emulator
mkdir build
# if DISASSEMBLE is enabled, each instruction will be disassembled to the console as the VM
# executes. This is very slow. Will take ~1 minute to get into the VM.
cmake -DDISASSEMBLE=OFF -DVIRTUAL_DISK=ON -DCMAKE_BUILD_TYPE=Release -B build .
cmake --build build -j

# You'll need "roms/flash.bin", which can be generated by ONE of the following methods:

# Option 1 - Building image from files from official FTP site
# Requires: curl unzip
# Script will download images and assemble to rom image as "roms/flash.bin"
bash download-flash-image.bash

# Option 2 - Dumping the image using my tool (TS-3100 only)
# Requires: nasm
# Build flashdump tool
make -C tools/flashdump
# copy "tools/flashdump/dump.com" to TS-3100
# run dump.com on TS-3100
# copy C:\FLASH.BIN from TS-3100 to "roms/flash.bin"

# Option 3 - Dumping the image using the official tool
# Requires a C: drive on the TS-3100 with >512 KiB free
# Download FLASHUP from Embedded ARM
curl -O http://ftp.embeddedarm.com/ftp/ts-x86-sbc/old-downloads/FLASHUP.ZIP
# (sha256sum: b03a0bad8866ef2160392d2dfc496167063ea549df14f7cd1e8aa87814c8c61f)
unzip FLASHUP.ZIP
# copy FLASHUP.EXE to TS-3100 C: drive
# from C:, run "FLASHUP.EXE /CREATE BIOS=BIOS.BIN DOS=DOS.BIN DISK=DISK.BIN"
# transfer files back to the host
cat DISK.BIN DOS.BIN BIOS.BIN > roms/flash.bin

# (Optional) Build virtual disk option rom from source ("roms/virtual-disk/option.rom")
# Requires: nasm python3
make -C roms/virtual-disk

# Run emulator
build/src/kvm-emulator

# In another terminal, connect to virtual COM2 port
minicom -D unix\#/tmp/3100.com2.socket

</pre>

FAQ
---

Q: Why is the text console so slow?

A: I tried my best to emulate the experience of using the board over a serial console. It calculates the print speed from the baudrate of the serial console. running "console /p:2 /s:115200" will speed it up significantly, as the default baudrate is 9600.

Q: How do I copy files to the VM?

A: The quickest way is to mount the drivec.img file with loopback and copy files. The second way is to use Zmodem via minicom. Run "dl.bat" and then hit Ctrl-A + S and choose Zmodem. Find the path to the file you wish to transfer. It will be slow. It also might hang, there are still a few bugs with the interrupt handling.

Q: How do I make my own "roms/drivec.img" image?

A: The DOS-ROM image doesn't seem to like the fat16 formatting that the dosfstools package creates. Perform the following instructions:

<pre>
# specific size because I don't support dynamic sizes yet
rm -f roms/drivec.img
truncate -s 67108864 roms/drivec.img
fdisk roms/drivec.img
# Command (m for help): o
# Command (m for help): n
# Select (default p): p
# Partition number (1-4, default 1): 1
# First sector (2048-131071, default 2048): 2048
# Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-131071, default 131071): 131071
# Command (m for help): t
# Hex code (type L to list all codes): 6
# Command (m for help): w

curl -O http://ftp.embeddedarm.com/ftp/ts-x86-sbc/old-downloads/Disks/TS-3100.ZIP
# (sha256sum: 6d8bd7119d5bdcbd63976252ea6e8611048acf8dc5b4ec67bf55db2f260a92bb)
unzip TS-3100.ZIP DOS/FORMAT.EXE

# Boot virtual machine with disk
# Copy FORMAT.EXE to TS-3100 (using Zmodem)
# Run "FORMAT.EXE C: /Q"
# C: should now be a usable FAT-16 volume
</pre>

Background
----------

I obtained these boards when I was a teenager (2006 or 2007) from a guy in the Atlanta Hobby Robotics Club. Backstory as I recall was that the company he worked for had an ordering mixup (got the cut down product on accident) and thew out a pallet of these computers. So, rather than going in a landfill, he was allowed to give them away to the robotics club. I ended up with a half dozen or so. PC/104 required more investment than I could make as a kid and eventually ended up in my parts box like so many other things. Late last year I saw Ben Eater's series on building breadboard computers and while the concept wasn't new to me, I hadn't really exercised my electronics hobby since getting into college. I thought a fun project would be hooking up one of these old computers to an FPGA board to build fun virtual devices. I also saw Phil's blog_os project on building a hobby kernel for x86_64 OS in Rust and figured a fun project would be to get his series running on the first x86 core with paged virtual memory. I'd say I'm a lot more capable now than when I was 14, so I pulled them out of my parts box and took a deep dive into how they actually worked.

The eventual goal is to replace the on board BIOS image with a custom loader, so that I can reclaim the shadowed memory and run whatever without losing onboard resources to DOS. Since I currently lack some things I need, I decided that I'd dump the flash image and see if I could get it working in a virtual machine so I could experiment with all sorts of changes.

Specifications
--------------
- 25 MHz Intel 386EX
- Chipset implemented by two Xilinx XC9500 series CPLDs.
- 512 KiB of 8 bit DRAM
  - 448 KiB at 0x00000000-0x0006FFFF (user ram)
  - 64 KiB at 0x000F0000-0x000FFFFF (reserved for BIOS shadowing)
  - Interestingly, it's implemented as a 4 bit DRAM which is driven at twice the CPU's bus speed, making 8 bits available per CPU bus cycle.
- 512 KiB of Flash
  - 384 KiB at 0x03400000-0x345FFFF (virtual A: drive)
    - 289 KiB usable (due to wear leveling and formatting)
  - 64 KiB at 0x03460000 - 0x0346FFFF (DOS-ROM)
  - 64 KiB at 0x03470000 - 0x0347FFFF (BIOS)
- 8 bit PC/104 bus (more or less an 8 bit ISA port)
- 2x COM ports provided by 386EX (16450 compatible)
  - If a MAX485 is installed, one port is RS-485 capable.
- DS1687 Real Time Clock
- Socket for either:
  - up to 32 KiB JEDEC compatible parallel memory (SRAM, ROM, EPROM, EEPROM, FLASH, etc.)
  - DiskOnChip 2000 (http://www.digital-circuitry.com/MyLAB_IC_PROG_DISKONCHIP.htm)
- GPIO
- LCD port for HD44780 compatible LCDs

The TS-3100 is a significantly cut down version of their TS-3200 product. The TS-3200 is available with 8 or 16 MiB of 16-bit SDRAM, 2 to 8 MiB of onboard flash, and a 16-bit PC/104 port. It's capable of running DOS or TS-Linux (a distro put together by Technologic Systems on the 2.4 kernel). By comparison, the TS-3100 is cut down to 512 KiB (8 bit) / 512 KiB / 8 bit respectively, leaving it only capable of running DOS. The 8 bit, slower DRAM also means it can only execute code at 40% of the speed of the TS-3200 except in tight loops (that can all live in the 386's prefetch buffer). The target market seems to have been primarily to replace aging DOS based computers in industrial applications while retaining PC compatibility. The 8 bit ISA slot limits you to 1 MiB of address space.

The startup process for the board is actually rather neat. The 386EX, being intended for embedded applications, has 8 "chip select units". They are essentially self-contained programmable address comparators for providing chip select lines without external logic. An extremely useful feature is that they can insert wait states without external logic. At boot, all of these units are disabled except unit 7, otherwise known as the "Upper Chip Select Unit", which at reset matches the entire address space of the processor. The flash memory is selected by this unit. The reset vector of all 32 bit x86 processors is 0xFFFFFFF0, bit since we're only dealing with a 19 bit address (512 KiB flash), the address seen there is 0x7FFF0. This falls in the BIOS region of the flash device. Only after copying the BIOS to the upper block of RAM does it reprogram the upper chip select unit to reside at 0x03400000 and jump to the BIOS in RAM.

Resources
---------
- Webpage (Link Dead): https://www.embeddedarm.com/products/TS-3100
- Wiki: https://docs.embeddedarm.com/TS-3100
- Manual: https://www.embeddedarm.com/documentation/ts-3100-manual.pdf
- Manual (TS-3200): https://www.embeddedarm.com/documentation/ts-3200-manual.pdf
- Schematic: https://www.embeddedarm.com/documentation/ts-3100-schematic.pdf
- Utility Disk: http://ftp.embeddedarm.com/ftp/ts-x86-sbc/old-software-pages/downloads/Disks/TS-3100.ZIP
- Technologic Systems x86 SBC Resources: http://ftp.embeddedarm.com/ftp/ts-x86-sbc/
- Intel 386EX Manual: http://bitsavers.trailing-edge.com/components/intel/80386/272485-001_80386EX_Users_Manual_Feb95.pdf
- Intel 386 Programmers' Manual: https://css.csail.mit.edu/6.858/2014/readings/i386.pdf
- KVM API: https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt
- Using the KVM API: https://lwn.net/Articles/658511/
