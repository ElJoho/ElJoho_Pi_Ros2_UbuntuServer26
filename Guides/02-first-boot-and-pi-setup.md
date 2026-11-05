# Guide 2 — First boot and setting up the Pi

This guide takes the Pi from its first power-on to a fully updated Ubuntu 26.04 server. At the end it's reachable over **Wi-Fi** and over a **direct Ethernet cable** at the fixed address `192.168.77.2`, and its network starts by itself at every boot.

It follows what was actually done on this project's Pi on 2026-11-04/05, including the problems that came up and how they were fixed.

> Before starting: [Guide 1](01-write-ubuntu-server-with-pi-imager.md) (card written) and [Guide 3](03-ubuntu-pc-ethernet-connection.md), Parts 1–4 (PC's cable connection ready).
> Time needed: about 45–60 minutes, mostly waiting for updates.

---

## Values used in this guide

| What | Value for this Pi |
|---|---|
| Board | Raspberry Pi 5 |
| System | Ubuntu Server 26.04.1 LTS (kernel `7.0.0-…-raspi`) |
| Hostname and user | `eljohopi` |
| Password | `eljohopi` (published on purpose: public-use Pi; also used for `sudo`) |
| Wi-Fi address | `192.168.0.16`, handed out by the home router (can change unless reserved) |
| Cable address (fixed) | `192.168.77.2` |
| PC cable address | `192.168.77.1`, from the `pi-shared` profile ([Guide 3](03-ubuntu-pc-ethernet-connection.md)) |
| PC Ethernet port | `enp4s0` (yours may differ: `nmcli device status`) |
| Pi link-local address (cable) | `fe80::2ecf:67ff:fe6c:4e6e` |
| Hardware IDs | Cable `2c:cf:67:6c:4e:6e`, Wi-Fi `2c:cf:67:6c:4e:70` |
| SSH fingerprint | `SHA256:Wfg76GKg1mLrQLbDDW9ElpJQCNAgNSca4a+DSPwa5Po` |

### Where to type each command

The prompt shows which computer a command runs on:

| Prompt | Computer | Used for |
|---|---|---|
| `johotan@johotan:~$` (your PC's user and name) | **PC** | `nmcli`, `ping`, `ssh`, anything about `enp4s0` |
| `eljohopi@eljohopi:~$` | **Pi** (inside SSH) | `netplan`, `apt`, `systemctl`, anything about `eth0`/`wlan0` |

`sudo` asks for **the Pi's password** (`eljohopi`). Nothing appears while you type it, which is normal. To leave the Pi and return to the PC, type `exit`.

---

## Part 1 — Power on and wait

1. **Put the card in the Pi.**
2. **Connect the Ethernet cable** from the Pi to the PC. The PC's `pi-shared` profile must be on ([Guide 3](03-ubuntu-pc-ethernet-connection.md), Part 4). The cable is your way in if Wi-Fi doesn't come up on the first boot, which is what happened here.
3. **Connect the power supply** (27 W USB-C for a Pi 5).
4. **Watch the LED:**
   - **Green, flickering:** running and reading the card. ✅
   - **Red only:** powered but not running. On a Pi 5, press the power button once.
   - **Repeating blink pattern:** a boot problem, such as a bad card or bad image. Count the long and short blinks.
5. **Wait 3–5 minutes.** On the first boot, cloud-init (Ubuntu's first-boot setup tool) creates your user, sets the hostname and applies the Wi-Fi settings. The Pi may restart once by itself. **Don't unplug it during this time.**

---

## Part 2 — Find the Pi

### Over Wi-Fi

- **Router page:** open `http://192.168.0.1` and look for `eljohopi` in the list of connected devices.
- **Search from the PC** (replace `192.168.0` with the start of your network's addresses):

  ```bash
  for i in $(seq 1 254); do ping -c1 -W1 192.168.0.$i >/dev/null 2>&1 & done; wait
  ip neigh | grep -iE '2c:cf:67|d8:3a:dd|dc:a6:32|e4:5f:01|b8:27:eb|28:cd:c1|88:a2:9e'
  ```

  The second line keeps only devices with Raspberry Pi hardware-ID prefixes.

> `eljohopi.local` does **not** work yet on the first boot. The service behind it (`avahi-daemon`) gets installed in Part 7.

### Over the cable (always works)

Every Ethernet port gives itself an automatic **link-local** address (`fe80::…`) as soon as a cable is connected, even when nothing is configured. From the PC:

```bash
ping -6 -c 3 ff02::1%enp4s0
```

Every device on the cable answers, **including the PC itself**. Compare with `ip -br addr show enp4s0`: the address that isn't the PC's own is the Pi. For this Pi it's `fe80::2ecf:67ff:fe6c:4e6e`. It's built from the hardware ID, so it never changes.

> **What happened here:** on the first boot, the Pi joined neither Wi-Fi nor the cable network (Part 4 explains why). The first login was therefore over the link-local address.

---

## Part 3 — First login

From the PC, use whichever address you found:

```bash
ssh eljohopi@fe80::2ecf:67ff:fe6c:4e6e%enp4s0     # over the cable; the %enp4s0 part says which PC port to use
ssh eljohopi@192.168.0.16                         # or over Wi-Fi, if the Pi showed up there
```

This is what you'll see:

```
The authenticity of host '…' can't be established.
ED25519 key fingerprint is: SHA256:Wfg76GKg1mLrQLbDDW9ElpJQCNAgNSca4a+DSPwa5Po
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes      ← type yes
eljohopi@…'s password:                                                         ← nothing shows while typing
Welcome to Ubuntu 26.04.1 LTS (GNU/Linux 7.0.0-1017-raspi aarch64)
 …
  System information as of Wed Aug 26 03:22:33 UTC 2026
  … IPv4 address for eth0: 10.10.10.2
 …
eljohopi@eljohopi:~$                                                           ← you're on the Pi now
```

- **The "authenticity" question** appears once per address. SSH remembers the Pi's identity in `~/.ssh/known_hosts`. On a new card, the fingerprint will differ from the one above.
- **"Permission denied":** cloud-init may still be creating your user. Wait a minute and try again.
- **⚠️ The "System information" block is fake on the first boot.** The date (Aug 26) and `10.10.10.2` are a snapshot from when Ubuntu built the image. Ignore them.

Confirm the system is the right one:

```bash
grep PRETTY_NAME /etc/os-release      # must say: Ubuntu 26.04.1 LTS
```

If it says 24.04, the wrong image was written. Go back to [Guide 1](01-write-ubuntu-server-with-pi-imager.md).

---

## Part 4 — Check what the first boot did

Run these on the Pi and compare with what this Pi showed:

```bash
cloud-init status --long
networkctl list
ip -br addr
sudo netplan get ethernets
ls -l /etc/netplan/
timedatectl
```

| Command | What this Pi showed | What it means |
|---|---|---|
| `cloud-init status --long` | `status: error`, failed to install `avahi-daemon` | First-boot setup **finished**. Imager asks it to install `avahi-daemon`, which needs internet the Pi didn't have yet. Harmless; fixed in Part 7. |
| `networkctl list` | `eth0` and `wlan0` both `unmanaged`, OPERATIONAL `-` | **The network service (`systemd-networkd`) isn't running.** It ran only during the very early boot stage and wasn't started in the real system on the first boot. That's why Wi-Fi was down even though its settings were correct. |
| `ip -br addr` | `eth0` only has `fe80::…`, and `wlan0` is `DOWN` | No real addresses, so no internet |
| `sudo netplan get ethernets` | `null` | **The cable has no settings at all.** Imager replaced Ubuntu's network file with one that holds only Wi-Fi. |
| `ls -l /etc/netplan/` | only `50-cloud-init.yaml` | That's the Wi-Fi-only file cloud-init created from Imager's `network-config` |
| `timedatectl` | a date in the past, `System clock synchronized: no` | No internet, so the clock can't be set. It fixes itself once online. |

To look at the Wi-Fi settings **with the password hidden**:

```bash
sudo sed -E 's/(password).*/\1: <hidden>/I' /etc/netplan/50-cloud-init.yaml
```

This Pi showed:

```yaml
network:
  version: 2
  wifis:
    wlan0:
      optional: true
      dhcp4: true
      access-points:
        "<your Wi-Fi name>":
          auth:
            key-management: "psk"
            password: <hidden>
```

> Don't use `cat` on the netplan files, and don't run `sudo netplan get` without a section name. Both print your Wi-Fi password.

If `networkctl list` already shows `routable configured` for both links on your Pi, the service started fine. Still do Part 5.1, so the cable gets its settings.

---

## Part 5 — Fix the network

### 5.1 Give the cable its settings

This file tells `eth0` to (1) accept an address from a router or a sharing PC, and (2) **always** have the fixed address `192.168.77.2`. Paste the whole block at once:

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
```

| Line | What it does |
|---|---|
| `sudo tee … <<'EOF' … EOF` | Writes everything between the `EOF` lines into `/etc/netplan/90-direct.yaml`. Netplan is Ubuntu's network configuration tool. |
| `sudo chmod 600 …` | Makes the file readable only by root. Netplan warns about anything else. |
| `sudo netplan apply` | Applies all network settings now |

**Expected message** on the first run:

```
Failed to connect to network service /run/systemd/netif/io.systemd.Network: Connection refused
Falling back to a hard restart of systemd-networkd.service
```

That's the fix happening: the network service wasn't running, and netplan started it. Your SSH session may freeze for a few seconds. If it hasn't come back after 30 seconds, close the terminal and reconnect with the same `ssh` command.

> `192.168.77.x` was chosen because it's rarely used. `192.168.50.x`, for example, is the default network of ASUS routers and could clash if the Pi is ever plugged into one.

### 5.2 Make the network service start at every boot

```bash
sudo systemctl enable systemd-networkd
systemctl is-enabled systemd-networkd      # must say: enabled
```

Before this, it said `enabled-runtime`, meaning on for the current boot only. Most likely, on the first boot Ubuntu decided whether to start the network service before cloud-init had written the network files.

### 5.3 Check

On the Pi:

```bash
networkctl list            # eth0 and wlan0: routable  configured
ip -br addr                # eth0: 192.168.77.2/24 (+ 192.168.77.x from the PC); wlan0: 192.168.0.16/24
ping -c 3 ubuntu.com       # replies = the Pi is online
timedatectl | grep -E 'Local time|synchronized'     # correct time, synchronized: yes
```

On the PC (in a second terminal):

```bash
ping -c 3 192.168.77.2
ssh eljohopi@192.168.77.2      # SSH asks the first-time question again for this address: same fingerprint, answer yes
```

---

## Part 6 — Update the system

On the Pi:

```bash
sudo apt update
sudo apt full-upgrade -y
```

- **"Waiting for cache lock: … held by process … (unattended-upgr)"** is expected. Ubuntu's automatic security updater starts as soon as the Pi gets internet for the first time. `apt` waits and continues on its own, which can take **10–30 minutes**.
  - Pressing `Ctrl+C` on **your** waiting command is safe; nothing has changed yet.
  - **Never** kill the updater, delete lock files or unplug the Pi while it runs. That can damage the package system.
  - To watch what it's doing (`Ctrl+C` only stops the watching):

    ```bash
    tail -f /var/log/unattended-upgrades/unattended-upgrades-dpkg.log
    ```

- **"Release file … is not valid yet"** means the clock hasn't synced. Wait a minute and run `sudo apt update` again.
- **It's finished** when your prompt comes back with no errors above it.

On this Pi, 161 packages were updated, including the kernel (`7.0.0-1017` → `7.0.0-1020`).

---

## Part 7 — Install `avahi-daemon` (the `eljohopi.local` name)

```bash
sudo apt install -y avahi-daemon
systemctl is-active avahi-daemon      # must say: active
```

Then, from the PC:

```bash
ping -c 2 eljohopi.local
ssh eljohopi@eljohopi.local
```

This works from Linux, Windows 10/11 and macOS on the same network, whatever address the router gave the Pi.

---

## Part 8 — Reboot test

This proves that everything comes back by itself:

```bash
sudo reboot
```

1. Your SSH session closes; that's normal.
2. Wait **2–4 minutes**. After a kernel update, the Pi restarts **twice**: Ubuntu test-boots the new kernel before keeping it. Don't unplug it.
3. Reconnect from the PC:

   ```bash
   ssh eljohopi@192.168.0.16      # or eljohopi.local, or 192.168.77.2 over the cable
   ```

4. Check:

   ```bash
   networkctl list      # eth0 and wlan0: routable  configured
   ```

The welcome screen now shows real information: the correct date, and the Pi's real address.

✅ **Passed on this Pi on 2026-11-05** (kernel `7.0.0-1020-raspi`, "0 updates can be applied").

**If the Pi doesn't come back within 5 minutes,** log in over the cable's link-local address and restart the network service:

```bash
ssh eljohopi@fe80::2ecf:67ff:fe6c:4e6e%enp4s0
sudo systemctl restart systemd-networkd
```

---

## Part 9 — Shutting down and starting safely

| Action | How |
|---|---|
| Turn off | `sudo poweroff`. Wait until the LED is **red** (Pi 5) or stops flickering (about 10–20 s), **then** unplug the power. |
| Restart | `sudo reboot` |
| Turn on | Plug in the power. On a Pi 5, if the LED stays red, press the power button. |
| **Never** | Pull the power while the Pi is running or updating. That can corrupt the SD card. |

---

## Part 10 — Recommended next steps

- **Reserve the Wi-Fi address** in the router (`http://192.168.0.1` → "DHCP reservation", "Static lease" or similar). Use the Wi-Fi hardware ID `2c:cf:67:6c:4e:70` so `192.168.0.16` never changes.
- **Set up the cable on other PCs:** Ubuntu → [Guide 3](03-ubuntu-pc-ethernet-connection.md). Windows/macOS → [PLAN.md](../PLAN.md) section 5.
- **Password logins stay on.** This Pi is for public use and its login is published in the README. SSH keys are optional, for password-free logins from your own PCs ([PLAN.md](../PLAN.md) section 7).
- **Check the Wi-Fi country.** Imager's network file has no `regulatory-domain` line. To check:

  ```bash
  sudo apt install -y iw
  iw reg get | head -3      # should say: country CO
  ```

  If it doesn't say `CO`, add the country to the Wi-Fi settings in a separate file, and test it while connected **over the cable** (if Wi-Fi breaks, `netplan try` reverts after 2 minutes):

  ```bash
  sudo tee /etc/netplan/91-wifi-country.yaml > /dev/null <<'EOF'
  network:
    version: 2
    wifis:
      wlan0:
        regulatory-domain: "CO"
  EOF
  sudo chmod 600 /etc/netplan/91-wifi-country.yaml
  sudo netplan try        # press Enter to keep the change if Wi-Fi still works
  ```

- **Install ROS 2 Lyrical** (`ros-lyrical-ros-base`) and test it with the PC ([PLAN.md](../PLAN.md) section 7 and the checklist).

---

## Troubleshooting

| What you see | Why | Fix |
|---|---|---|
| The Pi is on neither Wi-Fi nor the cable after 5 minutes | The network service didn't start on the first boot | Log in over link-local (Part 2), then Part 5 |
| `networkctl list` shows every link `unmanaged` | Same | `sudo netplan apply`, then `sudo systemctl enable systemd-networkd` |
| The cable never gets an address; `sudo netplan get ethernets` prints `null` | Imager's network file has only Wi-Fi | Part 5.1 |
| Banner says `IPv4 address for eth0: 10.10.10.2` | A snapshot from when the image was built | Ignore it; use `ip -br addr` |
| `cloud-init status` says `error` (`avahi-daemon`) | No internet on the first boot | Part 7 |
| `Could not get lock … (unattended-upgr)` | Automatic updates running | Wait; never kill it |
| Clock in the past, "not valid yet" | Not online yet | Fixes itself once online |
| `ssh: Could not resolve hostname eljohopi.local` | `avahi-daemon` isn't installed or running on the Pi | Part 7; use the IP address meanwhile |
| `Connection refused` | Reachable, but SSH isn't running (yet) | Wait 2 minutes. If it persists, rewrite the card with SSH enabled. |
| `Permission denied` | Wrong username or password, or cloud-init still running | Username and password are both `eljohopi` (lowercase); wait a minute |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | The card was rewritten (new identity), or another device now uses that address | If expected: `ssh-keygen -R <address>` on the PC |
| `grep PRETTY_NAME /etc/os-release` says 24.04 | Wrong image, or the Pi starts from another drive | Rewrite ([Guide 1](01-write-ubuntu-server-with-pi-imager.md)); unplug SSD/USB drives |

---

## What happened on this Pi (2026-11-04/05)

1. The first card turned out to run **Ubuntu 24.04**: its SSH reported `OpenSSH_9.6p1`, while 26.04 has `10.2p1`. It was rewritten with 26.04.1.
2. On the first boot of 26.04.1, the Pi stayed off both Wi-Fi and the cable network. The first login was over the cable's link-local address.
3. The diagnosis: the network service wasn't running, the cable had no settings, and the `10.10.10.2` banner was fake.
4. The fix: `90-direct.yaml`, then `netplan apply`, which started the network service, then `systemctl enable systemd-networkd`. Wi-Fi (`192.168.0.16`) and the cable came up, and the clock synced.
5. `apt full-upgrade` waited for `unattended-upgrades`, then upgraded 161 packages, including a new kernel.
6. The reboot test passed: both links came back by themselves.
7. The fixed cable address `192.168.77.2` was added and tested from the PC with `ssh eljohopi@192.168.77.2`.
