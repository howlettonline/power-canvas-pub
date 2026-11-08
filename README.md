# Power Canvas Hub

The Power Canvas Hub is a small service that runs on a Raspberry Pi (or another always-on Linux machine) in your home. It reads your solar battery inverter over your local network and sends the readings to [Power Canvas](https://portal.powercanvas.app). Where you allow it, it also applies the battery charge settings that Power Canvas works out, so Power Canvas does not have to go through the manufacturer's cloud.

This repository holds the install guide and the release downloads for the Hub. It is not the Hub's source code. Each release has a bundle (`power-canvas-hub.tar.gz`) with the files that start the Hub, and the Hub's container image is published as [`ghcr.io/howlettonline/power-canvas-hub`](https://github.com/users/howlettonline/packages/container/package/power-canvas-hub).

## Contents

- [What you need](#what-you-need)
- [Install](#install)
- [Settings](#settings)
- [Upgrade](#upgrade)
- [Uninstall](#uninstall)
- [Troubleshooting](#troubleshooting)
- [What the Hub sends and changes](#what-the-hub-sends-and-changes)
- [Questions and problems](#questions-and-problems)

## What you need

**A Power Canvas location, and the Owner role on it.** Only an Owner can add a Hub.

**A supported inverter.** The Hub only works with models it has been tested with, and does nothing at all with any other. Today that is the GivEnergy GIV-HY3.6 Gen 3, read through its WiFi or Ethernet dongle. It writes to an inverter only after an Owner turns writes on for it in Power Canvas.

**A computer to run it on, switched on all the time.** A Raspberry Pi 4 or 5 with a 64-bit operating system (Raspberry Pi OS, desktop or Lite) is the intended setup, and an SSD is better than an SD card for something that never stops writing. Any 64-bit Linux machine (ARM or x86-64) with Docker works too.

**Docker with Compose.**

- Install Docker Engine and its Compose plugin by following [Docker's instructions for Debian](https://docs.docker.com/engine/install/debian/). Raspberry Pi OS is Debian.
- Let your user run Docker without `sudo`, then sign out and back in:

  ```bash
  sudo usermod -aG docker $USER
  ```

- Check both work: `docker compose version` and `uname -m`, which should say `aarch64` on a Pi or `x86_64` on a PC.

**A network that lets it reach the inverter and Power Canvas.**

- The Hub's computer must reach the inverter's dongle on your local network, on TCP port 8899. The dongle copes badly with several clients polling it at once, so avoid running other tools against it at the same time (the Hub's own GivTCP counts as one).
- It only makes outbound connections, to Power Canvas and its Azure IoT Hub (ports 443 and 8883) and to time servers. Nothing needs to be opened or forwarded into your network, and the Hub publishes no ports.

**An accurate clock.** The Hub stamps each reading with the computer's clock and does not trust a clock that is not synchronised. Raspberry Pi OS keeps time through the network by default. Check with:

```bash
timedatectl show -p NTPSynchronized --value
```

which should say `yes`.

## Install

1. Download the latest release and start it. This makes a `power-canvas-hub` folder in the current directory (for example your home directory):

   ```bash
   curl -fsSL https://github.com/howlettonline/power-canvas-pub/releases/latest/download/power-canvas-hub.tar.gz | tar -xz
   cd power-canvas-hub
   mkdir -p secrets && touch secrets/hub_activation
   docker compose up -d
   ```

   The first start downloads the images, which takes a few minutes. The `secrets/hub_activation` file must exist but stays empty; it is only used for an activation code (see [Settings](#settings)). The containers restart by themselves after a reboot.

2. Find the Hub's claim code. It appears in the log a minute or so after the start:

   ```bash
   docker compose logs hub | grep "Claim code"
   ```

   It looks like `Claim code K7Q2M-9XWDT`. The Hub makes its own identity the first time it starts, and the code is how Power Canvas recognises it.

3. In Power Canvas, open **Settings > Hubs & devices**, choose **Add Hub**, then **I have its claim code**, and enter the code and a name. Within a minute or two the Hub shows as Online.

4. On the same tab, add the inverter as a device with its serial number, and give it the Hub as its manager. Leave writes off to begin with. The Hub checks that the inverter is a tested model, and Power Canvas then shows the result. Turn writes on in the device's settings only when you want Power Canvas to change the inverter's charge settings.

5. Check that all three containers are running, and watch the log:

   ```bash
   docker compose ps
   docker compose logs -f hub
   ```

   The log says `Connected to the local bus`, and then `Sending snapshots of inverter …` once the first readings arrive. Press Ctrl+C to stop watching; that does not stop the Hub.

## Settings

Most people change nothing. Every setting has a default, so there is no settings file to create.

To change one, copy `.env.example` to `.env` in the `power-canvas-hub` folder, edit it, and run `docker compose up -d` so the containers pick it up. Upgrades never overwrite `.env`. Useful settings:

| Setting | Default | What it does |
| --- | --- | --- |
| `HUB_FORCE_READ_ONLY` | `false` | `true` stops the Hub changing the inverter, whatever Power Canvas says. |
| `HUB_LOG_LEVEL` | `INFO` | `DEBUG` makes the log more detailed. |
| `GIVTCP_POLL_SECONDS` | `30` | How often the inverter is polled. Raise it if something else shares the dongle. |

The inverter's address, serial number and the location's time zone are not settings here: they come from Power Canvas once the Hub is claimed.

**An activation code** is the other way to connect a Hub, for a Hub that cannot be claimed. An Owner makes one in Power Canvas and puts it in `secrets/hub_activation`. Treat it like a password, and never post it anywhere. Restart the Hub afterwards with `docker compose restart hub`.

## Upgrade

Each release names the exact version of the Hub it starts, and the Hub never updates itself. To upgrade, read the [release notes](https://github.com/howlettonline/power-canvas-pub/releases) first, then run this from the folder that holds `power-canvas-hub`:

```bash
curl -fsSL https://github.com/howlettonline/power-canvas-pub/releases/latest/download/power-canvas-hub.tar.gz | tar -xz
cd power-canvas-hub
docker compose pull
docker compose up -d
```

The bundle replaces `compose.yaml`, `.env.example` and the `mosquitto` and `givtcp` folders, so do not edit those. It never touches your `.env`, your `secrets` folder or the Hub's data, so the Hub keeps its identity and any readings it has not yet sent.

**To go back to an earlier version,** or to install a particular one, replace `latest/download` in the address with `download/vX.Y.Z` (for example `download/v1.2.3`) and run the same commands. The release notes mention anything that stops a Hub going back.

## Uninstall

1. **In Power Canvas, retire the Hub** under **Settings > Hubs & devices**. This hands its devices over and removes the Hub from Power Canvas's cloud, which is what stops it being counted as lost. If the location's inverter control is set to the Hub, switch it back to the GivEnergy cloud first; Power Canvas will not let you retire the Hub until then.

2. **On the Hub's computer,** stop it and delete everything it made:

   ```bash
   cd power-canvas-hub
   docker compose down -v --rmi all
   cd ..
   rm -rf power-canvas-hub
   ```

   `-v` deletes the Hub's identity, its settings and any readings it had not yet sent. `--rmi all` deletes the downloaded images. This cannot be undone.

To stop the Hub for a while and keep everything, run `docker compose down` instead, with neither `-v` nor `--rmi`. `docker compose up -d` starts it again.

## Troubleshooting

Start with the log of the whole stack: `docker compose logs --tail 100`.

| What you see | What to check |
| --- | --- |
| No claim code in the log | The Hub needs a minute to start. If it is still missing, look for errors in `docker compose logs hub`, and check that the computer can reach the internet. |
| The Hub never shows as Online | That the code was entered correctly, and that the computer's clock is synchronised. The Hub tries to connect every minute. |
| The log keeps reporting a clock problem | The computer's clock is not synchronised (see [What you need](#what-you-need)). |
| `... is not a model this Hub has been tested with` | The Hub deliberately ignores an inverter model it has not been tested with. |
| No inverter readings | That the device's serial number is right in Power Canvas, and that the computer can reach the dongle: `nc -zv <the dongle's address> 8899`. Another tool polling the dongle can cause failures. |
| A container keeps restarting | `docker compose ps`, then `docker compose logs <name>` for the one that is. |

## What the Hub sends and changes

- **Sends.** Inverter readings (power, battery level and energy totals), its own health and events, and the inverter's clock drift. It sends them to Power Canvas through Azure IoT Hub. Readings that cannot be sent wait on the Hub and are sent when the connection returns.
- **Changes.** Nothing, unless an Owner turns writes on for that inverter in Power Canvas. Then it changes only the battery's charge targets, within the range it has been tested with. It never sets the inverter's clock, and it never changes anything on a model it has not been tested with. `HUB_FORCE_READ_ONLY=true` stops it changing anything whatever Power Canvas says.
- **Stores.** The Hub's private key stays in a Docker volume on its computer and is never sent anywhere. It keeps no readings except those waiting to be sent.
- **Network.** No ports are published, so it can only be reached from the computer it runs on.

## Questions and problems

Open an [issue](https://github.com/howlettonline/power-canvas-pub/issues). Before you paste a log, remove your inverter's serial number and your IP addresses, and never paste an activation code.
