# Extracting bits from flash

The kernel (assumed <= 4MiB by u-boot loader)

dd if=/dev/mmcblk0 bs=1024 skip=1024 count=4096

It is a u-boot legacy uImage.  Drop the first 64 bytes to get a
standard zImage.

U-boot (assumed <= 767KiB by layout)

dd if=/dev/mmcblk0 bs=1024 skip=1 count=767

U-boot environment (u-boot code is set for 128KiB)

dd if=/dev/mmcblk0 bs=1024 skip=768 count=128

# Connecting

http://www.cyberciti.biz/hardware/5-linux-unix-commands-for-connecting-to-the-serial-console/

The stty command can be used to configure the serial port if raw
reading/writing is desired

stty -F /dev/ttyUSB0 115200 raw pass8

The cu command gives an old school connection

cu -l /dev/ttyUSB0 -s 115200

or just use screen for something easy

screen /dev/ttyUSB0 115200

ctrl+a k -- quit
ctrl+a H -- record
ctrl+a : !!!sx -kb uImage -- upload

Receiving files via rx in Linux requires it to be directly connected.
If it is through a getty it will fail (due to escaping?).  X-modem
doesn't have error detection, so always a good idea to check a hash
after (have had correuption in many larger transfers).

#::respawn:/sbin/getty -L ttymxc0 115200 vt100
::askfirst:-/bin/sh

# U-boot

The default u-boot environment loads 4MiB from mmcblk0+1MiB and boots
it with root=/dev/mmcblk0p1.  It seems likely that holding down an
external button ((e.g., backlight for Glo and Aura) will switch this
to root=/dev/mmcblk0p2.  This image re-images mmcblk0p1 and mmcblk0p2.

To do more it is nessesary to change bootdelay=0 to bootdelay=1 (or a
larger number) in the u-boot environment.  This will cause u-boot to
wait 1 second before running the bootcmd.  If any data is received on
the serial port in this time, it will drop to the u-boot command line.

## Watchdog

There is a watchdog that will kill your session before you have a
chance to do much.  From the Linux kernel code (pmic_core_i2c.c) it
seems you have to send command 22 with argument 0xff to the msp430
over the i2c bus (from pmic_core_i2c.c) to disable it.

Probing the bus gives a list of devices

i2c probe

Linux pulls the firmware version it reports (d726) from register 0.
Reading address 0 from the i2c reveals it is chip 0x43.

i2c md 0x43 0x0.1 0x2

This should then do the watchdog disable as the kernel does it

i2c mm 0x43 0x16.1   # msp430 is i2c device 0x43 and 0x16 is the command
0xff                 # argument is 0xff
0x2a                 # lsb of crc-ccit 0xffff
0xaa                 # msb of crc-ccit 0xffff
.

In practice though it seems the initial bus probe is sufficient.

## Modifying the environment

The environment is 128KiB and stored a mmcblk0+768KiB.  It consists of
a (little endian) crc32 checksum followed by a series of NULL
seperated key=value pairs.  It is NULL padded to the full length
(minus the checksum).  The checksum is over the values and padding.

A default environment emedded in u-boot (mmcblk0+0x22dd8) is used if
it does not match.  It has bootdelay=1 set and so can be interrupted.
A simple way to get into u-boot then is to invalided the checksum.  It
can then be setup in u-boot and saved with the writeenv command.

It can also be created from a key=value CR separated pairs file

keyvals=""
while read line; do
  keyvals="${keyvals}$(echo -n "$line" | xxd -p)00"
done < config.ascii
keyvals="${keyvals//$'\n'/}"

padding="00"
for ((i=0; i<17; i++)); do
  padding="${padding}${padding}"
done

output="${keyvals}${padding:${#keyvals}+8}"
{ crc32 <(echo "$output" | xxd -r -p) | tac -r -s '..'
  echo "$output"
} | xxd -r -p > config.raw

dd if=config.raw of=/dev/mmcblk0 bs=1024 seek=768 count=128

Kobo updates will update the u-boot and the kernel areas but does not
touch the environment.  The default environment normal boots with root
set to mmcblk0p1.  Holding a specific external button while booting
(e.g., backlight for Glo and Aura) should switch this to mmcblk0p2.

The default mmcblk0p2 rootfs image re-images mmcblk0p1 and mmcblk0p2.

## A better u-boot

The builtin boot command does the following

mmc read 2 ${loadaddr} 0x800 0x2000

It is possible, however, to directly load a specific image off the
onboard mnt partition (it has to be FAT as the ext2 commands just
silently fail on the ext4 filesystem)

fatload mmc 2:3 ${loadaddr} uImage  # internal storage

or a plugged in SD card (the external SD mmc device number varies:
Aura=0, Glo=1, etc.).  The command `fatls mmc 0:3` with an optional
directory will list the contents of the file system.  Changing

bootcmd=run bootcmd_mmc

to

bootcmd=run bootargs_base bootargs_mmc; load_ntxkernel; mmc rescan 0; fatload mmc 0:3 ${loadaddr} uImage; bootm

will cause it to use uImage from the FAT formated 3rd partition on the
external SD disk as the kernel if it exists and fall back to the
builtin one otherwise.  This gives a way to play with kernels without
having to open up the Kobo and solder onto the serial port.

An even more powerful option is to have it try and load a u-boot
script for the external SD drive and execute it.

run bootargs_base bootargs_mmc; load_ntxkernel; mmc rescan 0; fatload mmc 0:3 0x70400000 uScript; source 0x70400000; bootm

The script is assembled using mkImage

mkimage -A arm -O linux -T script -C none -a 0 -e 0 -n 'u-boot script' -d script uScript

The load and source commands fall through if uScript cannot be found,
and the boot boots as normal.  The advantage here is arbitrary u-boot
commands can be run.  This means, for example, root=/dev/mmcblk1p3
could be passed so everything will run off the external SD card making
it possible to leave the Kobo image entirely untouched.

An example script that optionally override the kernel from a uImage
file (note again that external SD mmc device number varies) if it
exists and boot with root=/dev/mmcblk1p1 (see bootargs_SD) is

run bootargs_base bootargs_SD; fatload mmc 0:3 ${loadaddr} uImage

# Patching kernel

The official kobo provided kernel can be downloaded from here

https://github.com/kobolabs/Kobo-Reader/tree/master/hw/imx507-aura

The following configuration options have to be enabled

CONFIG_USB_EHCI_ARC_OTG=y
CONFIG_USB_EHCI_FSL_UTMI=y

It may be desirable to enable additional debugging as well

CONFIG_USB_DEBUG=y
CONFIG_USB_ANNOUNCE_NEW_DEVICES=y

The system will still hang with "mma7660 sensor triggered ..."
followed by "usb unpluggged" unless two of the patches identified by
CazYokoyama are also applied.

--- a/arch/arm/plat-mxc/usb_common.c
+++ b/arch/arm/plat-mxc/usb_common.c
@@ -338,7 +338,7 @@ static void usbh1_set_utmi_xcvr(void)
         */
        msleep(100);
        
-#if 1
+#if 0
        // force suspend otg port 
        printk ("[%s-%d] %s() \n",__FILE__,__LINE__,__func__);
        USB_PHY_CTR_FUNC |= USB_UTMI_PHYCTRL_OC_DIS;
--- a/drivers/usb/core/hcd.c
+++ b/drivers/usb/core/hcd.c
@@ -2065,6 +2065,7 @@ irqreturn_t usb_hcd_irq (int irq, void *__hcd)
 
        if (unlikely(hcd->state == HC_STATE_HALT ||
                     !test_bit(HCD_FLAG_HW_ACCESSIBLE, &hcd->flags))) {
+                schedule_work(&hcd->wakeup_work);
                rc = IRQ_NONE;
        } else if (hcd->driver->irq(hcd) == IRQ_NONE) {
                rc = IRQ_NONE;

https://github.com/ifly7charlie/XCSoar-Kobo-Build/commits/AX8817X
https://github.com/brunotl/kernel-kobo-mx50-ntx/commits/master

CazYokoyama has an additional patch to "not activate acin_pg
interrupt, i.e.  plugin USB connector".  Likely this will get rid of
the kernel interrupt and nobody cared messages, but I haven't tried
it.

# Compiling a kernel

The kernel has to be compiled with a 4.x gcc.  This can be obtained
from the embedded Debian repo in the jessie relase (anything more
recent than jessie only have 5.x and above).  Add a source file
to /etc/apt/sources.list.d

deb http://emdebian.org/tools/debian/ jessie main

and add the armhf architecture (the jessie release requires items from
armhf) and install the 11.7 version of crossbuild-essential-armhf

dpkg --add-architecture armhf
apt-get install -t jessie crossbuild-essential-armhf

The compiler generates unaligned access as it expects the kernel to
fix them up.  The kernel isn't set to do fixups on itself though (see
https://community.nxp.com/thread/266293 for details), so this has to be
turned off by adding an option in Makefile to KBUILD_CFLAGS

KBUILD_CFLAGS += -mno-unaligned-access

From more recent versions of Perl is is nessesary to make the
suggested modification to the kernel/timeconst.pl file as technically
defined(@...) doesn't check to see if the list is empty (see Perl
warnings documentation for details).

The kernel can then be built with the following command.  

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- uImage

# Uploading a kernel

The Kobo u-boot bootm command is hacked to run load_ntxkernel if it
hasn't already been run before and overwrites the kernel we
downloaded.  Run load_ntxkernel before the download to avoid this.

load_ntxkernel
loady
~+sx -kb uImage
setenv bootargs console=ttymxc0,115200 rootwait rw no_console_suspend lpj=3997696
imi
bootm

The hacked bootm command also adds several additional parameters if
they are not already set (this can be seen in the kernel log).  This
includes appending mx50_1GHz to the end which screws up using
init=/bin/sh in an emergency.  This can be worked around with

init=/bin/sh -c sh
