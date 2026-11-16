# INEAUTO / OIBIRO OWE3000 — OpenWrt device notes

Outdoor dual-band **AX3000 Wi-Fi 6** range extender / access point based on
**MediaTek MT7981B** (Filogic 820). White-label / ODM device: manufacturer
**INEAUTO** ([ineauto.com](https://ineauto.com)); also sold on Amazon as
**OIBIRO OWE3000** (ASIN `X004WL0VML`). PCB silkscreen: **CPE-AX3000 V3.1**.

- OpenWrt PR: https://github.com/openwrt/openwrt/pull/26757
- OEM manual: https://ineauto.com/pages/manual

> This document is the author's working record of adding OpenWrt support.
> The canonical OpenWrt device page lives at `toh/ineauto/owe3000` on the
> OpenWrt wiki.

---

## Hardware

| Function | Component |
|---|---|
| SoC | MediaTek MT7981B, 2× Cortex-A53 @ 1.3 GHz |
| RAM | 256 MB DDR3 (Nanya NT5CB128M16HP-DI) |
| Flash | 16 MB SPI-NOR (Winbond W25Q128) |
| WiFi | MediaTek MT7976, 2×2 2.4 GHz + 2×2 5 GHz (AX3000) |
| Ethernet 2.5G | Realtek RTL8221B PHY, "LAN1" (PoE-in), `eth0` |
| Ethernet 1G | MT7981 built-in GbE PHY, "LAN2/WAN", `eth1` |
| Antennas | 6× external detachable (RP-SMA); 5× u.FL leads on PCB |
| LEDs | status + a 7-segment signal-strength display |
| Button | reset |
| Power | 24 V passive PoE (24 V 1 A injector) or 9–24 V DC barrel |
| Enclosure | IP65 (box front; Amazon listing states IP67) |

Switch: none — two direct MAC↔PHY links. The front 7-segment digit shows
link/RSSI status in stock firmware (H = router mode, 0 = no link,
3/5/7/9 = increasing signal); under OpenWrt the segment GPIOs are LEDs.

---

## Serial / UART

The device has an **unpopulated 4-pin UART header** near the lower edge of
the PCB, beside the "100" inductor. **The pads are not labelled on the board.**
The pinout below was determined with a multimeter and confirmed by using the
console successfully.

Orient the board so the silkscreen **"CPE-AX3000 V3.1" reads upright**. With
the board in that orientation the header runs left-to-right, pin 1 leftmost:

| Pin | Signal | Confirmed |
|---|---|---|
| 1 | GND | continuity to ground plane / RJ45 shield |
| 2 | TX (device → adapter) | idles ~3.3 V, bursts during boot |
| 3 | RX (adapter → device) | idles ~3.3 V, no boot traffic |
| 4 | VCC, **+3.3 V** | measured **3.31 V**; **do not connect** for console |

Logic level **3.3 V TTL**; console **115200 8N1**.

Hook up a 3.3 V USB-TTL adapter, crossing TX/RX:
- device TX (pin 2) → adapter RX
- device RX (pin 3) → adapter TX
- device GND (pin 1) → adapter GND
- **leave pin 4 (+3.3 V) unconnected**

Adapter colour note: the adapter used was an FTDI-type labelled Red=5V,
Black=GND, White=RXD, Green=TXD. Its RXD/TXD are named from the adapter's
side, so the working hookup is White (adapter RXD) → device pin 2 (TX),
Green (adapter TXD) → device pin 3 (RX), Black → pin 1 (GND); the red 5 V
lead is left disconnected.

---

## Flash layout (SPI-NOR, 16 MB)

| MTD | Offset | Size | Name |
|---|---|---|---|
| mtd0 | 0x000000 | 0x040000 | BL2 |
| mtd1 | 0x040000 | 0x010000 | u-boot-env |
| mtd2 | 0x050000 | 0x0b0000 | **Factory** |
| mtd3 | 0x100000 | 0x080000 | FIP |
| mtd4 | 0x180000 | 0xe00000 | firmware |

**Factory** holds the WiFi calibration EEPROM and MAC addresses and is unique
per unit — back it up and keep it safe.

| Interface | MAC source |
|---|---|
| 2.5G (`eth0`) | Factory `0x4` (base) |
| 1G (`eth1`) | Factory `0x4` (base) + 1 |
| WiFi | derived from Factory EEPROM |

---

## Installation

Stock firmware has no unlocked upgrade path. Install over serial using the
vendor U-Boot's TFTP **receive** support. The vendor U-Boot has `tftpboot`
but **not** `tftpput`, so stock flash can't be pushed out over the network
from U-Boot — back it up from the booted OpenWrt initramfs instead.

1. Attach a 3.3 V USB-TTL adapter to the UART header (115200 8N1, GND/TX/RX;
   leave pin 4 unconnected). Power on.
2. Interrupt boot to reach the U-Boot console (`MT7981>`). U-Boot drives the
   2.5G "LAN1" port for TFTP — use that jack.
3. Boot the OpenWrt initramfs into RAM (adjust addresses to your network):
   ```
   setenv serverip 192.168.1.2
   setenv ipaddr 192.168.1.1
   tftpboot 0x46000000 openwrt-mediatek-filogic-ineauto_owe3000-initramfs-kernel.bin
   bootm 0x46000000
   ```
4. From booted OpenWrt, back up the original flash before writing anything.
   At minimum keep Factory:
   ```
   dd if=/dev/mtd2 of=/tmp/Factory.bin
   scp /tmp/Factory.bin user@host:
   ```
   (Repeat for mtd0/mtd1/mtd3/mtd4 for a full recovery set.)
5. Flash the sysupgrade image:
   ```
   sysupgrade -n /tmp/openwrt-mediatek-filogic-ineauto_owe3000-squashfs-sysupgrade.bin
   ```

### Recovery

The vendor U-Boot and its TFTP recovery stay intact, so a bad flash is
recoverable: re-boot the initramfs over TFTP and re-flash. If calibration/
MACs are lost, write `Factory.bin` back from U-Boot:
```
tftpboot 0x46000000 Factory.bin
sf probe
sf erase 0x50000 0xb0000
sf write 0x46000000 0x50000 0xb0000
```

---

## Photos

Place images in `img/` and reference them here:

| File | Shows |
|---|---|
| `img/owe3000_enclosure.jpg` | Enclosure with six antennas (ruler for scale) |
| `img/owe3000_ports.jpg` | Port row: LAN1 2.5G PoE, 7-seg display, LAN2/WAN 1G PoE, DC |
| `img/owe3000_label.jpg` | Bottom label (ports, modes, LED codes) |
| `img/owe3000_board.jpg` | PCB top (CPE-AX3000 V3.1) |
| `img/owe3000_uart_header.jpg` | Unlabelled 4-pin UART header |
| `img/owe3000_uart_hookup.jpg` | USB-TTL adapter connected (GND/TX/RX) |
| `img/serial_app.jpg` | Serial terminal software used (Serial.app) |

---

## Notes

- White-label/ODM; also sold as **OIBIRO OWE3000**. Same board may appear
  under further brands.
- Vendor docs disagree: IP65 (box) vs IP67 (Amazon); antenna 7 dBi vs 8 dBi.
- The ethernet node currently emits two cosmetic "legacy DT" kernel messages;
  both Ethernet ports work normally.
- U-Boot Ethernet works on the 2.5G "LAN1" port; use that jack for TFTP.
