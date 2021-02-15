This document describes known methods of flashing BIOS on Lenovo ThinkPads
without external programmer. The main goal is flashing coreboot while running
stock BIOS, but it may be used for flashing modified vendor BIOS as well.


# Table of Contents

- [Ivy Bridge series (X230, T430, etc.)](#ivy-bridge-series-x230-t430-etc)
  - [Introduction](#ivy-bridge-series-x230-t430-etc)
  - [Requirements](#requirements)
  - [BIOS versions](#bios-versions)
  - [Downgrading BIOS](#downgrading-bios)
    - [Enabling UEFI mode](#enabling-uefi-mode)
  - [Examining protections (theory)](#examining-protections-theory)
  - [Creating a backup](#creating-a-backup)
    - [Restoring](#restoring)
  - [Removing protections (practice)](#removing-protections-practice)
  - [About Intel ME](#about-intel-me)
- [Sandy Bridge series (X220, T420, etc.): WIP](#sandy-bridge-series-x220-t420-etc-wip)
- [Penryn and Nehalem series (X200, T400, X201, T410, etc.): WIP](#penryn-and-nehalem-series-x200-t400-x201-t410-etc-wip)
- [Common methods](#common-methods)
  - [HDA_SDO/GPIO33](#hda_sdogpio33)
- [Credits](#credits)
- [License](#license)

# Ivy Bridge series (X230, T430, etc.)

> A guide for Ivy Bridge ThinkPads has also been merged to coreboot
  documentation, [see here](https://doc.coreboot.org/mainboard/lenovo/ivb_internal_flashing.html).

Old versions of stock BIOS for these models have several security issues. In
context of this guide, two of them are of interest.

**First** is the fact the SMM_BWP and BLE are not enabled in BIOS versions
released before 2014. I have tested many versions on T430 and X230 and found out
that SMM_BWP=1 only since the update, the changelog of which contains following
line:

> (New) Improved the UEFI BIOS security feature.

**Second** is [S3 Boot Script vulnerability](https://support.lenovo.com/eg/ru/product_security/s3_boot_protect),
that was discovered and fixed later.

## Requirements

- USB drive (in case you need to downgrade BIOS). There were reports that
  GPT-partitioned drive didn't work and the fix was to change partition table to
  MBR.
- Linux install that (can be) loaded in UEFI mode.
- [CHIPSEC](https://github.com/chipsec/chipsec)
- Recent [flashrom](https://flashrom.org). At least version 1.0 is required,
  older versions of flashrom do not have `--ifd` option. (Workaround by using
  layout file is possible, but it's more complicated.)
- `iomem=relaxed` kernel parameter for using internal programmer.

## BIOS versions

Below is a table of BIOS versions that are vulnerable enough for our goals, per
model. The version number means that you need to downgrade to that or earlier
version.

| Model      | BIOS version |
|------------|--------------|
| X230       | 2.60         |
| X230t      | 2.58         |
| T430       | 2.64         |
| T430s      | 2.59         |
| T530       | 2.60         |
| W530       | 2.58         |

If your BIOS version is equal or lower, skip to the
**[Examining protections](#examining-protections-theory)** section. If not, go
through the downgrade process, described next.

## Downgrading BIOS

Go to the Lenovo web site and download BIOS Update Bootable CD for your machine
of needed version (see above).

Lenovo states that BIOS has "security rollback prevention", meaning once you
update it to some version X, you will not be able to downgrade it to pre-X
version. That's not true. It seems that this is completely client-side
restriction in flashing utilities (both Windows utility and Bootable CD). You
just need to call `winflash.exe` or `dosflash.exe` directly. Therefore you need
to modify the bootable CD image you just downloaded.

Extract an El Torito image:
```
geteltorito -o ./bios.img g1uj41us.iso
```
Mount the partition in that image:
```
sudo mount -t vfat ./bios.img /mnt -o loop,offset=16384
```
List files, find the `AUTOEXEC.BAT` file and the `FLASH` directory:
```
ls /mnt
ls /mnt/FLASH
```

Inside the `FLASH` directory, there should be a directory called `G1ET93WW` or
similar (exact name depends on your ThinkPad model). See what's inside:
```
ls /mnt/FLASH/G1ET93WW
```
There must be a file with `.FL1` extension called `$01D2000.FL1` or something
similar.

Now open the `AUTOEXEC.BAT` file:
```
sudo vim /mnt/AUTOEXEC.BAT
```
You will see a list of commands:
```
@ECHO OFF
PROMPT $p$g
cd c:\flash
command.com
```
Replace the last line (`command.com`) with this (change path to the `.FL1` file
according to yours):
```
dosflash.exe /sd /file G1ET93WW\$01D2000.FL1
```

Save the file, then unmount the partition:
```
sudo umount /mnt
```

Write this image to a USB drive (replace `/dev/sdX` with your USB drive device
name):
```
sudo dd if=./bios.img of=/dev/sdX bs=1M
```

Now reboot and press F1 to enter BIOS settings. Open the **Startup** tab and set
the startup mode to **Legacy** (or **Both**/**Legacy First**):

<a href="/screenshots/bios-legacy-only.jpg"><img src="/screenshots/bios-legacy-only.jpg" width="250px" /></a>
<a href="/screenshots/bios-legacy-first.jpg"><img src="/screenshots/bios-legacy-first.jpg" width="250px" /></a>

Press F10 to save changes and reboot.

Now, before you process, make sure that AC adapter is connected! If your battery
will die during the process, you'll likely need external programmer to recover.

Boot from the USB drive (press F12 to select boot device), and BIOS flashing
process should begin:

<a href="/screenshots/flashing1.jpg"><img src="/screenshots/flashing1.jpg" width="250px" /></a>
<a href="/screenshots/flashing2.jpg"><img src="/screenshots/flashing2.jpg" width="250px" /></a>

It may reboot a couple of times in the process. Do not interrupt it.

#### Enabling UEFI mode

When downgrade is completed, go back to the BIOS settings and set startup mode
to **UEFI** (or **Both**/**UEFI First**). This is required for vulnerability
exploitation.

<a href="/screenshots/bios-uefi-only.jpg"><img src="/screenshots/bios-uefi-only.jpg" width="250px" /></a>
<a href="/screenshots/bios-uefi-first.jpg"><img src="/screenshots/bios-uefi-first.jpg" width="250px" /></a>

Then boot to your system and make sure that `/sys/firmware/efi` or
`/sys/firmware/efivars` exist.

## Examining protections (theory)

There are two main ways that Intel platform provides to protect BIOS chip:
- **BIOS_CNTL** register of LPC Interface Bridge Registers (accessible via PCI
  configuration space, offset 0xDC). It has:
  * **SMM_BWP** (*SMM BIOS Write Protect*) bit. If set to 1, the BIOS is
    writable only in SMM. Once set to 1, cannot be changed anymore.
  * **BLE**  (*BIOS Lock Enable*) bit. If set to 1, setting BIOSWE to 1 will
    raise SMI. Once set to 1, cannot be changed anymore.
  * **BIOSWE** (*BIOS Write Enable*) bit. Controls whether BIOS is writable.
    This bit is always R/W.
- SPI Protected Range Registers (**PR0**-**PR4**) of SPI Configuration Registers
  (SPIBAR+0x74 - SPIBAR+0x84). Each register has bits that define protected
  range, plus WP bit, that defines whether write protection is enabled.

  There's also **FLOCKDN** bit of HSFS register (SPIBAR+0x04) of SPI
  Configuration Registers. When set to 1, PR0-PR4 registers cannot be written.
  Once set to 1, cannot be changed anymore.

To be able to flash, we need SMM_BWP=0, BIOSWE=1, BLE=0, FLOCKDN=0 or SPI
protected ranges (PRx) to have a WP bit set to 0.

Let's see what we have. Make sure that you have
[enabled and booted in UEFI mode](#enabling-uefi-mode), then examine HSFS
register:
```
sudo chipsec_main -m chipsec.modules.common.spi_lock
```
You should see that FLOCKDN=1:
```
[x][ =======================================================================
[x][ Module: SPI Flash Controller Configuration Locks
[x][ =======================================================================
[*] HSFS = 0xE009 << Hardware Sequencing Flash Status Register (SPIBAR + 0x4)
    [00] FDONE            = 1 << Flash Cycle Done
    [01] FCERR            = 0 << Flash Cycle Error
    [02] AEL              = 0 << Access Error Log
    [03] BERASE           = 1 << Block/Sector Erase Size
    [05] SCIP             = 0 << SPI cycle in progress
    [13] FDOPSS           = 1 << Flash Descriptor Override Pin-Strap Status
    [14] FDV              = 1 << Flash Descriptor Valid
    [15] FLOCKDN          = 1 << Flash Configuration Lock-Down
```


Then check BIOS_CNTL and PR0-PR4:
```
sudo chipsec_main -m common.bios_wp
```
Good news: on old BIOS versions, SMM_BWP=0 and BLE=0.

Bad news: there are 4 write protected SPI ranges:

```
[x][ =======================================================================
[x][ Module: BIOS Region Write Protection
[x][ =======================================================================
[*] BC = 0x 8 << BIOS Control (b:d.f 00:31.0 + 0xDC)
    [00] BIOSWE           = 0 << BIOS Write Enable
    [01] BLE              = 0 << BIOS Lock Enable
    [02] SRC              = 2 << SPI Read Configuration
    [04] TSS              = 0 << Top Swap Status
    [05] SMM_BWP          = 0 << SMM BIOS Write Protection
[-] BIOS region write protection is disabled!

[*] BIOS Region: Base = 0x00500000, Limit = 0x00BFFFFF
SPI Protected Ranges
------------------------------------------------------------
PRx (offset) | Value    | Base     | Limit    | WP? | RP?
------------------------------------------------------------
PR0 (74)     | 00000000 | 00000000 | 00000000 | 0   | 0
PR1 (78)     | 8BFF0B40 | 00B40000 | 00BFFFFF | 1   | 0
PR2 (7C)     | 8B100B10 | 00B10000 | 00B10FFF | 1   | 0
PR3 (80)     | 8ADE0AD0 | 00AD0000 | 00ADEFFF | 1   | 0
PR4 (84)     | 8AAF0800 | 00800000 | 00AAFFFF | 1   | 0
```

Other way to examine SPI configuration registers is to just dump SPIBAR:
```
sudo chipsec_util mmio dump SPIBAR
```

You will see SPIBAR address (0xFED1F800) and registers (for example, 00000004 is
HSFS):
```
[mmio] MMIO register range [0x00000000FED1F800:0x00000000FED1F800+00000200]:
+00000000: 0BFF0500
+00000004: 0004E009
...
```
As you can see, the only thing we need is to unset WP bit on PR0-PR4. But that
cannot be done once FLOCKDN is set to 1.

Now the fun part!

FLOCKDN may only be cleared by a hardware reset, which includes S3 state. On S3
resume boot path, the chipset configuration has to be restored and it's done by
executing so-called S3 Boot Scripts. You can dump these scripts by executing:
```
sudo chipsec_util uefi s3bootscript
```
There are many entries. Along them, you can find instructions to write to HSFS
(remember, we know that SPIBAR is 0xFED1F800):
```
Entry at offset 0x2B8F (len = 0x17, header len = 0x0):
Data:
02 00 17 02 00 00 00 01 00 00 00 04 f8 d1 fe 00 |
00 00 00 09 e0 04 00                            |
Decoded:
  Opcode : S3_BOOTSCRIPT_MEM_WRITE (0x0002)
  Width  : 0x02 (4 bytes)
  Address: 0xFED1F804
  Count  : 0x1
  Values : 0x0004E009
```
These scripts are stored in memory. The vulnerability is that we can overwrite
this memory, change these instructions and they will be executed on S3 resume.
Once we patch that instruction to not set FLOCKDN bit, we will be able to write
to PR0-PR4 registers.

## Creating a backup

Before you proceed, please create a backup of the `bios` region. Then, in case
something goes wrong, you'll be able to flash it back externally.

The `me` region is locked due to active ME, so an attempt to create a full dump
will fail. But you can back up the `bios`:
```
sudo flashrom -p internal -r bios_backup.rom --ifd -i bios
```

#### Restoring
If you will even need to flash it back, use `--ifd -i bios` as well:
```
sudo flashrom -p <YOUR_PROGRAMMER> -w bios_backup.rom --ifd -i bios
```
**Important:** if you will omit `--ifd -i bios` for flashing, you will brick
your machine, because your backup has `FF`s in place of `fd` and `me` regions.
Flash only `bios` region!

## Removing protections (practice)
The original boot script writes 0xE009 to HSFS. FLOCKDN is 15th bit, so let's
write 0x6009 instead:

Run this:
```
sudo chipsec_main -m tools.uefi.s3script_modify -a replace_op,mmio_wr,0xFED1F804,0x6009,0x2
```
You will get a lot of output and in the end you should see something like this:
```
[*] Modifying S3 boot script entry at address 0x00000000DAF49B8F..
[mem] 0x00000000DAF49B8F
[*] Original entry:
 2  0 17  2  0  0  0  1  0  0  0  4 f8 d1 fe  0 |
 0  0  0  9 e0  4  0                            |
[mem] buffer len = 0x17 to PA = 0x00000000DAF49B8F
 2  0 17  2  0  0  0  1  0  0  0  4 f8 d1 fe  0 |
 0  0  0  9 60  0  0                            |     `
[mem] 0x00000000DAF49B8F
[*] Modified entry:
 2  0 17  2  0  0  0  1  0  0  0  4 f8 d1 fe  0 |
 0  0  0  9 60  0  0                            |     `
[*] After sleep/resume, check the value of register 0xFED1F804 is 0x6009
[+] PASSED: The script has been modified. Go to sleep..
```
Now go to S3, then resume and check FLOCKDN. It should be 0:
```
sudo chipsec_main -m chipsec.modules.common.spi_lock
```
```
...
[x][ =======================================================================
[x][ Module: SPI Flash Controller Configuration Locks
[x][ =======================================================================
[*] HSFS = 0x6008 << Hardware Sequencing Flash Status Register (SPIBAR + 0x4)
    [00] FDONE            = 0 << Flash Cycle Done
    [01] FCERR            = 0 << Flash Cycle Error
    [02] AEL              = 0 << Access Error Log
    [03] BERASE           = 1 << Block/Sector Erase Size
    [05] SCIP             = 0 << SPI cycle in progress
    [13] FDOPSS           = 1 << Flash Descriptor Override Pin-Strap Status
    [14] FDV              = 1 << Flash Descriptor Valid
    [15] FLOCKDN          = 0 << Flash Configuration Lock-Down
[-] SPI Flash Controller configuration is not locked
[-] FAILED: SPI Flash Controller not locked correctly.
...
```
Set BIOSWE:
```
sudo setpci -s 00:1f.0 dc.b=09
```
Remove WP from protected ranges:
```
sudo chipsec_util mmio write SPIBAR 0x74 0x4 0xAAF0800
sudo chipsec_util mmio write SPIBAR 0x78 0x4 0xADE0AD0
sudo chipsec_util mmio write SPIBAR 0x7C 0x4 0xB100B10
sudo chipsec_util mmio write SPIBAR 0x80 0x4 0xBFF0B40
```
Verify that it worked:
```
sudo chipsec_main -m common.bios_wp
```
```
[x][ =======================================================================
[x][ Module: BIOS Region Write Protection
[x][ =======================================================================
[*] BC = 0x 9 << BIOS Control (b:d.f 00:31.0 + 0xDC)
    [00] BIOSWE           = 1 << BIOS Write Enable
    [01] BLE              = 0 << BIOS Lock Enable
    [02] SRC              = 2 << SPI Read Configuration
    [04] TSS              = 0 << Top Swap Status
    [05] SMM_BWP          = 0 << SMM BIOS Write Protection
[-] BIOS region write protection is disabled!

[*] BIOS Region: Base = 0x00500000, Limit = 0x00BFFFFF
SPI Protected Ranges
------------------------------------------------------------
PRx (offset) | Value    | Base     | Limit    | WP? | RP?
------------------------------------------------------------
PR0 (74)     | 0AAF0800 | 00800000 | 00AAF000 | 0   | 0
PR1 (78)     | 0ADE0AD0 | 00AD0000 | 00ADE000 | 0   | 0
PR2 (7C)     | 0B100B10 | 00B10000 | 00B10000 | 0   | 0
PR3 (80)     | 0BFF0B40 | 00B40000 | 00BFF000 | 0   | 0
PR4 (84)     | 00000000 | 00000000 | 00000000 | 0   | 0
```

Bingo!

Now you can flash coreboot (or anything else) with flashrom.

Remember to flash only `bios` region (use `--ifd -i bios -N`). `fd` and `me` are
still not writable.

**Note:** if you're flashing coreboot for the first time, you should have an
external SPI programmer just in case. It will help you recover if you flash
non-working ROM.

## About Intel ME

Unlocking PRx doesn't allow you to flash ME or Flash Descriptor, so you cannot
use [me_cleaner](https://github.com/corna/me_cleaner). Both `me` and `fd`
regions will still be write-protected even if you flash coreboot. For now, the
best way to get rid of ME is to flash me_cleaned firmware externally.

As an alternative, you can use Soft Temporary Disable Mode. This is ME's
built-in mechanism that BIOS can use to "ask" ME to disable itself on the next
boot. In this mode ME doesn't load its kernel and stops at BUP phase. This mode
is saved to ME NVRAM and thus preserved between reboots and poweroffs. Working
patch for coreboot is [available](https://review.coreboot.org/c/coreboot/+/37115).

Some possible future solutions are described below.

#### HMRFPO (updated: won't work)

**HMRFPO** (Host ME Region Flash Protection Override) is a command BIOS can use
to unlock ME region on SPI flash for writing. Vendor can disable it during
manufacturing. Unfortunately it's disabled on ThinkPads and can't be used.

#### HDA_SDO (not implemented)

It is possible to make flash descriptor writable for one boot by asserting
HDA_SDO pin high on boot. It should be possible to flash unlocked descriptor
while in this state, and that should open ability to flash ME afterwards.
[See here for more info](#hda_sdogpio33).

# Sandy Bridge series (X220, T420, etc.): WIP
S3 Boot Scripts are unprotected on these models too (even on the most recent
BIOS versions), but it's not useful, because FLOCKDN and SPI protected ranges
are set by **LenovoFlashProtectPei** UEFI module. It is trivial to patch it, but
it resides in protected range, so it can only be flashed externally.

Currently there are no known methods to unlock PRs on these devices internally,
but investigation is ongoing.


# Penryn and Nehalem series (X200, T400, X201, T410, etc.): WIP
Currently, the only known way to flash these models without external SPI
programmer is by using GPIO33. It is not properly documented yet, and no scripts
to automate the process were written. There was a success story though: a user
from #coreboot/#libreboot IRC channels has successfully corebooted his X201
without touching the SPI chip.

Progress is tracked [here](https://notabug.org/libreboot/libreboot/issues/689).

# Common methods

## HDA_SDO/GPIO33
There's known Intel "feature", sometimes called "pinmod": you can make flash
descriptor writable for one boot by asserting HDA_SDO pin on the motherboard
high (on older platforms, you need to assert GPIO33 pin low) at boot time.
**Writable flash descriptor** sometimes **allows to flash whole BIOS region**
using a couple of neat tricks.  I will show this using X220 as an example.

This is SPI flash layout on X220:
```
00000000:00000fff fd
00500000:007fffff bios
00003000:004fffff me
00001000:00002fff gbe
```
Lenovo BIOS protects, using PR0 register, only small (but critical) part of it:
`0x00780000-0x1ffffff`.

> PR0 defines the end of protected range as `0x1ffffff`, but since it's 8MB chip
and `bios` ends at `0x007fffff`, `0x00780000-0x007fffff` is what actually is
protected. This is the last 512K.

As you can see, `0x00500000-0x0077ffff` part is totally writable.

Since flash descriptor is writable too, here is what we can do:
- patch flash descriptor:
  - redefine `bios` as `00500000:0077ffff` using ifdtool;
  - define the rest of it (`00780000:007fffff`) as `pd` (Platform Data). This is
    **required**, otherwise you will not be able to flash it in future;
  - unlock FD and ME regions (`ifdtool -u`);
- build and flash (small enough) coreboot ROM to this region (must be careful
  and make sure that reset vector is at the end of our custom region and not at
  the end of 8MB ROM; use dd or ifdtool for that);
- flash modified descriptor;
- power off (new FD takes effect after cold boot);
- wait a few seconds and power on.

After this, new FD will be in effect and coreboot will be loaded from
`0x00500000-0x0077ffff` region, instead of Lenovo BIOS. At this point whole
`0x00500000-0x007fffff` will be writable and you will be able to flash coreboot
again or patched vendor BIOS (don't forget to revert `fd` changes regarding the
`bios` region).

A user from #coreboot/#libreboot IRC channels was able to flash his X201
internally using the idea described above. However, this is not for mass use
yet: we need to write scripts to automate the process, locate HDA_SDO or GPIO33
pins on all motherboards in interest, take their photos and explain, what to do
with what.

# Credits
**Rafal Wojtczuk** and **Corey Kallenberg** for discovering the S3 Boot Scripts
vulnerability

[pgera](https://github.com/hamishcoleman/thinkpad-ec/issues/70#issuecomment-417903315)
for the initial research and working solution

Lenovo for fake rollback protection

# License

This documented is licensed under [CC-BY 4.0](https://creativecommons.org/licenses/by/4.0/)
terms.
