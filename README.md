These are my notes from hacking on my Kobo Aura, and later a Kobo Glo
too, for the purposes of using it with XCSoar.

When I started with the Aura, I opened it up and soldered a breakout
cable onto the onboard serial header on the board.  I then made my
initial connection and did a lot of fooling around/figruing out over
this serial port using Dangerous Prototypes' Bus Pirate device in
serial port mode.

Only much later did I figured out how to modify the u-boot environment
to first boot off of an sd-card if present (see below).  So, when I
got the Glo, I never bothered soldering onto the serial port.  I just
installed XCSoar, which gave me telnet root access, modified the
u-boot, and now the sd-card services as the emergency recovery.

That said, for kernel hacking, it is still nice to have the serial
port so you can see the boot messages.

# Extracting bits from flash

The kernel (assumed <= 4MiB by u-boot loader)

```bash
dd if=/dev/mmcblk0 bs=1024 skip=1024 count=4096
```

It is a u-boot legacy uImage.  Drop the first 64 bytes to get a
standard zImage.

U-boot (assumed <= 767KiB by layout)

```bash
dd if=/dev/mmcblk0 bs=1024 skip=1 count=767
```

U-boot environment (u-boot code is set for 128KiB)

```bash
dd if=/dev/mmcblk0 bs=1024 skip=768 count=128
```

# Connecting

http://www.cyberciti.biz/hardware/5-linux-unix-commands-for-connecting-to-the-serial-console/

The stty command can be used to configure the serial port if raw
reading/writing is desired

```bash
stty -F /dev/ttyUSB0 115200 raw pass8
```

The cu command gives an old school connection

```bash
cu -l /dev/ttyUSB0 -s 115200
```

or just use screen for something easy

```bash
screen /dev/ttyUSB0 115200
```

| command                | action |
|------------------------|--------|
| C^a k                  | quit   |
| C^a H                  | record |
| C^a : !!!sx -kb uImage | upload |

Receiving files via rx in Linux requires it to be directly connected.
If it is through a getty it will fail (due to escaping?).  X-modem
doesn't have error detection, so always a good idea to check a hash
after (have had correuption in many larger transfers).

```
#::respawn:/sbin/getty -L ttymxc0 115200 vt100
::askfirst:-/bin/sh
```

# U-boot

The default u-boot environment loads 4MiB from `mmcblk0`+1MiB and boots
it with `root=/dev/mmcblk0p1`.  It seems likely that holding down an
external button ((e.g., backlight for Glo and Aura) will switch this
to `root=/dev/mmcblk0p2`.  This image re-images `mmcblk0p1` and `mmcblk0p2`.

To do more it is nessesary to change `bootdelay=0` to `bootdelay=1` (or a
larger number) in the u-boot environment.  This will cause u-boot to
wait 1 second before running the `bootcmd`.  If any data is received on
the serial port in this time, it will drop to the u-boot command line.

## Watchdog

There is a watchdog that will kill your session before you have a
chance to do much.  From the Linux kernel code (pmic_core_i2c.c) it
seems you have to send command 22 with argument 0xff to the msp430
over the i2c bus (from pmic_core_i2c.c) to disable it.

Probing the bus gives a list of devices

```
i2c probe
```

Linux pulls the firmware version it reports (d726) from register 0.
Reading address 0 from the i2c reveals it is chip 0x43.

```
i2c md 0x43 0x0.1 0x2
```

This should then do the watchdog disable as the kernel does it

```
i2c mm 0x43 0x16.1   # msp430 is i2c device 0x43 and 0x16 is the command
0xff                 # argument is 0xff
0x2a                 # lsb of crc-ccit 0xffff
0xaa                 # msb of crc-ccit 0xffff
.
```

In practice though it seems the initial bus probe is sufficient.

## Modifying the environment

The environment is 128KiB and stored a `mmcblk0`+768KiB.  It consists of
a (little endian) crc32 checksum followed by a series of NULL
seperated key=value pairs.  It is NULL padded to the full length
(minus the checksum).  The checksum is over the values and padding.

A default environment emedded in u-boot (`mmcblk0`+0x22dd8) is used if
it does not match.  It has bootdelay=1 set and so can be interrupted.
A simple way to get into u-boot then is to invalided the checksum.  It
can then be setup in u-boot and saved with the writeenv command.

It can also be created from a key=value CR separated pairs file

```bash
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
```

Kobo updates will update the u-boot and the kernel areas but does not
touch the environment.  The default environment normal boots with root
set to `mmcblk0p1`.  Holding a specific external button while booting
(e.g., backlight for Glo and Aura) should switch this to `mmcblk0p2`.

The default `mmcblk0p2` rootfs image re-images `mmcblk0p1` and `mmcblk0p2`.

## A better u-boot

The builtin boot command does the following

```
mmc read 2 ${loadaddr} 0x800 0x2000
```

It is possible, however, to directly load a specific image off the
onboard mnt partition (it has to be FAT as the ext2 commands just
silently fail on the ext4 filesystem)

```
fatload mmc 2:3 ${loadaddr} uImage  # internal storage
```

or a plugged in SD card (the external SD mmc device number varies:
Aura=0, Glo=1, etc.).  The command `fatls mmc 0:3` with an optional
directory will list the contents of the file system.  Changing

```
bootcmd=run bootcmd_mmc
```

to

```
bootcmd=run bootargs_base bootargs_mmc; load_ntxkernel; mmc rescan 0; fatload mmc 0:3 ${loadaddr} uImage; bootm
```

will cause it to use uImage from the FAT formated 3rd partition on the
external SD disk as the kernel if it exists and fall back to the
builtin one otherwise.  This gives a way to play with kernels without
having to open up the Kobo and solder onto the serial port.

An even more powerful option is to have it try and load a u-boot
script for the external SD drive and execute it.

```
run bootargs_base bootargs_mmc; load_ntxkernel; mmc rescan 0; fatload mmc 0:3 0x70400000 uScript; source 0x70400000; bootm
```

The script is assembled using mkImage

```bash
mkimage -A arm -O linux -T script -C none -a 0 -e 0 -n 'u-boot script' -d script uScript
```

The load and source commands fall through if uScript cannot be found,
and the boot boots as normal.  The advantage here is arbitrary u-boot
commands can be run.  This means, for example, `root=/dev/mmcblk1p3`
could be passed so everything will run off the external SD card making
it possible to leave the Kobo image entirely untouched.

An example script that optionally override the kernel from a uImage
file (note again that external SD mmc device number varies) if it
exists and boot with `root=/dev/mmcblk1p1` (see bootargs_SD) is

```
run bootargs_base bootargs_SD; fatload mmc 0:3 ${loadaddr} uImage
```

# Patching kernel

The official kobo provided kernel can be downloaded from here

https://github.com/kobolabs/Kobo-Reader/tree/master/hw/imx507-aura

The following configuration options have to be enabled

```
CONFIG_USB_EHCI_ARC_OTG=y
CONFIG_USB_EHCI_FSL_UTMI=y
```

It may be desirable to enable additional debugging as well

```
CONFIG_USB_DEBUG=y
CONFIG_USB_ANNOUNCE_NEW_DEVICES=y
```

The system will still hang with "mma7660 sensor triggered ..."
followed by "usb unpluggged".  CazYokoyama determined the issue was
the wakeup routine wasn't running and did a bit of hack that manually
scheduled it from an interrupt service routine to get around this

https://github.com/ifly7charlie/XCSoar-Kobo-Build/commits/AX8817X

https://github.com/brunotl/kernel-kobo-mx50-ntx/commits/master

After quite a bit of digging, I determined the reason for this was
there was a bitmask screw up and the interupt determination code was
relying exclusively on the ID bit (which is permantely pulled high on
the Kobo because it is lefted unconnected)


```
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
--- a/drivers/usb/otg/fsl_otg.c
+++ b/drivers/usb/otg/fsl_otg.c
@@ -1070,7 +1070,7 @@ int usb_otg_start(struct platform_device *pdev)
 	temp = readl(&p_otg->dr_mem_map->otgsc);
 	if (!pdata->id_gpio)
 		temp |= OTGSC_INTR_USB_ID_EN;
-	temp &= ~(OTGSC_CTRL_VBUS_DISCHARGE | OTGSC_INTR_1MS_TIMER_EN);
+	temp &= ~OTGSC_CTRL_VBUS_DISCHARGE;
 
 	writel(temp, &p_otg->dr_mem_map->otgsc);
 
diff --git a/drivers/usb/otg/fsl_otg.h b/drivers/usb/otg/fsl_otg.h
index 0857a53e6fad..64b24ee53ae7 100644
--- a/drivers/usb/otg/fsl_otg.h
+++ b/drivers/usb/otg/fsl_otg.h
@@ -237,7 +237,7 @@
 /* OTG interrupt status bit masks */
 #define  OTGSC_INTERRUPT_STATUS_BITS_MASK  \
 	(OTGSC_INTSTS_USB_ID          |    \
-	OTGSC_INTR_1MS_TIMER_EN       |    \
+	OTGSC_INTSTS_1MS              |    \
 	OTGSC_INTSTS_A_VBUS_VALID     |    \
 	OTGSC_INTSTS_A_SESSION_VALID  |    \
 	OTGSC_INTSTS_B_SESSION_VALID  |    \
--- a/arch/arm/mach-mx5/usb_dr.c
+++ b/arch/arm/mach-mx5/usb_dr.c
@@ -204,18 +204,35 @@ static void _host_phy_lowpower_suspend(struct fsl_usb2_platform_data *pdata, boo
 static enum usb_wakeup_event _is_host_wakeup(struct fsl_usb2_platform_data *pdata)
 {
 	int wakeup_req = USBCTRL & UCTRL_OWIR;
-	int otgsc = UOG_OTGSC;
-
-	/* if ID change sts, it is a host wakeup event */
-	if (wakeup_req && (otgsc & OTGSC_IS_USB_ID)) {
-		printk(KERN_INFO "otg host ID wakeup\n");
-		/* if host ID wakeup, we must clear the b session change sts */
-		UOG_OTGSC = otgsc & (~OTGSC_IS_USB_ID);
-		return WAKEUP_EVENT_ID;
-	}
-	if (wakeup_req && (!(otgsc & OTGSC_STS_USB_ID))) {
-		printk(KERN_INFO "otg host Remote wakeup\n");
-		return WAKEUP_EVENT_DPDM;
+	int portsc = UOG_PORTSC1;
+	int usbsts = UOG_USBSTS;
+
+	/* if there has been a port change detect, it is for us */
+	if (wakeup_req && (usbsts & USBSTS_PCI)) {
+		/* Original ID only code said: if host ID wakeup, we must clear the b session change sts
+
+                   This is not done (seems to still work).  Original code also never cleared the ID
+                   interrupt status though (nor did the driver), so it figured everything was ID. */
+		if (portsc & PORTSC_CONNECT_STATUS_CHANGE) {
+			if (portsc & PORTSC_WKCN_EN && portsc & PORTSC_CURRENT_CONNECT_STATUS) {
+				printk(KERN_INFO "otg host connect wakeup\n");
+			}
+			if (portsc & PORTSC_WKDC_EN && !(portsc & PORTSC_CURRENT_CONNECT_STATUS)) {
+				printk(KERN_INFO "otg host disconnect wakeup\n");
+			}
+			return WAKEUP_EVENT_ID;    /* not really, but the type isn't used anyway */
+		}
+		if (portsc & PORTSC_OVER_CURRENT_CHG && portsc & PORTSC_WKOC_EN && portsc & PORTSC_OVER_CURRENT_ACT) {
+			printk(KERN_INFO "otg host over current wakeup\n");
+			return WAKEUP_EVENT_DPDM;  /* not really, but the type isn't used anyway */
+		}
+		if (portsc & PORTSC_PORT_FORCE_RESUME) {
+			printk(KERN_INFO "otg host device resume wakeup\n");
+			return WAKEUP_EVENT_DPDM;
+		}
+
+		printk(KERN_INFO "otg host unknown wakeup\n");
+		return WAKEUP_EVENT_DPDM;  /* not really, but the type isn't used anyway */
 	}
 
 	return WAKEUP_EVENT_INVALID;
diff --git a/arch/arm/plat-mxc/include/mach/arc_otg.h b/arch/arm/plat-mxc/include/mach/arc_otg.h
index 8e1c4cd0ca96..fad1def30d08 100644
--- a/arch/arm/plat-mxc/include/mach/arc_otg.h
+++ b/arch/arm/plat-mxc/include/mach/arc_otg.h
@@ -166,6 +166,9 @@ extern volatile u32 *mx3_usb_otg_addr;
 #define PORTSC_STS			(1 << 29)	/* serial xcvr select */
 #define PORTSC_PTW                      (1 << 28)       /* UTMI width */
 #define PORTSC_PHCD                     (1 << 23)       /* Low Power Suspend */
+#define PORTSC_WKOC_EN                  (1 << 22)       /* Wake on Over-current Enable */
+#define PORTSC_WKDC_EN                  (1 << 21)       /* Wake on Disconnect Enable */
+#define PORTSC_WKCN_EN                  (1 << 20)       /* Wake on Connect Enable */
 #define PORTSC_PORT_POWER		(1 << 12)	/* port power */
 #define PORTSC_LS_MASK			(3 << 10)	/* Line State mask */
 #define PORTSC_LS_SE0			(0 << 10)	/* SE0     */
```

# Compiling a kernel

The kernel has to be compiled with a 4.x gcc.  This can be obtained
from the embedded Debian repo in the jessie release (anything more
recent than jessie only have 5.x and above) or by using the Nix.
package manager.

To get it via Debian, add a source file to /etc/apt/sources.list.d

```
deb http://emdebian.org/tools/debian/ jessie main
```

and add the armhf architecture (the jessie release requires items from
armhf)  and install  the  11.7  version of  crossbuild-essential-armhf

```bash
dpkg --add-architecture armhf
apt-get install -t jessie crossbuild-essential-armhf
```

To get it via Nix, install the Nix package manager [as per the
website](https://nixos.org/nix/) and then build the gcc49 compiler in
a cross system

```bash
nix-env -iA nixpkgs.buildPackages.binutils --arg crossSystem '{ config = "armv7l-linux-gnueabihf"; }'
nix-env -iA nixpkgs.buildPackages.gcc49 --arg crossSystem '{ config = "armv7l-linux-gnueabihf"; }'
```

The compiler generates unaligned access as it expects the kernel to
fix them up.  The kernel isn't set to do fixups on itself though (see
https://community.nxp.com/thread/266293 for details), so this has to be
turned off by adding an option in Makefile to KBUILD_CFLAGS

```bash
KBUILD_CFLAGS += -mno-unaligned-access
```

From more recent versions of Perl is is nessesary to make the
suggested modification to the kernel/timeconst.pl file as technically
defined(@...) doesn't check to see if the list is empty (see Perl
warnings documentation for details).

The kernel can then be built with the following command for Debian

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- uImage
```

or for Nix

```bash
hardeningDisable=all make ARCH=arm CROSS_COMPILE=armv7l-linux-gnueabihf- uImage
```

# Uploading a kernel

The Kobo u-boot bootm command is hacked to run load_ntxkernel if it
hasn't already been run before and overwrites the kernel we
downloaded.  Run load_ntxkernel before the download to avoid this.

```
load_ntxkernel
loady
~+sx -kb uImage
setenv bootargs console=ttymxc0,115200 rootwait rw no_console_suspend lpj=3997696
imi
bootm
```

```bash
umount /mnt/onboard
modprobe g_file_storage file=/dev/mmcblk0p3
```

The hacked bootm command also adds several additional parameters if
they are not already set (this can be seen in the kernel log).  This
includes appending mx50_1GHz to the end which screws up using
init=/bin/sh in an emergency.  This can be worked around with

```
init=/bin/sh -c sh
```

# Serial breakout cable

| load                 | max draw               |
|----------------------|------------------------|
| usb to serial        |  19mA ( 15mA by meter) |
| usb to serial + kobo | 540mA (532mA by meter) |

| color | use |
|-------|-----|
| grey  | +5V |
| green |  0V |
| red   |     |
| org   |     |
