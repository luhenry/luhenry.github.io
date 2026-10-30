---
title: "Installing Ubuntu 26.10 on the SpacemiT K3 with PXE and labgrid"
layout: post
---

I wanted a repeatable way to install Canonical's Ubuntu 26.10 Server image on a SpacemiT K3 Pico-ITX without dedicating a keyboard, display, or USB stick to it. My setup uses a Raspberry Pi as a small lab host. The Pi controls power, exports the K3's serial console, serves DHCP and TFTP, and provides the network path used by the installer.

The result is a useful pattern for headless board bring-up: update the firmware over USB, boot the installer over PXE, drive it through UART, and install onto NVMe.

{% include image.md file="install-ubuntu-26.10-on-spacemit-k3/IMG_0908.jpeg" description="My desk with a Spacemit K3, a Raspberry Pi 3b+, and all the cables to connect Serial, Network, and Power" %}

## Hardware and topology

My lab host is a Raspberry Pi running Ubuntu. Two K3 boards are attached through CH340 USB-to-TTL adapters. An Energenie EG-PMS2 USB power strip controls their power. Both boards currently share one outlet, which is not ideal, but labgrid still gives each UART an independent target.

The Pi's USB Ethernet adapter forms an isolated board network:

```text
ludovic-pi:    192.168.254.1/24
ludovic-k3-0:  192.168.254.10
ludovic-k3-1:  192.168.254.11
```

The Pi forwards traffic to its upstream `eth0` interface and applies NAT. This matters because PXE only delivers the first-stage files over TFTP. The Ubuntu initrd later downloads the full installer ISO from Canonical over HTTPS.

## Making UART control reliable with labgrid

I installed labgrid in a virtual environment:

```sh
cd ~/git/k3-host
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install labgrid
```

A trap with identical USB serial adapters is relying on `/dev/ttyUSB0` and `/dev/ttyUSB1`. Their numbering can change after reconnecting a hub. I matched each adapter by its physical udev path instead:

```yaml
k3-pico-itx-0:
  k3-pico-itx-serial-port-1:
    cls: USBSerialPort
    match:
      '@ID_PATH': 'platform-3f980000.usb-usb-0:1.1.3.2.3:1.0'

k3-pico-itx-1:
  k3-pico-itx-serial-port-1:
    cls: USBSerialPort
    match:
      '@ID_PATH': 'platform-3f980000.usb-usb-0:1.1.3.2.1:1.0'
```

The exporter also declares the shared `SiSPMPowerPort` and TFTP provider. Labgrid opens each local `USBSerialPort` through ser2net and exposes it as an RFC2217 `NetworkSerialPort`. That is why `labgrid-client` reports a network serial device even when both client and exporter run on the Pi.

I created one coordinator place per board:

```sh
labgrid-client -p k3-pico-itx-0 create
labgrid-client -p k3-pico-itx-0 add-match 'pi-0-exporter/k3-pico-itx-0/*'
labgrid-client -p k3-pico-itx-1 create
labgrid-client -p k3-pico-itx-1 add-match 'pi-0-exporter/k3-pico-itx-1/*'
```

The console and power controls then become straightforward:

```sh
labgrid-client -p k3-pico-itx-1 acquire
labgrid-client -p k3-pico-itx-1 console -l
```

In another terminal:

```sh
labgrid-client -p k3-pico-itx-1 power cycle -t 12
```

The 12-second delay lets the board discharge fully. Since my outlet is shared, this command currently reboots both K3s.

One debugging lesson was particularly useful. `ludovic-k3-1`'s console repeatedly went silent even though the USB device, udev path, and ser2net process were correct. `lsof /dev/ttyUSB0` revealed an old system service running `cat /dev/ttyUSB0`. Serial readers compete for bytes, so the logger consumed output before labgrid saw it. Removing that service fixed the recurring failure.

```
(.venv) ubuntu@ludovic-pi:~/git/k3-host$ labgrid-client -p k3-pico-itx-1 console -l
connecting to NetworkSerialPort(target=Target(name='k3-pico-itx-1', env=None), name='k3-pico-itx-serial-port-1', state=<BindingState.bound: 1>, avail=True, host='ludovic-pi', port=44875, speed=115200, protocol='rfc2217') calling telnet ludovic-pi 44875
Trying 127.0.0.1...
Connected to ludovic-pi.
Escape character is '^]'.
sys: 0x10000200
try sd...
bm:3
ERROR:   CMD8
ERROR:   sd f! l:90
bm:4
nor m:0xef d:0x6017
j...

U-Boot SPL 2022.10 (Sep 17 2026 - 14:17:15 +0000)
DDR Part Number: MT62F2G32D4DS, Size: 8192MB, Data Rate: 6400MT/s
DDR quick boot consume 9ms
```

## Updating the firmware

Canonical Ubuntu requires the newer EDK2 firmware. To enter the K3 BootROM download mode, power the board off, hold the FDL flashing button, apply power, release the button, then connect the board's flashing USB-C port to the host.

The device appears as `361c:1001`:

```text
????????????    DFU download
```

Initially, fastboot only worked through `sudo`. I added a udev rule at `/etc/udev/rules.d/70-spacemit-k3.rules`:

```udev
SUBSYSTEM=="usb", ATTR{idVendor}=="361c", ATTR{idProduct}=="1001", MODE="0660", GROUP="plugdev"
```

After reloading udev rules, I could verify BootROM mode as an ordinary user:

```sh
fastboot devices
fastboot getvar version-brom
```

```text
version-brom: 1.0
```

I then used Canonical's experimental firmware script:

```sh
curl https://raw.githubusercontent.com/canonical/spacemit-k3-flash-firmware/refs/heads/main/flash-k3-firmware.sh | bash
```

The flashing gadget has no useful unique serial number. With multiple boards attached, physically trace the cable or disconnect the other board before writing firmware.

```
(.venv) ubuntu@ludovic-pi:~/git/k3-host$ curl https://raw.githubusercontent.com/canonical/spacemit-k3-flash-firmware/refs/heads/main/flash-k3-firmware.sh | bash
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total   Spent   Left   Speed
100  11697 100  11697   0      0  31172      0                              0
==> K3 firmware flasher
==>   PPA:     ppa:ubuntu-risc-v-team/k3
==>   suite:   resolute
==>   workdir: /tmp/k3-firmware-8srudy

==> pulling opensbi-spacemit from ppa:ubuntu-risc-v-team/k3 (resolute)...
Found opensbi-spacemit 1.0.5+fix1-5 in resolute
Downloading opensbi-spacemit_1.0.5+fix1-5.dsc from launchpadlibrarian.net (0.001 MiB)
[=====================================================>]100%

NOTICE: 'opensbi-spacemit' packaging is maintained in the 'Git' version control system at:
 https://salsa.debian.org/opensbi-team/opensbi.git

Please use:
 $ git clone https://salsa.debian.org/opensbi-team/opensbi.git
to retrieve the latest (possibly unreleased) updates to the package.

Downloading opensbi-spacemit_1.0.5+fix1-5_all.deb from launchpadlibrarian.net (0.124 MiB)
[=====================================================>]100%
==> pulling u-boot-spacemit from ppa:ubuntu-risc-v-team/k3 (resolute)...
Found u-boot-spacemit 1.0.5-5 in resolute
Downloading u-boot-spacemit_1.0.5-5.dsc from launchpadlibrarian.net (0.001 MiB)
[=====================================================>]100%

NOTICE: 'u-boot-spacemit' packaging is maintained in the 'Git' version control system at:
 https://git.spacemit.com/buildroot-k3/uboot.git

Please use:
 $ git clone https://git.spacemit.com/buildroot-k3/uboot.git
to retrieve the latest (possibly unreleased) updates to the package.

Downloading u-boot-spacemit_1.0.5-5_riscv64.deb from launchpadlibrarian.net (0.519 MiB)
[=====================================================>]100%
Downloading u-boot-spl-spacemit_1.0.5-5_riscv64.deb from launchpadlibrarian.net (0.230 MiB)
[=====================================================>]100%
==> pulling edk2-spacemit from ppa:ubuntu-risc-v-team/k3 (resolute)...
Found edk2-spacemit 0.0.9-5 in resolute
Downloading edk2-spacemit_0.0.9-5.dsc from launchpadlibrarian.net (0.001 MiB)
[=====================================================>]100%
Downloading edk2-spacemit_0.0.9-5_riscv64.deb from launchpadlibrarian.net (1.662 MiB)
[=====================================================>]100%
==> pulling esos-spacemit from ppa:ubuntu-risc-v-team/k3 (resolute)...
Found esos-spacemit 1.0.5-1 in resolute
Downloading esos-spacemit_1.0.5-1.dsc from launchpadlibrarian.net (0.001 MiB)
[=====================================================>]100%
Downloading esos-spacemit_1.0.5-1_riscv64.deb from launchpadlibrarian.net (0.541 MiB)
[=====================================================>]100%
==> pulling spacemit-ec-firmware from ppa:ubuntu-risc-v-team/k3 (resolute)...
Found spacemit-ec-firmware 1:0.0.31 in resolute
Downloading spacemit-ec-firmware_0.0.31.dsc from launchpadlibrarian.net (0.001 MiB)
[=====================================================>]100%
Downloading spacemit-ec-firmware_0.0.31_riscv64.deb from launchpadlibrarian.net (0.591 MiB)
[=====================================================>]100%

==> cloning SpacemiT K3-Ubuntu-Images...
Cloning into '/tmp/k3-firmware-8srudy/spacemit'...
remote: Enumerating objects: 26, done.
remote: Counting objects: 100% (26/26), done.
remote: Compressing objects: 100% (24/24), done.
remote: Total 26 (delta 1), reused 14 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (26/26), 26.46 KiB | 1.56 MiB/s, done.
Resolving deltas: 100% (1/1), done.
==>   edk2.itb  <-  edk2-spacemit
==>   bootinfo_spinor.bin  <-  u-boot-spacemit
==>   env.bin  <-  u-boot-spacemit
==>   ec.bin  <-  spacemit-ec-firmware
==>   fw_dynamic.itb  <-  opensbi-spacemit
==>   FSBL.bin  <-  u-boot-spacemit
==>   u-boot.itb  <-  u-boot-spacemit
==>   esos.itb  <-  esos-spacemit
==> payloads in /tmp/k3-firmware-8srudy/spacemit/temp:
  FSBL.bin
  bootinfo_spinor.bin
  ec.bin
  edk2.itb
  env.bin
  esos.itb
  fw_dynamic.itb
  u-boot.itb
==> About to flash firmware to the board via USB fastboot.
==> Ensure the board is in FDL flash mode (hold FDL button while powering on).
!! Once flashing starts, do **not** interrupt it, or you could brick your board!

==> flashing firmware via SpacemiT image_flash.py...
  $ fastboot getvar version-brom
version-brom: 1.0
Finished. Total time: 0.007s
  $ fastboot stage temp/FSBL.bin
Sending 'temp/FSBL.bin' (462 KB)                   OKAY [  0.055s]
Finished. Total time: 0.062s
  $ fastboot continue
Resuming boot                                      OKAY [  0.006s]
Finished. Total time: 0.006s
  $ fastboot getvar version
version: 0.4
Finished. Total time: 0.000s
  $ fastboot oem speed:super-speed
OKAY [  0.000s]
Finished. Total time: 0.000s
  $ fastboot stage temp/u-boot.itb
Sending 'temp/u-boot.itb' (2356 KB)                OKAY [  0.093s]
Finished. Total time: 0.094s
  $ fastboot continue
Resuming boot                                      OKAY [  0.000s]
Finished. Total time: 0.001s
  $ fastboot stage temp/ec.bin
Sending 'temp/ec.bin' (237 KB)                     OKAY [  0.015s]
Finished. Total time: 0.015s
  $ fastboot oem ec:flash
OKAY [  4.352s]
Finished. Total time: 4.352s
  $ fastboot getvar mtd-size
mtd-size: 8M
Finished. Total time: 0.000s
  $ fastboot getvar blk-size
blk-size: universal
Finished. Total time: 0.332s
[flash] querying mtd-size from device...
[flash] mtd-size = '8M'
[flash] using partition file: partition_4M.json  (format: mtd)
  $ fastboot flash mtd partition_4M.json
Warning: skip copying mtd image avb footer (mtd partition size: 0, mtd image size: 965).
Sending 'mtd' (0 KB)                               OKAY [  0.009s]
Writing 'mtd'                                      OKAY [  0.338s]
Finished. Total time: 0.367s
  $ fastboot flash bootinfo temp/bootinfo_spinor.bin
Sending 'bootinfo' (0 KB)                          OKAY [  0.007s]
Writing 'bootinfo'                                 OKAY [  0.078s]
Finished. Total time: 0.094s
  $ fastboot flash fsbl temp/FSBL.bin
Sending 'fsbl' (462 KB)                            OKAY [  0.023s]
Writing 'fsbl'                                     OKAY [  4.385s]
Finished. Total time: 4.418s
  $ fastboot flash env temp/env.bin
Sending 'env' (16 KB)                              OKAY [  0.009s]
Writing 'env'                                      OKAY [  0.366s]
Finished. Total time: 0.382s
  $ fastboot flash esos temp/esos.itb
Sending 'esos' (437 KB)                            OKAY [  0.022s]
Writing 'esos'                                     OKAY [  4.141s]
Finished. Total time: 4.188s
  $ fastboot flash opensbi temp/fw_dynamic.itb
Sending 'opensbi' (264 KB)                         OKAY [  0.015s]
Writing 'opensbi'                                  OKAY [  2.550s]
Finished. Total time: 2.591s
  $ fastboot flash uboot temp/edk2.itb
Sending 'uboot' (3140 KB)                          OKAY [  0.130s]
Writing 'uboot'                                    OKAY [ 28.257s]
Finished. Total time: 28.424s
[flash] querying blk-size from device...
[flash] blk-size = 'universal'
[flash] using partition file: partition_universal.json  (format: gpt)
  $ fastboot flash gpt partition_universal.json
Warning: skip copying gpt image avb footer (gpt partition size: 0, gpt image size: 1643).
Sending 'gpt' (1 KB)                               OKAY [  0.010s]
Writing 'gpt'                                      OKAY [  0.078s]
Finished. Total time: 0.124s
  $ fastboot flash env temp/env.bin
Sending 'env' (16 KB)                              OKAY [  0.009s]
Writing 'env'                                      OKAY [  0.185s]
Finished. Total time: 0.215s
  $ fastboot flash bootinfo temp/bootinfo_spinor.bin
Sending 'bootinfo' (0 KB)                          OKAY [  0.008s]
Writing 'bootinfo'                                 OKAY [  0.006s]
Finished. Total time: 0.038s
  $ fastboot flash fsbl temp/FSBL.bin
Sending 'fsbl' (462 KB)                            OKAY [  0.023s]
Writing 'fsbl'                                     OKAY [  0.006s]
Finished. Total time: 0.054s
  $ fastboot flash esos temp/esos.itb
Sending 'esos' (437 KB)                            OKAY [  0.022s]
Writing 'esos'                                     OKAY [  0.006s]
Finished. Total time: 0.053s
  $ fastboot flash opensbi temp/fw_dynamic.itb
Sending 'opensbi' (264 KB)                         OKAY [  0.016s]
Writing 'opensbi'                                  OKAY [  0.006s]
Finished. Total time: 0.046s
  $ fastboot flash uboot temp/edk2.itb
Sending 'uboot' (3140 KB)                          OKAY [  0.129s]
Writing 'uboot'                                    OKAY [  0.006s]
Finished. Total time: 0.170s
  [esp] skipped (not in --only list)
  [cidata] skipped (not in --only list)
  [writable] skipped (not in --only list)
[flash] log saved: temp/flash.txt

[flash] done
!! verify: /home/ubuntu/git/k3-host/fastboot-dump.py not found — skipping readback check

==============================================================================
 Firmware flashed. Please reboot the board.

 Next: install Ubuntu on the board's storage (NVMe/UFS), using an installer
 image on an USB thumb drive. Press F2 when EDK2 loads to enter EDK2 menu and
 select boot device.

 Once Ubuntu is up, make sure to maintain all firmware up to date:
   sudo add-apt-repository ppa:ubuntu-risc-v-team/k3
   sudo apt update && sudo apt install spacemit-firmware
==============================================================================
```

And from the serial of the board.

```
(.venv) ubuntu@ludovic-pi:~/git/k3-host$ labgrid-client -p k3-pico-itx-0 console
connecting to NetworkSerialPort(target=Target(name='k3-pico-itx-0', env=None), name='k3-pico-itx-serial-port-1', state=<BindingState.bound: 1>, avail=True, host='ludovic-pi', port=58989, speed=115200, protocol='rfc2217') calling telnet ludovic-pi 58989
Trying 127.0.0.1...
Connected to ludovic-pi.
Escape character is '^]'.
sys: 0x10001200
bm:2
usb_init : enter 3296,3072
usb_core_init : enter
DWC3_GRXTHRCFG:0x4400000
ROM: usb download handler
 rst
done H
setup= 0x1000680 0x400000,
 rst
done H
setup= 0x350500 0x0,
setup= 0x1000680 0x120000,
setup= 0x2000680 0x90000,
setup= 0x2000680 0x200000,
setup= 0x3000680 0xff0000,
setup= 0x3020680 0xff0409,
setup= 0x3010680 0xff0409,
setup= 0x10900 0x0,
usb_rx_bytes : start len[4096]
setup= 0x3020680 0xff0409,
setup= 0x3040680 0xff0409,
fastboot_handle_command: version-brom
usb_tx_bytes : start len[7]
usb_rx_bytes : start len[4096]
fastboot_handle_command: max-download-size
usb_tx_bytes : start len[14]
usb_rx_bytes : start len[4096]
fastboot_handle_command: 00073820
Starting download of 473120 bytes
usb_tx_bytes : start len[12]
usb_rx_bytes : start len[473120]
usb_tx_bytes : start len[4]
usb_rx_bytes : start len[4096]
fastboot_handle_command: continue
usb_tx_bytes : start len[4]
j...

U-Boot SPL 2022.10 (Sep 17 2026 - 14:17:15 +0000)
DDR Part Number: MT62F2G32D4DS, Size: 8192MB, Data Rate: 6400MT/s
DDR training consume 5616ms
.....[ 161.827]

U-Boot 2022.10 (Sep 17 2026 - 14:17:15 +0000)

[ 161.829] CPU:   rv64imafdcvh
[ 161.832] Model: spacemit k3 pico itx board
[ 161.836] DRAM:  16 GiB
[ 161.843] espi_post_bind: espi@cac8c000, seq=0
[ 161.844] espi_child_post_bind: binding child device ec@0
[ 161.853] reset driver probe finish
[ 161.947] Slave general capabilities: 0x0308000f
[ 161.948] spacemit_espi espi@cac8c000: SpacemiT eSPI controller initialized successfully
[ 161.961] RT7451: Retimer already initialized, skipping
[ 161.966] Core:  628 devices, 33 uclasses, devicetree: board
[ 161.970] WDT:   Started watchdog@d4014000 with servicing (60s timeout)
[ 161.975] MMC:
[ 161.977] Loading Environment from mtdENV... k1x_qspi spi@d420c000: qspi iobase:0x0x00000000d420c000, ahb_addr:0x0x00000000b8000000, max_hz:26000000Hz
[ 161.990] k1x_qspi spi@d420c000: rx buf size:128, tx buf size:256, ahb buf size=512
[ 161.998] k1x_qspi spi@d420c000: AHB read enabled
[ 162.003] k1x_qspi spi@d420c000: AHB buf size: 512
[ 162.007] k1x_qspi spi@d420c000: Speed Change: 26000000 Hz -> 26500000 Hz
[ 162.014] SF: Detected w25q64dw with page size 256 Bytes, erase size 4 KiB, total 8 MiB
OK
[ 162.026] Cannot support showing bootlogo in this boot mode!
[ 162.029] Unsupported boot mode for splash screen
[ 162.033] initialize_console_log_buffer
[ 162.037] Have allocated memory for console log buffer
[ 162.042] In:    serial
[ 162.044] Out:   serial
[ 162.046] Err:   serial
[ 162.053] Found device 'dp1@cac88000', disp_uc_priv=00000003fbe9f490
[ 162.209] dp cannot get HPD signal
[ 162.210] spacemit_display_init: device 'dpu@c0340000' display won't probe (ret=-1)
[ 162.369] dp cannot get HPD signal
[ 162.370] display devices not found or not probed yet: -1
[ 162.435] Found 1 valid MAC addresses.
[ 162.435] TLV item: product_name = k3-pico-itx
[ 162.440] TLV item: part# = MPK3
[ 162.443] TLV item: serial# = HW3MPK3161280066
[ 162.447] TLV item: ddr_partnumber = MT62F2G32D4DS
[ 162.453] fail to get alias node efuse_power
[ 162.657] save DDR training info
[ 163.022] write 38912byte to spinor @offset 65536
[ 163.025] spacemit reboot: read PMIC reg 0xab value 0xf0
[ 163.030] Can't delete "stdout"
[ 163.032] No USB maximum speed specified. Using super speed
[ 163.039] k3-usb3-phy phy@0cad30000: k3 usb3 phy enter...
[ 163.046] mv_usb_phy phy@cad20000: k1x-ci-usb-phy-probe: Enter...
[ 163.050] k3-usb3-phy phy@0cad30000: PUPHY Rx Reg Configured
[ 163.055] k3-usb3-phy phy@0cad30000: PHY version: 0x302 init as USB3 mode
[ 163.061] k3-usb3-phy phy@0cad30000: PUPHY Rx Reg Configured
[ 163.067] k3-usb3-phy phy@0cad30000: PHY version: 0x302 init as USB3 mode
[ 163.074] dwc3-generic-peripheral dwc3@cad00000: this is a DesignWare USB3 DRD Core
[ 163.283] fusb301 tcpc@25: Device ID = 0x12
[ 163.297] fusb301 tcpc@25: fusb301 set pwr_mode: 1.5A
[ 163.301] fusb301 tcpc@25: fusb301 set toggle time: Toggle_35ms
[ 163.307] fusb301 tcpc@25: fusb301 set mode: Drp_Acc
[ 163.312] fusb301 tcpc@25: fusb301 set state: Error_Recovery
[ 163.318] fusb301 tcpc@25: mode[0x20], host_cur[0x02], dttime[0x00]
[ 163.322] fusb301 tcpc@25: failed to read status
[ 163.326] fusb301 tcpc@25: failed to reset device, ret: 8
[ 163.331] 'port' subnode not found
[ 163.334] fusb301 tcpc@25: failed to get typec switch
[ 163.339] fusb301 tcpc@25: TCPM: set voltage limit = 0 mV
[ 163.344] fusb301 tcpc@25: TCPM: set current limit = 0 mA
[ 163.351] fusb301 tcpc@25: get vbus: On
[ 163.355] fusb301 tcpc@25: fusb301 set pwr_mode: none
[ 163.360] fusb301 tcpc@25: fusb301 set mode: Drp_Acc
[ 163.365] fusb301 tcpc@25: get orientation: 0
[ 163.367] fusb301 tcpc@25: get cc1 = open, cc2 = open
[ 163.372] fusb301 tcpc@25: TCPM: init finished
[ 163.378] fusb301 tcpc@25: vbus=On
[ 163.380] fusb301 tcpc@25: get vbus: On
[ 163.447] fusb301 tcpc@25: IRQ: ATTACH detected
[ 163.452] fusb301 tcpc@25: get orientation: 2
[ 163.453] fusb301 tcpc@25: get cc1 = open, cc2 = rp-def
[ 163.661] fusb301 tcpc@25: fusb301 set pwr_mode: none
[ 163.665] fusb301 tcpc@25: fusb301 set mode: Drp_Acc
[ 169.268] fusb301 tcpc@25: TCPM: exit in state TOGGLING
[ 169.830] dwc3-generic-peripheral dwc3@cad00000: Connection Speed: high-speed
[ 169.983] handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x0 value 0x100 length 0x40
[ 169.998] dwc3-generic-peripheral dwc3@cad00000: Connection Speed: high-speed
[ 170.151] handle setup SET_ADDRESS, 0x0, 0x5 index 0x0 value 0x37 length 0x0
[ 170.166] handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x0 value 0x100 length 0x12
[ 170.171] handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x0 value 0x200 length 0x9
[ 170.178] handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x0 value 0x200 length 0x20
[ 170.186] handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x0 value 0x300 length 0xff
[ 170.193] handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x409 value 0x302 length 0xff
[ 170.201] handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x409 value 0x301 length 0xff
[ 170.208] handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x409 value 0x303 length 0xff
[ 170.217] handle setup SET_CONFIGURATION, 0x0, 0x9 index 0x0 value 0x1 length 0x0
[ 170.223] handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x409 value 0x302 length 0xff
[ 170.231] handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x409 value 0x304 length 0xff
[ 171.845] Starting download of 242688 bytes
[ 171.856] .
[ 171.856] downloading/uploading of 242688 bytes finished
[ 171.883] Fastboot oem ec:flash
[ 171.883] Fastboot EC image: addr=0000000140000000 size=0x3b400
[ 171.889] Calculating incoming EC RW firmware MD5...
[ 171.894] Calculating current EC RW firmware MD5...
[ 176.228] Current EC RW MD5:  8e1aaf3a9b6ca66442923d43407fbd75
[ 176.230] EC RW firmware is unchanged, skip update
[ 176.524] ufs: bRefClkFreq current=0 expected=0
[ 176.595] ufs-spacemit_k3 ufs@c0e00000: [RX, TX]: gear=[3, 3], lane[2, 2], pwr[FAST MODE, FAST MODE], rate = 2
[ 176.656] incorrect device type in mtd_a
[ 176.657] cannot find partition: 'mtd_a'
[ 176.661] incorrect device type in mtd
[ 176.664] cannot find partition: 'mtd'
[ 176.669] incorrect device type in mtd
[ 176.672] cannot find partition: 'mtd'
[ 176.676] Starting download of 965 bytes
[ 176.679]
[ 176.680] downloading/uploading of 965 bytes finished
[ 176.685] Starting fastboot_mtd_flash_write for mtd
[ 176.690] Normal mtd partition ......
[ 176.693] Part info: bootinfo, factory/bootinfo_spinor.bin
[ 176.698] Part info: fsbl, factory/FSBL.bin
[ 176.702] Part info: env, env.bin
[ 176.705] Part info: esos, esos.itb
[ 176.709] Part info: opensbi, fw_dynamic.itb
[ 176.713] Part info: uboot, edk2.itb
[ 176.716] parse gpt/mtd table okayupdata mtd env, table:128K@0K(bootinfo),512K@128K(fsbl),64K@640K(env),1M@704K(esos),384K@1728K(opensbi),-@2112K(uboot)
[ 176.731] ........ erased mtd part
[ 176.869] Saving Environment to mtdENV... OK
[ 177.048] incorrect device type in bootinfo_a
[ 177.049] cannot find partition: 'bootinfo_a'
[ 177.056] Starting download of 80 bytes
[ 177.057]
[ 177.058] downloading/uploading of 80 bytes finished
[ 177.063] Starting fastboot_mtd_flash_write for bootinfo
[ 177.068] Normal mtd partition ......
[ 177.071] fb_mtd_lookup returned 0 for bootinfo
[ 177.076] Erasing MTD partition bootinfo
[ 177.079] ........ erased mtd part
[ 177.082] align erase_size to mtd->erase_size:1000
[ 177.117] need_erase: 1
[ 177.117] Flashing raw image at offset
[ 177.120] ........ wrote 80 bytes to 'bootinfo'
[ 177.125] compare data valid or not
[ 177.128]
download and flash div 0
[ 177.131] get calculate value:ecbb4b55, compare calculate:ecbb4b55
[ 177.137] compare over, use time:9 ms
[ 177.164] incorrect device type in fsbl_a
[ 177.165] cannot find partition: 'fsbl_a'
[ 177.174] Starting download of 473120 bytes
[ 177.194] .
[ 177.194] downloading/uploading of 473120 bytes finished
[ 177.198] Starting fastboot_mtd_flash_write for fsbl
[ 177.202] Normal mtd partition ......
[ 177.205] fb_mtd_lookup returned 0 for fsbl
[ 177.209] Erasing MTD partition fsbl
[ 177.213] ........ erased mtd part
[ 177.216] align erase_size to mtd->erase_size:1000
[ 181.019] need_erase: 1
[ 181.019] Flashing raw image at offset
[ 181.534] ........ wrote 473120 bytes to 'fsbl'
[ 181.535] compare data valid or not
[ 181.540]
download and flash div 0
[ 181.576] get calculate value:b6fadf1e, compare calculate:b6fadf1e
[ 181.579] compare over, use time:39 ms
[ 181.605] incorrect device type in env_a
[ 181.606] cannot find partition: 'env_a'
[ 181.612] Starting download of 16384 bytes
[ 181.615]
[ 181.615] downloading/uploading of 16384 bytes finished
[ 181.621] flash env: saving opensbi_offset=1b0000, esos_offset=b0000, uboot_offset=210000
[ 181.628] Starting fastboot_mtd_flash_write for env
[ 181.633] Normal mtd partition ......
[ 181.636] fb_mtd_lookup returned 0 for env
[ 181.640] Erasing MTD partition env
[ 181.643] ........ erased mtd part
[ 181.783] need_erase: 1
[ 181.783] Flashing raw image at offset
[ 181.799] ........ wrote 16384 bytes to 'env'
[ 181.801] compare data valid or not
[ 181.804]
download and flash div 0
[ 181.808] get calculate value:5d8f80b4, compare calculate:5d8f80b4
[ 181.813] compare over, use time:9 ms
[ 181.817] flash env: importing downloaded environment
[ 181.822] flash env: restoring offsets
[ 181.825] flash env: saving environment
[ 181.829] Saving Environment to mtdENV... OK
[ 182.010] "nor0" partitions still in use, can't delete them
[ 182.014] incorrect device type in esos_a
[ 182.016] cannot find partition: 'esos_a'
[ 182.020] "nor0" partitions still in use, can't delete them
[ 182.028] "nor0" partitions still in use, can't delete them
[ 182.035] Starting download of 447843 bytes
[ 182.054] .
[ 182.054] downloading/uploading of 447843 bytes finished
[ 182.057] Starting fastboot_mtd_flash_write for esos
[ 182.062] Normal mtd partition ......
[ 182.065] "nor0" partitions still in use, can't delete them
[ 182.072] fb_mtd_lookup returned 0 for esos
[ 182.075] Erasing MTD partition esos
[ 182.078] ........ erased mtd part
[ 182.081] align erase_size to mtd->erase_size:1000
[ 185.664] need_erase: 1
[ 185.664] Flashing raw image at offset
[ 186.151] ........ wrote 447843 bytes to 'esos'
[ 186.152] compare data valid or not
[ 186.157]
download and flash div 0
[ 186.191] get calculate value:a67796dc, compare calculate:a67796dc
[ 186.194] compare over, use time:37 ms
[ 186.222] "nor0" partitions still in use, can't delete them
[ 186.226] incorrect device type in opensbi_a
[ 186.229] cannot find partition: 'opensbi_a'
[ 186.233] "nor0" partitions still in use, can't delete them
[ 186.241] "nor0" partitions still in use, can't delete them
[ 186.247] Starting download of 270647 bytes
[ 186.259] .
[ 186.259] downloading/uploading of 270647 bytes finished
[ 186.263] Starting fastboot_mtd_flash_write for opensbi
[ 186.267] Normal mtd partition ......
[ 186.271] "nor0" partitions still in use, can't delete them
[ 186.277] fb_mtd_lookup returned 0 for opensbi
[ 186.280] Erasing MTD partition opensbi
[ 186.284] ........ erased mtd part
[ 186.287] align erase_size to mtd->erase_size:1000
[ 188.483] need_erase: 1
[ 188.483] Flashing raw image at offset
[ 188.778] ........ wrote 270647 bytes to 'opensbi'
[ 188.780] compare data valid or not
[ 188.784]
download and flash div 0
[ 188.806] get calculate value:ca77a998, compare calculate:ca77a998
[ 188.809] compare over, use time:25 ms
[ 188.836] "nor0" partitions still in use, can't delete them
[ 188.840] incorrect device type in uboot_a
[ 188.843] cannot find partition: 'uboot_a'
[ 188.846] "nor0" partitions still in use, can't delete them
[ 188.855] "nor0" partitions still in use, can't delete them
[ 188.874] Starting download of 3215360 bytes
[ 188.895] ......
[ 189.001] downloading/uploading of 3215360 bytes finished
[ 189.004] Starting fastboot_mtd_flash_write for uboot
[ 189.008] Normal mtd partition ......
[ 189.011] "nor0" partitions still in use, can't delete them
[ 189.018] fb_mtd_lookup returned 0 for uboot
[ 189.021] Erasing MTD partition uboot
[ 189.024] ........ erased mtd part
[ 214.742] need_erase: 1
[ 214.742] Flashing raw image at offset
[ 217.009] ........ wrote 3215360 bytes to 'uboot'
[ 217.010] compare data valid or not
[ 217.023]
download and flash div 0
[ 217.255] get calculate value:c5f101ca, compare calculate:c5f101ca
[ 217.258] compare over, use time:235 ms
[ 217.308] "nor0" partitions still in use, can't delete them
[ 217.311] incorrect device type in gpt_a
[ 217.314] cannot find partition: 'gpt_a'
[ 217.318] "nor0" partitions still in use, can't delete them
[ 217.324] incorrect device type in gpt
[ 217.327] cannot find partition: 'gpt'
[ 217.332] "nor0" partitions still in use, can't delete them
[ 217.337] incorrect device type in gpt
[ 217.339] cannot find partition: 'gpt'
[ 217.344] Starting download of 1643 bytes
[ 217.347]
[ 217.348] downloading/uploading of 1643 bytes finished
[ 217.353] init fdev success
[ 217.355] !!!! patr name:env would set to hidden part !!!!
[ 217.361] Part info: env, env.bin
[ 217.364] !!!! patr name:bootinfo would set to hidden part !!!!
[ 217.369] Part info: bootinfo, factory/bootinfo_spinor.bin
[ 217.375] !!!! patr name:fsbl would set to hidden part !!!!
[ 217.380] Part info: fsbl, factory/FSBL.bin
[ 217.384] !!!! patr name:esos would set to hidden part !!!!
[ 217.390] Part info: esos, esos.itb
[ 217.393] !!!! patr name:opensbi would set to hidden part !!!!
[ 217.398] Part info: opensbi, fw_dynamic.itb
[ 217.403] !!!! patr name:uboot would set to hidden part !!!!
[ 217.408] Part info: uboot, edk2.itb
[ 217.411] Part info: esp, esp.vfat
[ 217.415] Part info: cidata, cidata.vfat
[ 217.418] Part info: writable, writable.ext4
[ 217.422] write gpt to dev:scsi
[ 217.425] Writing GPT: success!
[ 217.430] parse gpt/mtd table okay"nor0" partitions still in use, can't delete them
[ 217.457] incorrect device type in env_a
[ 217.460] cannot find partition: 'env_a'
[ 217.464] "nor0" partitions still in use, can't delete them
[ 217.471] "nor0" partitions still in use, can't delete them
[ 217.476] Starting download of 16384 bytes
[ 217.479]
[ 217.480] downloading/uploading of 16384 bytes finished
[ 217.485] flash env: saving opensbi_offset=1b0000, esos_offset=b0000, uboot_offset=210000
[ 217.493] Skip hidden partition 'env' on scsi 0 (exists on NOR)
[ 217.498] flash env: importing downloaded environment
[ 217.503] flash env: restoring offsets
[ 217.507] flash env: saving environment
[ 217.510] Saving Environment to mtdENV... OK
[ 217.694] "nor0" partitions still in use, can't delete them
[ 217.697] incorrect device type in bootinfo_a
[ 217.700] cannot find partition: 'bootinfo_a'
[ 217.704] "nor0" partitions still in use, can't delete them
[ 217.712] "nor0" partitions still in use, can't delete them
[ 217.717] Starting download of 80 bytes
[ 217.719]
[ 217.720] downloading/uploading of 80 bytes finished
[ 217.725] Skip hidden partition 'bootinfo' on scsi 0 (exists on NOR)
[ 217.755] "nor0" partitions still in use, can't delete them
[ 217.758] incorrect device type in fsbl_a
[ 217.761] cannot find partition: 'fsbl_a'
[ 217.765] "nor0" partitions still in use, can't delete them
[ 217.772] "nor0" partitions still in use, can't delete them
[ 217.779] Starting download of 473120 bytes
[ 217.798] .
[ 217.798] downloading/uploading of 473120 bytes finished
[ 217.802] Skip hidden partition 'fsbl' on scsi 0 (exists on NOR)
[ 217.832] "nor0" partitions still in use, can't delete them
[ 217.835] incorrect device type in esos_a
[ 217.838] cannot find partition: 'esos_a'
[ 217.842] "nor0" partitions still in use, can't delete them
[ 217.850] "nor0" partitions still in use, can't delete them
[ 217.857] Starting download of 447843 bytes
[ 217.875] .
[ 217.875] downloading/uploading of 447843 bytes finished
[ 217.879] Skip hidden partition 'esos' on scsi 0 (exists on NOR)
[ 217.909] "nor0" partitions still in use, can't delete them
[ 217.912] incorrect device type in opensbi_a
[ 217.915] cannot find partition: 'opensbi_a'
[ 217.919] "nor0" partitions still in use, can't delete them
[ 217.928] "nor0" partitions still in use, can't delete them
[ 217.933] Starting download of 270647 bytes
[ 217.945] .
[ 217.945] downloading/uploading of 270647 bytes finished
[ 217.949] Skip hidden partition 'opensbi' on scsi 0 (exists on NOR)
[ 217.978] "nor0" partitions still in use, can't delete them
[ 217.981] incorrect device type in uboot_a
[ 217.984] cannot find partition: 'uboot_a'
[ 217.988] "nor0" partitions still in use, can't delete them
[ 217.996] "nor0" partitions still in use, can't delete them
[ 218.013] Starting download of 3215360 bytes
[ 218.034] ......
[ 218.139] downloading/uploading of 3215360 bytes finished
[ 218.142] Skip hidden partition 'uboot' on scsi 0 (exists on NOR)
```

## Preparing PXE on the Pi

Ubuntu 26.10 publishes a RISC-V netboot archive containing GRUB, a kernel, an initrd, and GRUB configuration. I downloaded and extracted it into the dnsmasq TFTP root:

```sh
cd ~/git/k3-host
curl -fLO https://cdimage.ubuntu.com/ubuntu-server/stonking/daily-live/current/stonking-netboot-riscv64.tar.gz
mkdir -p tftp-root
tar -xzf stonking-netboot-riscv64.tar.gz \
  --strip-components=2 \
  -C tftp-root \
  ./riscv64
```

The resulting tree is small:

```text
tftp-root/grubriscv64.efi
tftp-root/grub/grub.cfg
tftp-root/linux
tftp-root/initrd
```

My active dnsmasq configuration binds only to the isolated USB Ethernet interface. The essential parts are:

```ini
interface=enxc84d4433b782
bind-interfaces
dhcp-authoritative
dhcp-range=192.168.254.10,192.168.254.200,255.255.255.0,24h
dhcp-host=50:0a:52:0b:e5:d5,192.168.254.11
dhcp-option=option:router,192.168.254.1
enable-tftp
tftp-root=/home/ubuntu/git/k3-host/tftp-root
dhcp-boot=grubriscv64.efi
```

I validated and restarted it:

```sh
sudo dnsmasq --test
sudo systemctl restart dnsmasq
sudo journalctl -fu dnsmasq
```

## Booting and installing

After leaving BootROM mode and booting normally, I opened `ludovic-k3-1`'s labgrid console and power-cycled it. During EDK2 startup I pressed F2, entered Boot Manager, and selected the IPv4 network entry.

{% include image.md file="install-ubuntu-26.10-on-spacemit-k3/IMG_2145.jpeg" description="Screenshots (literally) of the EDK2 bootloader screen (1)" %}
{% include image.md file="install-ubuntu-26.10-on-spacemit-k3/IMG_2146.jpeg" description="Screenshots (literally) of the EDK2 bootloader screen (2)" %}
{% include image.md file="install-ubuntu-26.10-on-spacemit-k3/IMG_2149.jpeg" description="Screenshots (literally) of the EDK2 bootloader screen (3)" %}
{% include image.md file="install-ubuntu-26.10-on-spacemit-k3/IMG_2151.jpeg" description="Screenshots (literally) of the EDK2 bootloader screen (4)" %}

EDK2 obtained `192.168.254.11`, downloaded `grubriscv64.efi`, and started GRUB. GRUB then fetched `linux` and `initrd` from TFTP. The supplied kernel command line points Subiquity at Canonical's full Ubuntu 26.10 Server ISO, which it downloads over HTTPS through the Pi's NAT connection.

```
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 available DHCP range: 192.168.254.10 -- 192.168.254.200
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 vendor class: PXEClient:Arch:00027:UNDI:003000
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 DHCPDISCOVER(enxc84d4433b782) 50:0a:52:0b:e5:d5
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 tags: known, enxc84d4433b782
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 DHCPOFFER(enxc84d4433b782) 192.168.254.11 50:0a:52:0b:e5:d5
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 1:netmask, 2:time-offset, 3:router, 4, 5,
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 6:dns-server, 12:hostname, 13:boot-file-size,
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 15:domain-name, 17:root-path, 18:extension-path,
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 22:max-datagram-reassembly, 23:default-ttl,
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 28:broadcast, 40:nis-domain, 41:nis-server,
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 42:ntp-server, 43:vendor-encap, 50:requested-address,
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 51:lease-time, 54:server-identifier, 58:T1,
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 59:T2, 60:vendor-class, 66:tftp-server, 67:bootfile-name,
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 97:client-machine-id, 128, 129, 130, 131,
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 132, 133, 134, 135
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 next server: 192.168.254.1
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 broadcast response
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  1 option: 53 message-type  2
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option: 54 server-identifier  192.168.254.1
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option: 51 lease-time  1d
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size: 16 option: 67 bootfile-name  grubriscv64.efi
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option: 58 T1  12h
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option: 59 T2  21h
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option:  1 netmask  255.255.255.0
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option: 28 broadcast  192.168.254.255
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option:  6 dns-server  192.168.1.1
Oct 29 22:54:38 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option:  3 router  192.168.254.1
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 available DHCP range: 192.168.254.10 -- 192.168.254.200
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 vendor class: PXEClient:Arch:00027:UNDI:003000
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 DHCPREQUEST(enxc84d4433b782) 192.168.254.11 50:0a:52:0b:e5:d5
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 tags: known, enxc84d4433b782
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 DHCPACK(enxc84d4433b782) 192.168.254.11 50:0a:52:0b:e5:d5
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 1:netmask, 2:time-offset, 3:router, 4, 5,
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 6:dns-server, 12:hostname, 13:boot-file-size,
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 15:domain-name, 17:root-path, 18:extension-path,
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 22:max-datagram-reassembly, 23:default-ttl,
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 28:broadcast, 40:nis-domain, 41:nis-server,
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 42:ntp-server, 43:vendor-encap, 50:requested-address,
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 51:lease-time, 54:server-identifier, 58:T1,
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 59:T2, 60:vendor-class, 66:tftp-server, 67:bootfile-name,
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 97:client-machine-id, 128, 129, 130, 131,
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 requested options: 132, 133, 134, 135
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 next server: 192.168.254.1
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 broadcast response
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  1 option: 53 message-type  5
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option: 54 server-identifier  192.168.254.1
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option: 51 lease-time  1d
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size: 16 option: 67 bootfile-name  grubriscv64.efi
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option: 58 T1  12h
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option: 59 T2  21h
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option:  1 netmask  255.255.255.0
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option: 28 broadcast  192.168.254.255
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option:  6 dns-server  192.168.1.1
Oct 29 22:54:41 ludovic-pi dnsmasq-dhcp[126343]: 224872522 sent size:  4 option:  3 router  192.168.254.1
Oct 29 22:54:41 ludovic-pi dnsmasq-tftp[126343]: error 8 User aborted the transfer received from 192.168.254.11
Oct 29 22:54:41 ludovic-pi dnsmasq-tftp[126343]: failed sending /home/ubuntu/git/k3-host/tftp-root/grubriscv64.efi to 192.168.254.11
Oct 29 22:54:42 ludovic-pi dnsmasq-tftp[126343]: sent /home/ubuntu/git/k3-host/tftp-root/grubriscv64.efi to 192.168.254.11
Oct 29 22:54:46 ludovic-pi dnsmasq-tftp[126343]: file /home/ubuntu/git/k3-host/tftp-root/grub/riscv64-efi/command.lst not found for 192.168.254.11
Oct 29 22:54:46 ludovic-pi dnsmasq-tftp[126343]: file /home/ubuntu/git/k3-host/tftp-root/grub/riscv64-efi/fs.lst not found for 192.168.254.11
Oct 29 22:54:46 ludovic-pi dnsmasq-tftp[126343]: file /home/ubuntu/git/k3-host/tftp-root/grub/riscv64-efi/crypto.lst not found for 192.168.254.11
Oct 29 22:54:46 ludovic-pi dnsmasq-tftp[126343]: file /home/ubuntu/git/k3-host/tftp-root/grub/riscv64-efi/terminal.lst not found for 192.168.254.11
Oct 29 22:54:46 ludovic-pi dnsmasq-tftp[126343]: sent /home/ubuntu/git/k3-host/tftp-root/grub/grub.cfg to 192.168.254.11
Oct 29 22:55:16 ludovic-pi dnsmasq-tftp[126343]: sent /home/ubuntu/git/k3-host/tftp-root/linux to 192.168.254.11
Oct 29 22:57:56 ludovic-pi dnsmasq-tftp[126343]: sent /home/ubuntu/git/k3-host/tftp-root/initrd to 192.168.254.11
```

I drove the text installer entirely over UART and selected the Samsung SSD 980 NVMe as the installation target. The K3 also exposes its 128 GB internal UFS as `/dev/sda`, so checking model names is important. After installation, the layout was:

```text
/dev/nvme0n1p1  1G       vfat       /boot/efi
/dev/nvme0n1p2  2G       ext4       /boot
/dev/nvme0n1p3  462.7G   LVM PV
ubuntu-vg/ubuntu-lv      ext4       /
```

Subiquity created a 100 GiB root logical volume and left roughly 363 GiB free in the volume group. Rather than repartitioning the active system disk, I extended the logical volume and filesystem online:

```sh
sudo lvextend --resizefs -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

`lvextend --resizefs` assigns every free extent in the volume group to the logical volume and grows ext4 in the same operation. No reboot is required.

```sh
sudo lvs
sudo vgs
df -h /
```

## What I would change

The shared power outlet is the obvious weakness. Independent outlets would let labgrid reboot one board without disturbing the other. I would also pin a dated Ubuntu netboot archive instead of using `current`, so repeated installations use identical bits.

Even with those limitations, the setup now provides a complete headless workflow: labgrid controls power and UART, fastboot updates the platform firmware, EDK2 starts a RISC-V PXE loader, and the Canonical installer deploys Ubuntu 26.10 to NVMe. Once assembled, reinstalling the board no longer requires a monitor, keyboard, or removable storage.