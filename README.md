
Build Fedora Gnome Desktop on RISC-V!!
=======================================================
The intent of this document is to share the hardware setup and source code
build instructions to bring up Fedora 29 GNOME desktop on HighFive Unleased
board. It is assumed that you know how to setup RISC-V development environment.
If not please follow the below instructions using [freedom-u-sdk](https://github.com/sifive/freedom-u-sdk.git).

```
git clone https://github.com/sifive/freedom-u-sdk.git
cd freedom-u-sdk
git submodule update --init --recursive
make
make qemu (for qemu boot)
```
Let's get back to the problem at hand i.e. Setup Fedora 29 GNOME desktop on RISC-V!!
Here are the hardware used for our setup.

Required Hardware:
----------------------------------------------------------------------------------

* HighFive Unleashed Board (This is the only Linux capable [RISC-V board](https://www.sifive.com/products/hifive-unleashed/))
* Microsemi HighFive Unleashed Expansion board([available here](https://www.crowdsupply.com/microsemi/hifive-unleashed-expansion-board))
	* Again only expansion board available with multiple I/O Support
	* However, only PCIe, SATA, M.2 SSD connectors are enabled right now.
* Radeon HD 6450 GPU card
	* Any Caicos-based card should be OK. But the kernel config instructs specific firmware to be used. It is recommended to use the above specific GPU as it is verified. In case you want to use any other GPU, load the appropriate firmware accordingly.
	- The gpu uses x16 PCI Express card connector.
* PCIe to USB card (I have used [this](http://a.co/d/du7drEo))
	- x1 PCI Express card connector can be used to provide USB ports for
	  mouse/keyboard.
* SATA Drive(HDD/SSD) or NVMe SSD. This is where the Feodra image will be copied.
	It is not recommended to use  image from micro SD card.


Source code:
----------------------------------------------------------------------------------
The current upstream kernel (v4.19-rc2) boots in QEMU for RISC-V. However, it is still
missing some of the HighFive Unleashed specific drivers. That's why the Linux kernel
source tree also has to be hosted out of mainline tree.

**riscv-linux-conf:**

It contains the required linux config files for this project. This needs to be copied  to freedom-u-sdk conf directory.

N.B. The Linux config file will only work for HighFive Unleashed board with a GPU card. You can't use this in a QEMU setup. Mainline kernel 4.19-rc2 should boot in QEMU without
any issues.

**riscv-linux:**

It is based on 4.19-rc2 and contains all required drivers and couple of kernel hacks.
Here is the summary of out-of-tree commits:

	-------------------------------------------------
	Kernel hacks
	-------------------------------------------------
	e5b7972a RISCV: Fix end PFN for low memory
	aa230e7d RISC-V: Networking fix Hack
	-------------------------------------------------
	Microsemi expansion board specific driver
	-------------------------------------------------
	1fd6c8fb pcie-microsemi: added support for the Vera-board root complex
	-------------------------------------------------
	Unleashed Specific drivers
	-------------------------------------------------
	e6655fae pwm-sifive: add a driver for SiFive SoC PWM
	28447771 gpio-sifive: support GPIO on SiFive SoCs
	ca0f9319 u54-prci: driver for core U54 clocks
	b7bd3468 u54-prci: driver for core U54 clocks
	c61e4295 gemgxl-mgmt: implement clock switch for GEM tx_clk
	58f76c1c serial/sifive: initial driver from Paul Walmsley
	dc4e19c2 spi-nor: add support for is25wp{32,64,128,256}
	f169ce24 spi-sifive: support SiFive SPI controller in Quad-Mode

**riscv-pk:**

This repository builds the BBL (Berkely Boot Loader) for RISC-V project.
To use Microsemi expansion board, the DT in HighFive Unleashed board has
to be updated unless you have the latest firmware with updated DT.
In that case, you don't need this change.

	9d561e92 Add microsemi pcie entry in bbl.
	5a45a6ea Add libfdt support.

Build
----------------------------------------------------------------------------------

The build instructions are based on freedom-u-sdk setup. If you are using your own tool chain build, you can just compile the riscv-pk and riscv-linux separately and use the `bbl.bin` image. All the commands are run from freedom-u-sdk root directory. The insturctions have only been tested on Ubuntu 16.04.

* Checkout your riscv-pk repo from this remote source.
```
cd riscv-pk
git remote add -f riscv_pk_fedora git@github.com:atishp04/riscv-pk.git
git fetch riscv_pk_fedora
git checkout microsemi_pcie_support
```
* Recompile bbl.
```
autoconf && autoheader
make bbl
```
Now the new BBL will add microsemi specific PCIe entry to the device tree.
   You can always verify the device tree entry by adding '--enable-print-device-tree'
   option to the root Makefile in freedom-u-sdk.
* Checkout riscv-linux repo from this remote source.
```
git remote add -f riscv_linux_fedora git@github.com:atishp04/riscv-linux.git
git fetch riscv_linux_fedora
git checkout 4.19_fedora_success_sep11
```
* Copy the linux config from this repo to your freedom-u-sdk directory.
   The current config mounts the Fedora image to the first partition of the disk.
   The present config file has two options
	- SATA SSD (root=/dev/sda1)
	- NVME SSD (root=/dev/nvme0n1p1)
   Enable/disable correct CONFIG_CMDLINE config based on your setup.
```
cd ~
git clone git@github.com:atishp04/riscv-linux-conf.git
cp riscv-linux-conf/config_fedora_success_4.19_demo_sep11 <freedom-u-sdk path>/conf/
```

* Update freedom-u-sdk Makefile to use the just copied config. Here are the required modification

```
cd <freedom-u-sdk path>
#Comment the below line and add the next one to the Makefile
#linux_defconfig := $(confdir)/linux_defconfig
linux_defconfig := $(confdir)/config_fedora_success_4.19_demo_sep11
```
* Recompile the kernel (from freedom-u-sdk repo).

```
make
```

* copy the bbl file to the first partition of your microSD card (i.e. /dev/disk2s1 in this case
	).

```
scp <build machine>:<freedom-u-sdk path>/work/bbl.bin /tmp
sudo dd if=/tmp/bbl.bin of=/dev/disk2s1 bs=1024
```

* Download the latest Fedora Gnome Desktop stage4 image from [this link](http://fedora-riscv.tranquillity.se/koji/tasks?state=closed&view=flat&method=createAppliance&order=-id).

```
wget http://fedora-riscv.tranquillity.se/kojifiles/work/tasks/4502/104502/Fedora-GNOME-Rawhide-20180906.n.0-sda.raw.xz
unxz Fedora-GNOME-Rawhide-20180906.n.0-sda.raw.xz
```
The disk image (above) is partitioned, but usually we need an unpartitioned ("naked") filesystem.

```
guestfish -a Fedora-GNOME-Rawhide-20180906.n.0-sda.raw run : download /dev/sda1 Fedora-GNOME-Rawhide-20180906.n.0-sda1.raw
```

This creates a naked ext4 filesystem called `*-sda1.raw`. The naked ext4 filesystem can be copied to first partition of your disk.

```
sudo dd if=Fedora-GNOME-Rawhide-20180906.n.0-sda1.raw of=/dev/sda1 bs=4M
```

(The drive should be connected to your Desktop/Laptop. If it is recognized as something other than /dev/sda1, please change accordingly.)

* Now connect the hard drive in appropriate slot on Expansion board.
* Connect the power to the Expansion board only. It powers the unleashed board as well.
* Turn on the Unleashed switch.
* Turn on the Expansion board switch.
* Connect to the serial terminal. If a display is connected, you should Fedora prompt in a while. Once you are done with the initial Fedora setup process in screen, you may have to reboot the system.  

Welcome to RISC-V Fedora Desktop!!

![Fedora Setup screenshot](images/fedora.JPG?raw=true "Title")


**Acknowledgement:**

[1] Fedora image instructions are available from [this page](https://fedoraproject.org/wiki/Architectures/RISC-V/Installing
).

(Responsible for Fedora builds and numerous help over irc)

David Abdurachmanov

DJ Delorie

Richard M Jones

**Reference:**

[1] https://github.com/sifive/freedom-u-sdk.git

[2] https://www.sifive.com/products/hifive-unleashed/

[3] https://www.crowdsupply.com/microsemi/hifive-unleashed-expansion-board

[4] http://a.co/d/du7drEo

For any questions: atish.patra@wdc.com
