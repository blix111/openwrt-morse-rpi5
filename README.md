# OpenWrt Morse HaLow Firmware for Raspberry Pi 5

OpenWrt-based Morse Micro HaLow firmware targeting the Raspberry Pi 5. Brings sub-GHz long-range 802.11ah (HaLow) Wi-Fi to the Pi 5 via Morse Micro radios.

This repo is a community backport. The [Morse Micro OpenWrt SDK](https://github.com/MorseMicro/openwrt) ships on OpenWrt 23.05 (kernel 5.15) and supports Pi 4. The bcm2712/RP1 hardware definitions from upstream OpenWrt 24.10 (kernel 6.6) have been vendored back so the same Morse SDK can build for Pi 5.

**Software Specifications**
- Morse Micro OpenWrt SDK, 23.05.5 base
- Linux kernel 6.6 (vendored from openwrt-24.10)
- mac80211 backports 6.12.61
- Morse Micro driver 1.16.4

## Supported Hardware

| Device                          | Chip   | Interface         | Status                     |
|---------------------------------|--------|-------------------|----------------------------|
| Seeed Studio HaLow HAT          | MM6108 | SPI               | Does not bind yet          |
| Gateworks MM8108                | MM8108 | USB (HAT-to-Pi)   | Working                    |

The original goal is the Seeed Studio HaLow HAT on Pi 5 over SPI — that path is still open. The DesignWare SPI / RP1 DMA / Morse SPI overlay are in place but the MM6108 chip isn't binding yet on Pi 5. As a working alternative, the MM8108 currently runs on Pi 5 via a USB cable from the HaLow board to the Pi.

## Building

Tested on Ubuntu 22.04 / 24.04.

### 1. Install build dependencies

```
sudo apt update
sudo apt install build-essential clang flex g++ gawk gcc-multilib g++-multilib git gettext \
  libncurses5-dev libssl-dev python3-setuptools rsync unzip zlib1g-dev swig file wget \
  libnl-3-dev libnl-genl-3-dev pkg-config
```

### 2. Clone

```
git clone -b rpi5-mm-23.05 https://github.com/buildwithparallel/openwrt-morse-rpi5.git
sudo chown -R $USER:$USER openwrt-morse-rpi5
cd openwrt-morse-rpi5
```

The `chown` ensures every file in the cloned tree is owned by your user. OpenWrt's build system refuses to run as root and trips over root-owned files, so this avoids permission errors later.

### 3. Configure feeds and pick the board

```
./scripts/morse_setup.sh -i -b ekh-bcm2712
```

This updates and installs the Morse HaLow / OpenWrt / LuCI feeds, then assembles a `.config` for the Pi 5 target.

### 4. Download sources

```
make -j$(nproc) download
```

### 5. Build

```
make -j$(nproc) V=sc 2>&1 | tee log.txt
```

`V=sc` enables verbose compile output; `tee log.txt` captures it for troubleshooting.

When the build finishes, the image is at:

```
bin/targets/bcm27xx/bcm2712/openwrt-bcm27xx-bcm2712-rpi-5-squashfs-factory.img.gz
```

Flash to an SD card with [Raspberry Pi Imager](https://www.raspberrypi.com/software/) (choose "Use custom" and select the `.img.gz`) and boot the Pi 5.

## Upstream sources

- Morse Micro OpenWrt SDK: https://github.com/MorseMicro/openwrt (23.05.5 base, branch `mm/v23.05.5`)
- Morse Micro package feed: https://github.com/MorseMicro/morse-feed
- Morse Micro driver: https://github.com/MorseMicro/morse_driver
- Upstream OpenWrt 24.10 (Pi 5 hardware donor): https://github.com/openwrt/openwrt
