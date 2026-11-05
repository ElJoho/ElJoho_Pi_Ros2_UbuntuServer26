# Setup plan: Ubuntu Server 26.04 LTS on a Raspberry Pi 4/5 for ROS 2

The full plan for running **ROS 2 Lyrical, YOLO, SLAM and OpenCV** on a Raspberry Pi 4 or 5 with **Ubuntu Server 26.04 LTS**, managed from a dual-boot (Ubuntu / Windows) PC. It covers choosing the image, hardware, writing the card, first boot, networking, remote access and a checklist.

> Facts checked on 2026-11-04/05 against Ubuntu 26.04.1, ROS 2 Lyrical and Raspberry Pi Imager 1.8.5.
> The examples use the hostname `rpi-ros` and `<user>`; on this project's Pi, both are `eljohopi`.
> Login details are in the [README](README.md), and step-by-step instructions are in [`Guides/`](Guides/).

## Contents

1. [Choose the right image](#1-choose-the-right-image)
2. [Hardware and compatibility](#2-hardware-and-compatibility)
3. [Write the card with Raspberry Pi Imager](#3-write-the-card-with-raspberry-pi-imager)
4. [First boot](#4-first-boot)
5. [Network: Wi-Fi or Ethernet cable](#5-network-wi-fi-or-ethernet-cable)
6. [Terminal vs graphical access](#6-terminal-vs-graphical-access)
7. [Remote access from Ubuntu and Windows](#7-remote-access-from-ubuntu-and-windows)
8. [NoMachine on Ubuntu Server](#8-nomachine-on-ubuntu-server)
9. [Step-by-step checklist](#9-step-by-step-checklist)
10. [Sources](#10-sources)

---

## 1. Choose the right image

**Correct image (downloaded and verified on 2026-11-04):** `ubuntu-26.04.1-preinstalled-server-arm64+raspi.img.xz` (1,528,288,776 bytes). It's a ready-to-run disk image with no installer, and Ubuntu lists it for the Pi 3, 4, 5, CM4 and Zero 2 W. Its checksum matches Ubuntu's `SHA256SUMS`.

| File | Link |
|---|---|
| Image | <https://cdimage.ubuntu.com/ubuntu/releases/26.04.1/release/ubuntu-26.04.1-preinstalled-server-arm64+raspi.img.xz> |
| Checksums | <https://cdimage.ubuntu.com/ubuntu/releases/26.04.1/release/SHA256SUMS> |
| Release folder | <https://cdimage.ubuntu.com/ubuntu/releases/26.04.1/release/> |

To download it and check it isn't corrupted:

```bash
cd ~/Documents/images
wget https://cdimage.ubuntu.com/ubuntu/releases/26.04.1/release/ubuntu-26.04.1-preinstalled-server-arm64+raspi.img.xz
wget https://cdimage.ubuntu.com/ubuntu/releases/26.04.1/release/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing     # expect: ...raspi.img.xz: OK
```

### Why not the first ISO

The first download, `ubuntu26arm64.iso`, **does not boot on a Raspberry Pi.**

| Check | Result |
|---|---|
| Volume label | `Ubuntu-Server 26.04.1 LTS arm64` (build 20260826) |
| What the file really is | `ubuntu-26.04.1-live-server-arm64.iso`, the generic ARM64 *installer* |
| How it boots | UEFI only (`EFI/boot/bootaa64.efi`, GRUB) |
| Raspberry Pi boot files (`start4.elf`, `config.txt`, `bcm2711`/`bcm2712` device trees) | none |

The Pi 4 and Pi 5 have no UEFI firmware. Their bootloader (stored in an EEPROM chip on the board) looks for Pi-specific firmware files on a FAT partition. That ISO is meant for UEFI ARM servers and virtual machines.

To check any image yourself (read-only):

```bash
file image.iso                                              # type and label
blkid -p image.iso                                          # label and partition table
7z l image.iso | grep -iE 'start4\.elf|config\.txt|bcm271[12]'   # no output = not a Pi image
```

> **Repo warning:** disk images in this folder are untracked and not covered by `.gitignore`. The `.img.xz` now in the repo root (1.5 GB) would be staged by `git add .`, and GitHub rejects files over 100 MB. Move it to `~/Documents/images/`, or add `*.iso`, `*.img` and `*.img.xz` to `.gitignore`.

---

## 2. Hardware and compatibility

The software lines up: the PC runs Ubuntu 26.04.1 with `ros-lyrical-desktop`, and ROS 2 Lyrical is fully supported (Tier 1) on Ubuntu 26.04 for arm64. The Pi can therefore install the same ROS version from apt (`ros-lyrical-ros-base`). Matching versions on both machines is what makes ROS 2 networking work smoothly.

| | Pi 4 | **Pi 5 (recommended)** |
|---|---|---|
| CPU | 4× Cortex-A72, 1.5–1.8 GHz | 4× Cortex-A76, 2.4 GHz (about 2–3× faster) |
| RAM | 4 GB minimum, 8 GB better | 8 GB (16 GB if you run YOLO and SLAM together) |
| Power | 5.1 V / 3 A USB-C (15 W) | 5.1 V / 5 A USB-C PD (27 W official supply) |
| Cooling | heatsink + fan | Active Cooler (long YOLO runs slow down from heat without it) |
| Fast storage | USB 3 SSD | NVMe SSD via M.2 HAT+, or USB 3 SSD |

- **Storage:** start with a 64 GB+ A2 microSD card. Move to an SSD later for speed and durability (builds, rosbags, datasets). Booting from an SSD needs a recent bootloader and a changed boot order (Imager → Misc utility images → Bootloader).
- **ROS 2:** runs well on both boards. The Pi runs without a desktop, and RViz2 runs on the PC.
- **SLAM:** `slam_toolbox` with a 2D LiDAR runs fine on either board. Visual or 3D SLAM (for example RTAB-Map) needs a Pi 5 with 8 GB or more.
- **YOLO:** runs on the CPU only. Use nano models exported to NCNN or ONNX. Expect roughly 5–10 FPS at 640 px on a Pi 5, and noticeably less on a Pi 4 (ballpark figures). An AI HAT+ (Hailo accelerator, Pi 5 only) helps a lot, but its software is aimed at Raspberry Pi OS first, so check Ubuntu support before buying.
- **OpenCV:** install it from apt (`python3-opencv`, `libopencv-dev`), because ROS's `cv_bridge` is built against that copy. Ubuntu 26.04 ships Python 3.14. Install Ultralytics and PyTorch in a virtual environment that can still see the system packages, and make sure pip doesn't add a second OpenCV or NumPy that clashes with ROS:

  ```bash
  python3 -m venv --system-site-packages ~/venvs/yolo
  ```

- **Camera:** a USB webcam just works (V4L2, OpenCV, the `v4l2_camera` ROS package). The Pi's ribbon-cable (CSI) camera works through libcamera, but takes more setup on Ubuntu than on Raspberry Pi OS.

---

## 3. Write the card with Raspberry Pi Imager

These steps are for Imager 1.8.5 (the Ubuntu package). Imager 2.x shows the same fields as wizard steps.

1. **(Optional, a good idea on a Pi 4) Update the bootloader first.** Misc utility images → Bootloader (Pi 4 or Pi 5 family) → SD Card Boot. Write it to a spare card and boot the Pi from it once. The green LED blinks steadily when it's done.
2. **Choose device:** Raspberry Pi 5 (or 4).
3. **Choose OS:** either route gives you the same image.
   - **From Imager's list:** Other general-purpose OS → Ubuntu → **Ubuntu Server 26.04.1 LTS (64-bit)**. Pick Server, not Desktop. Imager downloads the file again, but it knows it's an Ubuntu image and reliably applies your settings.
   - **The file you downloaded:** scroll to **Use custom** and pick `ubuntu-26.04.1-preinstalled-server-arm64+raspi.img.xz`. Imager reads `.xz` files directly, so don't unpack it. With this route, the step 7 check is required.
4. **Choose storage:** pick the microSD card, or the SSD in a USB adapter, and confirm it by its size. Leave **"Exclude system drives"** ticked; that's what protects the PC's dual-boot disks.
5. **Next → Edit settings:**
   - **General**
     - Hostname: e.g. `rpi-ros` (used in the examples below).
     - Username (lowercase) and a strong password.
     - Wireless LAN: network name (SSID), password and **country code**. A wrong country code can hide 5 GHz networks.
     - Locale: time zone and keyboard layout.
   - **Services:** enable SSH with **password authentication**. It stays on, because this Pi is for public use and its password is published in the README. SSH keys are optional (section 7).
   - **Options:** eject when finished; telemetry as you prefer.
   - **Raspberry Pi Connect** (Imager 2.x only): leave it off; it only works with Raspberry Pi OS.
6. **Save → Yes →** confirm the erase, then let Imager write and verify.
7. **Check that the settings were applied** (required with "Use custom", optional otherwise): put the card back in the PC, open the boot partition, and look at `user-data` and `network-config`. Your username and Wi-Fi name should be there. If they're missing, write the card again, this time picking the OS from Imager's list. `network-config` contains only your Wi-Fi. That's normal for Imager, and it's why the cable needs one extra file after the first login (section 5).

Ubuntu applies these settings with **cloud-init** (its first-boot setup tool).

---

## 4. First boot

- Give it **3–5 minutes**. cloud-init creates your user and joins Wi-Fi, and logging in too early can fail.
- **If the Pi shows up on neither Wi-Fi nor the cable**, see [First-boot problems](#first-boot-problems) below. With Imager 1.8.5 and Ubuntu 26.04.1, expect to fix the network once at the first login.
- Since Ubuntu 25.10, boot files live in `current/`, `new/` and `old/` folders, and the Pi **test-boots after each kernel update**. An extra automatic reboot is normal; don't cut the power. (One open bug report says a Pi 4 kept failing this test boot, which is another small reason to prefer a Pi 5.)

First commands after logging in:

```bash
grep PRETTY_NAME /etc/os-release          # confirms Ubuntu 26.04.1
cloud-init status --long                  # first-boot setup result (see the table below if it says error)
networkctl list                           # eth0 and wlan0 should both say "routable  configured"
sudo apt update && sudo apt full-upgrade  # may wait for automatic updates first; let it wait
```

### First-boot problems

Seen on 2026-11-04/05 with a Pi 5, Imager 1.8.5 and Ubuntu Server 26.04.1:

| What you see | Why | Fix |
|---|---|---|
| The Pi is on neither Wi-Fi nor the cable, and `networkctl list` shows every link as `unmanaged` | The network service (`systemd-networkd`) ran only during the very early boot stage and wasn't started in the real system on the first boot | `sudo netplan apply` restarts it. Then `sudo systemctl enable systemd-networkd` so it starts at every boot. Confirmed working after a reboot. |
| The cable never gets an address, and `sudo netplan get ethernets` prints `null` | Imager's Wi-Fi settings replace Ubuntu's network file, which drops the cable settings | Create `/etc/netplan/90-direct.yaml` (section 5, step 1) |
| The login banner says `IPv4 address for eth0: 10.10.10.2` | A snapshot taken when Ubuntu built the image, not a real address | Ignore it; `ip -br addr` shows the real addresses |
| `cloud-init status` says `error` about `avahi-daemon` | Imager asks for it to be installed on first boot, which fails while the Pi is offline | `sudo apt install avahi-daemon` once the Pi is online |
| `Could not get lock … held by process … (unattended-upgr)` | Automatic security updates start as soon as the Pi is online | Let `apt` wait (10–30 minutes). Pressing `Ctrl+C` on your own waiting command is safe, but never kill the updater, delete lock files or power off. |
| The clock shows a date in the past, or `apt update` says "not valid yet" | The Pi has no clock battery by default and no internet yet | Fixes itself once online (`timedatectl` shows `synchronized: yes`) |
| The Pi runs Ubuntu 24.04 instead of 26.04 | The wrong image was written (this happened on the first attempt), or the Pi starts from another drive | Rewrite the card, checking the OS name on Imager's main screen; unplug other drives |

**Getting in when the network is down.** Connect the cable to the PC and use the Pi's automatic link-local address, which works even before anything is configured:

```bash
ping -6 -c 3 ff02::1%enp4s0             # on the PC: every device on the cable answers, the PC included
ssh <user>@fe80::<pi-address>%enp4s0
```

In the `ping` output, the address that isn't the PC's own (compare with `ip -br addr show enp4s0`) is the Pi. Log in, then apply the first two fixes above.

---

## 5. Network: Wi-Fi or Ethernet cable

### Over Wi-Fi

The PC and the Pi must be on the same network. The PC's current Wi-Fi uses a regular password (WPA-PSK), which Imager can set up.

- **Finding the Pi:** check the router's list of connected devices, or `ping rpi-ros.local`. The `.local` name only works once `avahi-daemon` runs on the Pi. Imager asks for it to be installed on first boot, but that fails if the Pi isn't online yet (section 4), so install it yourself with `sudo apt install avahi-daemon`. The Ubuntu PC already runs it, and Windows 10/11 can look up `.local` names too.
- **Keeping its address:** reserve a fixed address for the Pi in the router.
- **Campus or enterprise Wi-Fi** (eduroam and other networks with a username and password) can't be set up in Imager, and these networks often block devices from reaching each other. Use a phone hotspot, a small travel router or the cable instead.

### Ethernet cable

Ubuntu's image sets the Pi's Ethernet port (`eth0`) to ask for an address automatically. **When Imager adds Wi-Fi settings, though, it replaces that file and the cable loses its settings**, so after the first login, give `eth0` its settings back with one small file (step 1 below). From then on the cable works instead of Wi-Fi or alongside it. With both connected, the Pi simply has two addresses. Any normal Ethernet cable works; you don't need a crossover cable.

```
Option 1 — Pi plugged into the router

   [PC] ~~~ Wi-Fi ~~~ [Router 192.168.0.1] ===== cable ===== [Pi]
                                                         192.168.0.x   ← the router picks it

Option 2 — cable straight from the Pi to any PC (no Wi-Fi or router needed)

   [Any PC] ========= cable ========= [Pi]
  192.168.77.1                     192.168.77.2   ← fixed, always the same
```

**When the cable is worth it:**

- **If Wi-Fi didn't connect on the first boot:** the cable needs no Wi-Fi settings.
- **For a faster, steadier link:** both Pi 4 and Pi 5 have gigabit Ethernet, which helps with camera streams, ROS bags and large files.
- **Where there's no Wi-Fi**, or the network blocks devices from seeing each other (common on campus Wi-Fi), a direct cable avoids the problem.

#### Step 1: cable settings on the Pi (once)

Connect over Wi-Fi, or over the link-local address (section 4), so changing the cable settings can't cut your session. Then:

```bash
sudo tee /etc/netplan/90-direct.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true                  # still accepts an address from a router or a sharing PC
      optional: true               # boot doesn't wait when no cable is plugged in
      addresses: [192.168.77.2/24] # fixed address, always present
EOF
sudo chmod 600 /etc/netplan/90-direct.yaml
sudo netplan apply
ip -br addr show eth0              # should include 192.168.77.2/24
```

`192.168.77.x` was picked because it's rarely used. `192.168.50.x`, for example, is the default network of ASUS routers and could clash.

#### Option 1: Pi to router

1. Plug the Pi into any free LAN port on the router.
2. Find `rpi-ros` in the router's device list (for example at `http://192.168.0.1`) and connect with `ssh <user>@192.168.0.x`.

From the PC's point of view this works exactly like Wi-Fi.

#### Option 2: Pi directly to any PC

A cable between two computers has no router, so each PC's Ethernet port needs an address in the same range as the Pi: `192.168.77.1`, mask `255.255.255.0`, **no gateway**. Set this up once per PC (it needs admin rights on that PC). Then connect from any of them with:

```bash
ssh <user>@192.168.77.2
```

**Ubuntu PC that also shares its internet with the Pi** (the setup on the main PC). This uses a separate NetworkManager profile, so the normal wired settings stay untouched:

```bash
nmcli connection add type ethernet ifname enp4s0 con-name pi-shared ipv4.method shared ipv4.addresses 192.168.77.1/24
nmcli connection up pi-shared
```

- **What it does:** the PC becomes `192.168.77.1`, gives the Pi an extra address (`192.168.77.10–254`) and shares its Wi-Fi internet, so `apt` works on the Pi without Wi-Fi.
- **Run the `add` command only once.** Running it twice creates two profiles with the same name, and the wrong one can win after a restart. Check with `nmcli -f NAME,UUID connection show | grep pi-shared`, and delete an extra one with `nmcli connection delete <UUID>`.
- **Port name:** replace `enp4s0` with the PC's Ethernet port name (`ip -br link`).
- **Switching back:** to use that Ethernet port with a normal router again, run `nmcli connection down pi-shared`.

**Any other PC.** This gives a fixed address only; the Pi gets no internet through it:

| PC | How |
|---|---|
| Ubuntu/Linux | Settings → Network → Wired → gear icon → IPv4 → **Manual** → Address `192.168.77.1`, Netmask `255.255.255.0`, Gateway empty → Apply |
| Windows | `Win+R` → `ncpa.cpl` → right-click **Ethernet** → Properties → **Internet Protocol Version 4** → "Use the following IP address": `192.168.77.1`, `255.255.255.0`, gateway empty → OK |
| macOS | System Settings → Network → Ethernet → Details → TCP/IP → Configure IPv4: **Manually** → `192.168.77.1`, `255.255.255.0` |

Laptops without an Ethernet port need a USB-to-Ethernet adapter. Afterwards, switch that port back to **Automatic (DHCP)** to use it on normal wired networks.

**Windows with internet sharing (optional).** Internet Connection Sharing (Wi-Fi adapter → Properties → Sharing → allow it, then choose Ethernet) forces the Ethernet port to `192.168.137.1`, so `192.168.77.2` isn't reachable while it's on. The Pi then gets a `192.168.137.x` address, which you can find with `arp -a`. Use this only when the Pi needs internet through Windows.

#### No setup at all: link-local address

Every Ethernet port gives itself an automatic link-local address (`fe80::…`) as soon as a cable is connected. It needs no settings or admin rights, so it works on any PC, including ones you can't change:

```bash
ssh <user>@fe80::2ecf:67ff:fe6c:4e6e%<port>     # this Pi's link-local address
```

- **The Pi's part:** shown by `ip -br addr show eth0` on the Pi. It comes from the hardware ID, so it never changes. For this Pi it's `fe80::2ecf:67ff:fe6c:4e6e`.
- **The PC's part (`%<port>`):**
  - **Linux:** the port name from `ip -br link`, for example `%enp4s0`.
  - **Windows:** run `ipconfig`. Under the Ethernet adapter, find "Link-local IPv6 Address … `fe80::…%12`" and use that number, for example `%12`.
  - **macOS:** the port name from `ifconfig`, for example `%en5`.

#### Summary

| Situation | PC setup | Command |
|---|---|---|
| Home Wi-Fi | none | `ssh <user>@rpi-ros.local` or the router-given address |
| Cable to the main Ubuntu PC | `pi-shared` profile (once) | `ssh <user>@192.168.77.2` |
| Cable to any other PC | manual `192.168.77.1/24` (once) | `ssh <user>@192.168.77.2` |
| Cable to a PC you can't change | none | `ssh <user>@fe80::…%<port>` |

---

## 6. Terminal vs graphical access

| | What you get | Cost on the Pi | Good for |
|---|---|---|---|
| **Terminal (SSH)** | A command line on the Pi | Almost nothing | apt, `colcon build`, `ros2 launch`, logs, editing code. Nearly everything. |
| **ROS tools on the PC** | RViz2, rqt and image viewers on the PC, fed by the Pi over the ROS 2 network | Only the data being sent | Visualizing robot data. **Usually the best option.** |
| **Graphical desktop** | The Pi's whole desktop streamed to the PC | A desktop environment (Ubuntu Server has none), plus RAM, CPU and disk | Occasional GUI-only tasks |

**Recommendation:** run the Pi without a desktop. Use SSH and VS Code for work, and the ROS tools on the PC for visualization. Add a desktop and NoMachine only if you really need them.

---

## 7. Remote access from Ubuntu and Windows

### Terminal tools

| Task | Ubuntu | Windows |
|---|---|---|
| Command line | `ssh <user>@rpi-ros.local` | Same command in PowerShell or Windows Terminal (SSH is built in) |
| Edit code on the Pi | VS Code + Remote-SSH | VS Code + Remote-SSH |
| Copy files | `scp`, `rsync`, or `sftp://` in the Files app | `scp`, WinSCP |

### Password logins stay on

This Pi is for **public use**. Its login (`eljohopi` / `eljohopi`) is published in the [README](README.md), so **password logins stay on**: anyone with a cable to the Pi, or on the same network, can log in with the password.

- **Full control:** `sudo` uses the same password, so anyone who can reach the Pi has full control of it. Keep it on a direct cable or a private network, and **never make its SSH reachable from the internet**.
- **Don't turn password logins off:** don't set `PasswordAuthentication no`, and don't add a key-only SSH setting, or the published password stops working for SSH.
- **If the Pi stops being for public use:** change its password with `passwd` and remove it from the README.

### SSH keys (optional, one per PC)

Keys are only a convenience for your own PCs, which then log in without typing the password. Password logins keep working for everyone else at the same time.

```bash
# On each system (Ubuntu terminal and Windows PowerShell):
ssh-keygen -t ed25519

# From Ubuntu:
ssh-copy-id <user>@rpi-ros.local

# From Windows PowerShell (Windows has no ssh-copy-id):
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <user>@rpi-ros.local "cat >> ~/.ssh/authorized_keys"
```

Test with a new login: it shouldn't ask for the password any more. The password still works from PCs without a key.

A short name saves typing, and VS Code lists it too. Add this to `~/.ssh/config` on Ubuntu, or `%USERPROFILE%\.ssh\config` on Windows:

```
Host rpi
    HostName rpi-ros.local
    User <user>

Host rpi-cable
    HostName 192.168.77.2
    User <user>
```

Then `ssh rpi` connects over Wi-Fi, and `ssh rpi-cable` over the direct cable.

After a reflash the Pi gets a new identity (host key), so remove the old one with `ssh-keygen -R rpi-ros.local` on each system.

### Graphical options, best first

1. **ROS tools on the PC (recommended).** The Pi runs your nodes without a desktop, and the PC runs `rviz2`, `rqt` and `rqt_image_view` (already installed with `ros-lyrical-desktop`).
   - Keep `ROS_DOMAIN_ID` the same on both machines (the default is 0).
   - Instead of `cv2.imshow` on the Pi, publish images as ROS topics, compressed if you're on Wi-Fi.
   - Quick connection test:

     ```bash
     # Pi
     ros2 topic pub /chatter std_msgs/msg/String "{data: hello}"
     # PC
     ros2 topic echo /chatter
     ```

     If nothing arrives, check that both machines use the same `ROS_DOMAIN_ID` and network, that `ROS_AUTOMATIC_DISCOVERY_RANGE` isn't set to `LOCALHOST`, and that no firewall is in the way.
   - **On Windows:** the simplest option is to boot into Ubuntu for visualization. Alternatives:
     - Run `foxglove_bridge` on the Pi and use the Foxglove or Lichtblick app on Windows.
     - Use ROS 2 under WSL2. It needs WSL's "mirrored" networking mode for ROS 2 machines to find each other.
2. **Single windows with `ssh -X` (X11 forwarding).** Fine for rqt or a small OpenCV debug window, but too slow for RViz or video. The Pi needs `xauth` installed. The Ubuntu PC handles these windows as is; Windows needs an X server program (MobaXterm has one built in).
3. **A full remote desktop:** NoMachine (next section), or xrdp, which works with Windows' built-in Remote Desktop and with Remmina on Ubuntu.

---

## 8. NoMachine on Ubuntu Server

NoMachine does work, but it needs a few extra pieces:

- **A desktop environment.** NoMachine's guide says one "must already be installed. This applies also to headless Linux machines." Use **XFCE**, which is light and works with NoMachine's virtual display. Avoid the full Ubuntu desktop: it's heavy, and GNOME on 26.04 is Wayland-only, which doesn't fit NoMachine's monitorless mode.
- **The ARM package.** Get "NoMachine for ARM", ARMv8 `.deb`, from nomachine.com. The guide only names the Pi 2, 3 and 4. The package isn't tied to a specific Pi, so it should run on a Pi 5, but test it.
- **A display for a Pi with no monitor.** If no display is running, NoMachine starts its own. The client shows "Cannot detect any display running" and offers to create one. The simplest setup is to skip a login screen manager (lightdm or gdm3) on the Pi and let NoMachine create the display.
- **Port and firewall.** NoMachine uses TCP port **4000**. Ubuntu Server's firewall (ufw) is off by default; if you turn it on, allow port 4000 (`sudo ufw allow 4000/tcp`). Don't open it to the internet.
- **Clients and licence.** There are clients for both Ubuntu and Windows, so it works the same from either boot. The free edition is for personal use, one connection at a time.
- **CPU cost.** The Pi 5 has no hardware H.264 video encoder, so the CPU has to encode the desktop stream, and that competes with YOLO and SLAM. Close the NoMachine session during heavy runs.

Sketch of the later install (only if you decide you need a desktop):

```bash
sudo apt install --no-install-recommends xfce4 xfce4-terminal dbus-x11
# check that no login manager (lightdm / gdm3) got pulled in
sudo apt install ./nomachine_<version>_arm64.deb    # downloaded from nomachine.com
```

---

## 9. Step-by-step checklist

**Prepare**

- [x] Download `ubuntu-26.04.1-preinstalled-server-arm64+raspi.img.xz` and verify its checksum (done 2026-11-04).
- [x] Raspberry Pi 5 and a 64 GB microSD card.
- [ ] Check that you have the 27 W USB-C power supply and an Active Cooler (important for long YOLO runs).
- [ ] Move the `.img.xz` out of the repo, or git-ignore disk images.
- [ ] *(Optional)* Update the bootloader with Imager → Misc utility images → Bootloader.

**Write the card (section 3)**

- [x] In Imager, select **Ubuntu Server 26.04.1 LTS (64-bit)**. This worked on the second attempt; the first card turned out to be 24.04.
- [x] Set the hostname, user, Wi-Fi with country code, time zone, and SSH with password authentication.

**First boot (sections 4–5)**

- [x] Boot with an Ethernet cable to the PC as the backup, and log in. The first login was over the link-local address.
- [x] Fix the network: `90-direct.yaml`, `sudo netplan apply`, then `sudo systemctl enable systemd-networkd`.
- [x] Run `sudo apt update && sudo apt full-upgrade`, then reboot and check `networkctl list` (passed 2026-11-05).
- [x] Fixed cable address `192.168.77.2` on the Pi and the `pi-shared` profile (`192.168.77.1`) on the Ubuntu PC, tested with `ssh <user>@192.168.77.2`.
- [ ] Install `avahi-daemon` so `rpi-ros.local` works.
- [ ] Set `192.168.77.1` on the Windows side of the dual boot, and on any other PC you'll use.
- [ ] Reserve the Pi's Wi-Fi address in the router.

**Remote access (section 7)**

- [x] Password logins stay on: the Pi is for public use and its login is published in the README (decided 2026-11-05).
- [ ] *(Optional)* Create SSH keys on your own PCs (Ubuntu and Windows) for password-free logins.
- [ ] Add the `Host rpi` entry to `~/.ssh/config` on both systems.
- [ ] Set up VS Code Remote-SSH on both systems.

**Software**

- [ ] Install ROS 2 Lyrical on the Pi (`ros-lyrical-ros-base`, following the official Ubuntu deb guide).
- [ ] Test the ROS link: `ros2 topic pub` on the Pi, `ros2 topic echo` on the PC, then RViz2 on the PC.
- [ ] OpenCV from apt (`python3-opencv`, `libopencv-dev`).
- [ ] YOLO in `~/venvs/yolo` (`--system-site-packages`), with a nano model exported to NCNN or ONNX.
- [ ] SLAM (`slam_toolbox` for 2D LiDAR).
- [ ] *(Optional)* XFCE + NoMachine, only if a Pi desktop is really needed.

---

## 10. Sources

- [Ubuntu 26.04.1 release images](https://cdimage.ubuntu.com/releases/26.04.1/release/)
- [ROS 2 Lyrical Luth release notes](https://docs.ros.org/en/lyrical/Releases/Release-Lyrical-Luth.html)
- [NoMachine Free Edition installation and configuration guide](https://kb.nomachine.com/?p=10744)
- [A/B boot on Ubuntu for Raspberry Pi](https://ubuntu.com/hardware/docs/boards/explanations/piboot-ab/)
- [Pi 4 test-boot bug report (LP #2142169)](https://lists.snapcraft.io/archives/foundations-bugs/2026-February/545811.html)
- [Raspberry Pi Imager 2.0 announcement](https://www.raspberrypi.com/news/a-new-raspberry-pi-imager/)
