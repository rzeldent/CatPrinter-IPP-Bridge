# Kitty IPP Bridge

CatPrinter IPP Bridge runs on an ESP32 and connects compatible Bluetooth Low
Energy (BLE) thermal printers to a Wi-Fi network. It discovers printers by
their BLE service UUIDs and exposes them as IPP printers using mDNS/DNS-SD.
The web dashboard shows discovered printers and print jobs and provides
settings, logs, and a test-page action.

The current printer profile supports classic Cat Printer devices using the
AE30 service (including devices advertising AF30), such as the GB0x, GT01,
MX0x, and YT01 families. Other protocols are not currently supported.
IPP jobs must be PWG Raster (`image/pwg-raster`) or Apple Raster
(`image/urf`, used by AirPrint), or one of these sent as
`application/octet-stream`; the bridge converts them to monochrome bitmaps.

## Requirements

- An ESP32-S3 development board with octal PSRAM (such as N16R8) and a USB data cable.
- A compatible BLE Cat Printer.
- A 2.4 GHz Wi-Fi network and a computer or phone on the same network.
- [PlatformIO](https://platformio.org/install) to build and upload the firmware.

## Build and flash

Connect the ESP32 by USB and run this command from the project
directory:

```sh
# ESP32-S3 with octal PSRAM, such as N16R8
pio run -e esp32s3-psram -t upload
```

The upload command builds the firmware first. The default environment is
`esp32s3-psram`; boards without PSRAM are not supported. Startup checks that
PSRAM is detected and halts with a serial error before starting Wi-Fi, BLE or
printing services if it is unavailable. If PlatformIO cannot find the board,
specify its serial port with `--upload-port`.

## Connect to Wi-Fi

1. On first startup, connect your computer or phone to the setup Wi-Fi network
   named `CatBridge-XXXX`. Its default password is `catbridge`.
2. Open [http://192.168.4.1/](http://192.168.4.1/) in a browser. If the
   address differs, check the setup access point address in the serial log.
3. Under **Network**, enter your Wi-Fi SSID and password. You can also change
   the hostname and setup access point password here.
4. Click **Save**, then **Reboot** to apply the network settings. The settings
   are stored on the device.
5. Reconnect your computer or phone to your usual Wi-Fi network. Open
   [http://catbridge.local/](http://catbridge.local/) (or use your chosen
   hostname) or use the IP address found below.

If the bridge cannot connect within about 20 seconds, it starts the setup
network again so you can correct the settings. It continues retrying the
configured network and disables the setup network after the connection has
been stable for about a minute.

The dashboard loads Bootstrap from a CDN. Without internet access it may appear
unstyled, and some controls may be unavailable.

The dashboard source is in [html/index.html](html/index.html). PlatformIO embeds
it in the firmware, so rebuild and upload after changing it.

## Find the IP address

Connect the ESP32 by USB and open the PlatformIO serial monitor at 115200 baud:

```sh
pio device monitor -b 115200
```

Restart the board if necessary and look for `Connected, IP ...` in the log.
The setup access point's name and address are also logged. Alternatively,
check your router's connected-device list for the bridge's hostname
(`catbridge` by default).

## Add a printer

1. Turn on a compatible printer near the bridge. The dashboard scans
   periodically; use **Scan now** to request a scan immediately.
2. Find the printer in the dashboard and use **Test page** to check it. If it
   is not discovered, check the logs and try enabling **Probe devices without
   full UUID list** in **Bridge settings**.
3. Add the discovered IPP printer using your operating system's printer
   settings. Where mDNS/DNS-SD discovery is available, the printer can appear
   automatically; otherwise use the **IPP URI** shown in the dashboard. The
   URI has the form `ipp://catbridge.local:631/ipp/<printer-id>`. Replace the
   hostname with the bridge IP if `.local` name resolution is unavailable.

### Windows 11

1. Open **Settings > Bluetooth & devices > Printers & scanners** and select
   **Add device**. The printer appears under its dashboard name when Windows
   finds it through mDNS.
2. If it does not appear, select **Add manually**, then **Add a printer using
   an IP address or hostname**. Set **Device type** to **IPP Device** and enter
   the URI with `http://` instead of `ipp://`, preferably using the bridge IP:
   `http://<bridge-ip>:631/ipp/<printer-id>`.
3. Windows installs its built-in IPP class driver; no vendor driver is needed.

Windows discovers printers only on the same subnet, and only when network
discovery is enabled (the default for networks set to **Private**).

### iPhone and iPad (AirPrint)

The bridge advertises each printer as an AirPrint printer. In an app, select
**Share > Print**, then **Printer**; the printer appears under its dashboard
name. The iPhone or iPad must be on the same Wi-Fi network and subnet as the
bridge, and the network must allow multicast (mDNS); guest networks and
client isolation often block it. After a firmware update, reboot the bridge so
it advertises the new printer details.

### Keep printers awake

Cat Printers power themselves off after a period of inactivity. To prevent
this, enable **Keep printers awake** in **Bridge settings**. While no job is
printing, the bridge then briefly connects to each online printer at the set
**Keep-alive interval** (default 120 seconds) and requests its status; nothing
is printed. Choose an interval shorter than the printer's power-off time.
A printer that is already off cannot be woken over Bluetooth; turn it on
again with its power button. Keeping printers awake drains their batteries
faster.

BLE transfers enforce at least 20 ms between writes, including when older saved
settings specify a shorter delay. Printer pause notifications still stop the
transfer; a pause lasting more than 30 seconds fails the job rather than sending
through backpressure. Enable debug logging to see pause/resume notifications.

The web dashboard and IPP service are intended for a trusted network; the
dashboard does not require a login. Change the default setup access point
password and avoid exposing the bridge to the internet.

## Run native tests

```sh
pio test -e native
```
