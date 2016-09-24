# Extracting bits from flash

Kernel from start of flash

dd if=/dev/mmcblk0 bs=1024 skip=1024

Is a u-boot legacy uImage.  Drop first 64 bytes to get a standard zImage.

U-boot environment block from start of flash

dd if=/dev/mmcblk0 of=config bs=1024 skip=768 count=128

It can be modified with hexedit and then written back with

dd if=config of=/dev/mmcblk0 bs=1024 seek=768 count=128

# Connecting

http://www.cyberciti.biz/hardware/5-linux-unix-commands-for-connecting-to-the-serial-console/

stty -F /dev/ttyUSB0 115200 raw pass8
cu -l /dev/ttyUSB0

# Disabling the watchdog

From the Linux kernel code (pmic_core_i2c.c) it seems you have to send
command 22 with argument 0xff to the msp430 over the i2c bus (from
pmic_core_i2c.c).  Probing the bus gives a list of devices

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

The system currently hangs after "mma7660 sensor triggered ..." the
next message from the original kernel is "usb unpluggged" so likely
there are issues with the modified USB stack.

- works with a recompilation of the existing config
- add support for DP host port on freescale controller
* internal UTMI transceiver hangs
* philips ISP1301 transceiver hangs
* freescale MC13783 transceiver hangs
- just "support for fresscale on-chip EHCI USB controller" doesn't compile
- could it just be low power?? -- doesn't seem to be (plugging in doesn't help)
- remove root hub transaction translate (only tried UTMI transceiver)

drivers/usb/host/Kconfig
CONFIG_USB_EHCI_ARC_OTG - enable support for USB OTG port in HS/FS mode
CONFIG_USB_EHCI_FSL_UTMI - enable support for the on-chip High Speed UTMI transceiver
./arch/arm/mach-mx5/usb_dr.c

stuff from arch/arm/mach-mx5 (more stuff in arch/arm/plat-mxc too)

CONFIG_MMU=y
CONFIG_ARCH_MXC=y

- system.o iomux.o cpu.o mm.o devices.o serial.o dma.o lpmodes.o pm.o sdram_autogating.o bus_freq.o usb_dr.o usb_h1.o usb_h2.o dummy_gpio.o  early_setup.o

CONFIG_ARCH_MX5=y
CONFIG_ARCH_MX51=y - clock.o suspend.o
CONFIG_ARCH_MX53=y - clock.o suspend.o mx53_wp.o pm_da9053.o
CONFIG_ARCH_MX50=y - clock_mx50.o dmaengine.o dma-apbh.o mx50_suspend.o mx50_freq.o mx50_ddr_freq.o mx50_wfi.o mx50_ntx_io.o
CONFIG_MACH_MX51_BABBAGE=y - mx51_babbage.o mx51_babbage_pmic_mc13892.o
CONFIG_MACH_MX53_EVK=y - mx53_evk.o mx53_evk_pmic_mc13892.o
CONFIG_MACH_MX53_ARD=y - mx53_ard.o mx53_ard_pmic_ltc3589.o
CONFIG_MACH_MX50_ARM2=y - mx50_arm2.o mx50_arm2_pmic_mc13892.o
CONFIG_MACH_MX50_RDP=y - mx50_rdp.o mx50_rdp_pmic_mc13892.o ntx_hwconfig.o


system.c
iomux.c
cpu.c
mm.c
devices.c
serial.c
dma.c
lpmodes.c
pm.c
sdram_autogating.c
bus_freq.c
usb_dr.c
usb_h1.c
usb_h2.c
dummy_gpio.c
early_setup.c
clock.c
suspend.c
clock.c
suspend.c
mx53_wp.c
pm_da9053.c
clock_mx50.c
dmaengine.c
dma-apbh.c
mx50_suspend.c
mx50_freq.c
mx50_ddr_freq.c
mx50_wfi.c
mx50_ntx_io.c
mx51_babbage.c
mx51_babbage_pmic_mc13892.c
mx53_evk.c
mx53_evk_pmic_mc13892.c
mx53_ard.c
mx53_ard_pmic_ltc3589.c
mx50_arm2.c
mx50_arm2_pmic_mc13892.c
mx50_rdp.c
mx50_rdp_pmic_mc13892.c
ntx_hwconfig.c


VERSION = 2
PATCHLEVEL = 6
SUBLEVEL = 35
EXTRAVERSION = .3
NAME = Sheep on Meth
