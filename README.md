# openwrt-jcg-q20-pb-boot

Dockerized build environment for producing OpenWrt firmware for the
**JCG Q20** with the **pb-boot** bootloader.

The actual OpenWrt source (with the pb-boot patch) lives in a separate
repo: [helscape/openwrt](https://github.com/helscape/openwrt/tree/jcg-q20-pb-boot), branch
`jcg-q20-pb-boot`.

## Hardware

- SoC: MediaTek MT7621AT
- RAM: 256 MiB
- Flash: 128 MiB NAND
- WiFi: MT7915 (2.4 GHz + 5 GHz, 2T2R)
- Ethernet: 3× Gigabit
- Bootloader: pb-boot (web recovery at 192.168.1.1)

## Requirements

- Docker or Podman
- Linux host (or macOS with OrbStack / Docker Desktop)
- ~25 GB free disk space

## Build

1. Clone the OpenWrt fork with the pb-boot patch:

       git clone -b jcg-q20-pb-boot \
         https://github.com/helscape/openwrt.git openwrt

2. Build the Docker image:

       HOST_UID=$(id -u) HOST_GID=$(id -g) docker compose build

3. Enter the build container:

       docker compose run --rm openwrt-builder

4. Inside the container:

       ./scripts/feeds update -a
       ./scripts/feeds install -a
       make menuconfig    # Target: ramips/mt7621, Profile: JCG Q20 (pb-boot)
       make -j$(nproc)

## Output

    openwrt/bin/targets/ramips/mt7621/
        openwrt-ramips-mt7621-jcg_q20-pb-boot-squashfs-factory.bin
        openwrt-ramips-mt7621-jcg_q20-pb-boot-squashfs-sysupgrade.bin
        openwrt-ramips-mt7621-jcg_q20-pb-boot-initramfs-kernel.bin

## Verified hashes

See [`pb-boot-build-hashes.txt`](./pb-boot-build-hashes.txt).

## Flashing

- **From pb-boot:** power off, hold Reset, power on, browse to
  `http://192.168.1.1`, upload the `-factory.bin`.
- **From a running OpenWrt:** `sysupgrade -n <image>` on the device.

## Bootloader layouts — do not cross-flash

| | U-Boot | pb-boot |
|---|---|---|
| `board_name` | `jcg,q20` | `jcg,q20-pb-boot` |
| kernel MTD | mtd3, 4 MB | mtd6, 4 MB |
| rootfs | `ubiconcat0` + `ubiconcat1` | `ubi` (single) |
| total MTD | 10 | 8 |
| recovery | TFTP `recovery.bin` from 192.168.1.2 | web UI at 192.168.1.1 |

Flashing an image built for one layout onto the other will result in a
kernel that loads but panics when it can't mount rootfs.

## Notes

- Built against OpenWrt **v25.12.2**.
- pb-boot support comes from ImmortalWrt commit `203604769b`
  ("ramips: jcg q20: add pb-boot variants"), cherry-picked onto the
  OpenWrt stable tag.
- The device tree for the pb-boot variant uses a standalone `.dts`
  (no shared `.dtsi`) to avoid stale bindings from older forks.
- `nvmem-layout` inside the `Factory` partition supplies WiFi EEPROM
  and both MAC addresses; without it, WiFi and Ethernet come up with
  random MACs.
