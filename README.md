# ElJoho Pi — Ubuntu Server 26.04 LTS on a Raspberry Pi 5 for ROS 2

A Raspberry Pi 5 running **Ubuntu Server 26.04.1 LTS**, set up for **ROS 2 Lyrical, YOLO, SLAM and OpenCV**, and reachable from any PC over a direct Ethernet cable.

> **Status (2026-11-05):** the Pi `eljohopi` runs Ubuntu 26.04.1 LTS, fully updated, and is reachable over a direct Ethernet cable at `192.168.77.2`. Next step: ROS 2 Lyrical (see [PLAN.md](PLAN.md)).

## Access

| | |
|---|---|
| **SSH** | `ssh eljohopi@192.168.77.2` |
| **User** | `eljohopi` |
| **Password** | `eljohopi` |

The password is published on purpose: this Pi is meant for shared, public use, so password logins stay on. The same password works for `sudo` on the Pi.

> [!WARNING]
> **Never forward port 22 to this Pi.** SSH uses port 22. Don't create a port-forwarding, DMZ or UPnP rule for the Pi on any router, and don't connect it to a network that is reachable from the internet. Because the password is public, anyone on the internet could log in and get full `sudo` control. The same applies to any other service on the Pi, such as NoMachine on port 4000.
>
> Over the cable (`192.168.77.2`) or home Wi-Fi (`192.168.0.16`), SSH stays inside the local network and never goes through the internet. Keep it that way.

## How to connect

1. **Connect the cable.** Plug an Ethernet cable from your PC into the Pi, then power the Pi on and wait about a minute (green LED).
2. **Set your PC's Ethernet port** to the address `192.168.77.1`, mask `255.255.255.0`, no gateway. You only do this once per PC:
   - **Ubuntu:** [Guide 3 — Connect any Ubuntu PC to the Pi with an Ethernet cable](Guides/03-ubuntu-pc-ethernet-connection.md)
   - **Windows / macOS:** [PLAN.md — Option 2: Pi directly to any PC](PLAN.md#option-2-pi-directly-to-any-pc)
3. **Log in:**

   ```bash
   ssh eljohopi@192.168.77.2
   ```

   Enter the password `eljohopi`. Nothing appears while you type it, which is normal.

**The first connection from a PC** asks you to confirm the Pi's identity. The fingerprint should be:

```
SHA256:Wfg76GKg1mLrQLbDDW9ElpJQCNAgNSca4a+DSPwa5Po
```

Answer `yes`. If the Pi's card is ever rewritten, the fingerprint changes. In that case, run `ssh-keygen -R 192.168.77.2` on your PC and connect again.

**To leave the Pi**, type `exit`. **To turn it off**, run `sudo poweroff`, wait for the red LED, then unplug the power.

### Other ways in

| Situation | Command |
|---|---|
| On the Pi's home Wi-Fi | `ssh eljohopi@192.168.0.16`, or `ssh eljohopi@eljohopi.local` once `avahi-daemon` runs |
| Cable to a PC whose network settings you can't change | `ssh eljohopi@fe80::2ecf:67ff:fe6c:4e6e%<port>` ([Guide 3, Part 8](Guides/03-ubuntu-pc-ethernet-connection.md#part-8--no-setup-at-all-the-link-local-address)) |

## Documentation

- **[PLAN.md](PLAN.md)** — the full setup plan: choosing the image, hardware, Imager settings, first boot, networking, remote access, NoMachine, checklist and sources.
- **[`Guides/`](Guides/)** — step-by-step guides:
  1. [Write Ubuntu Server 26.04 LTS to the card with Raspberry Pi Imager](Guides/01-write-ubuntu-server-with-pi-imager.md)
  2. [First boot and setting up the Pi](Guides/02-first-boot-and-pi-setup.md)
  3. [Connect any Ubuntu PC to the Pi with an Ethernet cable](Guides/03-ubuntu-pc-ethernet-connection.md)
