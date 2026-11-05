# Guide 1 — Write Ubuntu Server 26.04 LTS to the Pi's card with Raspberry Pi Imager

This guide takes you from an empty microSD card to a card that's ready for the Pi's first boot. When you're done, it holds Ubuntu Server 26.04.1 LTS with your username, password, Wi-Fi and SSH already set up.

> Tested on 2026-11-04/05 with Raspberry Pi Imager 1.8.5 (the Ubuntu package) on an Ubuntu 26.04.1 PC, a Raspberry Pi 5 and a 64 GB microSD card.
> Time needed: about 15–25 minutes, mostly downloading and writing.
> Next step afterwards: [Guide 2 — First boot and setting up the Pi](02-first-boot-and-pi-setup.md).

---

## Values used in this guide

These are the values used for this project's Pi. If you set up a different Pi, replace them with your own.

| Setting | Value for this Pi | Notes |
|---|---|---|
| Device | Raspberry Pi 5 | Pick Raspberry Pi 4 if that's your board |
| Operating system | **Ubuntu Server 26.04.1 LTS (64-bit)** | Not 24.04, not 26.10, not Desktop |
| Hostname | `eljohopi` | The Pi's name on the network |
| Username | `eljohopi` | Lowercase |
| Password | `eljohopi` | Published on purpose, because this Pi is for public use. It's also the `sudo` password. |
| Wi-Fi | your home network | Name and password exactly as your PC shows them |
| Wireless LAN country | `CO` | Use `US` if the Pi is used in the USA |
| Time zone | `America/Bogota` | |
| Keyboard layout | `latam` | Only matters for a keyboard plugged into the Pi |
| SSH | On, **password authentication** | Stays on (public-use Pi); SSH keys are optional |

---

## What you need

- **An Ubuntu PC** with internet access.
- **A microSD card** of 64 GB or more, ideally class **A2**. **Everything on it will be erased.**
- **A card reader**, either built into the PC or USB.
- **Your Wi-Fi name and password**, written down exactly as they are. Upper and lower case, spaces and symbols all matter.
- **The password you'll use on the Pi.** For this project's public-use Pi, it's `eljohopi`.

---

## Step 1 — Install Raspberry Pi Imager

In a terminal on the PC:

```bash
sudo apt update
sudo apt install rpi-imager
dpkg-query -W rpi-imager        # shows the installed version, e.g. 1.8.5+noembed-0ubuntu6
```

Open it from the applications menu (**Raspberry Pi Imager**), or run `rpi-imager` in a terminal.

This guide follows Imager **1.8.5**. Imager 2.x, downloaded from raspberrypi.com, shows the same settings as a series of wizard pages instead of tabs. It also has a "Raspberry Pi Connect" step, which you should leave off because it only works with Raspberry Pi OS.

---

## Step 2 — (Only for "Use custom") Download and verify the image yourself

Skip this step if you'll pick Ubuntu from Imager's list (Step 5, route A). In that case Imager downloads and checks the image itself.

If you'd rather use a file you downloaded yourself, keep it **outside** the git repository, because it's 1.5 GB:

```bash
mkdir -p ~/Documents/images && cd ~/Documents/images
wget https://cdimage.ubuntu.com/ubuntu/releases/26.04.1/release/ubuntu-26.04.1-preinstalled-server-arm64+raspi.img.xz
wget https://cdimage.ubuntu.com/ubuntu/releases/26.04.1/release/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing     # must print: ...raspi.img.xz: OK
```

- **Use the `+raspi` file.** The file named `…preinstalled-server-arm64+raspi.img.xz` is the Raspberry Pi version.
- **Don't use the ISO.** The `…live-server-arm64.iso` is a generic installer for PCs and servers, and a Raspberry Pi can't boot it.

---

## Step 3 — Insert the card and identify it

Put the microSD card in the reader. Then, in a terminal:

```bash
lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINTS
```

Find the card by its size. A 64 GB card shows as about **58–60 GB**. It's usually named `mmcblk0` for a built-in reader, or `sdb`/`sdc` with `TRAN` = `usb` for a USB reader. Note the size, because Imager will show the same number in Step 6.

> Your PC's own disks (for example `nvme0n1`, holding Ubuntu and Windows) must **never** be chosen. Knowing the card's size makes sure you don't.

---

## Step 4 — Choose the device

Click **CHOOSE DEVICE** → **Raspberry Pi 5** (or Raspberry Pi 4).

This narrows the OS list down to systems that work on that board.

---

## Step 5 — Choose the operating system

Click **CHOOSE OS** and use **one** of these routes.

**Route A — From Imager's list (recommended):**

**Other general-purpose OS** → **Ubuntu** → **Ubuntu Server 26.04.1 LTS (64-bit)**

Read the name carefully. Several Ubuntu entries sit right next to each other:

| Entry | Use it? | Why |
|---|---|---|
| **Ubuntu Server 26.04.1 LTS (64-bit)** | ✅ Yes | Long-term support until about 2031, and ROS 2 Lyrical has ready-made packages for it |
| Ubuntu Server 26.10 (64-bit) | ❌ | Short-term release (about 9 months), with no ROS 2 packages |
| Ubuntu Server 24.04.x LTS | ❌ | Older; doesn't match the PC's Ubuntu 26.04 and ROS 2 Lyrical |
| Ubuntu Desktop … | ❌ | Includes a full desktop the Pi doesn't need, which costs RAM and CPU |

**Route B — Your downloaded file:**

Scroll to the bottom of the list → **Use custom** → select `ubuntu-26.04.1-preinstalled-server-arm64+raspi.img.xz`. Imager reads `.xz` files directly, so **don't unpack it**. With this route, **Step 12 is required**.

**Check before going on:** on Imager's main screen, under "Operating System", the button must now say **UBUNTU SERVER 26.04.1 LTS (64-BIT)** or show your custom file's name.

> In this project, the first card ended up with Ubuntu 24.04 by mistake. This one look at the main screen prevents that.

---

## Step 6 — Choose the storage

Click **CHOOSE STORAGE** and pick the microSD card. **Check its size** against what `lsblk` showed in Step 3.

- **Leave "Exclude system drives" ticked.** That keeps the PC's own disks off the list.
- **If the card isn't listed,** take it out, put it back in, and wait a few seconds. You can also try another reader or USB port.

---

## Step 7 — Open the customisation settings

Click **NEXT**. Imager asks **"Use OS customisation?"** and offers four buttons:

| Button | What it does |
|---|---|
| **EDIT SETTINGS** | ✅ Opens the settings window. **Click this one.** |
| YES | Applies the settings saved from last time without showing them |
| NO | Writes Ubuntu with **no** settings: default user `ubuntu`/`ubuntu`, and no Wi-Fi |
| NO, CLEAR SETTINGS | Deletes the saved settings and writes without any |

The **OS Customisation** window has three tabs: **GENERAL**, **SERVICES** and **OPTIONS**.

> Imager remembers the values from the last time it was used. Check every field, even if it's already filled in.

---

## Step 8 — GENERAL tab

| Field | What to enter | Why it matters |
|---|---|---|
| ☑ **Set hostname** | `eljohopi` | The Pi's name. Later you can reach it as `eljohopi.local`. Use lowercase letters, digits and `-` only, with no spaces or `_`. |
| ☑ **Set username and password** | Username `eljohopi`; password `eljohopi` | The account you log in with over SSH. The same password is used for `sudo`. Start the username with a letter and keep it lowercase. |
| ☑ **Configure wireless LAN** | — | The Pi joins this network by itself at every boot. It must be **the same network your PC uses**. |
| └ SSID | Your Wi-Fi name, **exactly** | Case, spaces and symbols must match. To copy it from the PC, run `nmcli -t -f active,ssid dev wifi \| grep '^yes'` |
| └ Password | Your Wi-Fi password | Tick **Show password**, check it character by character, then untick. A typo here is the most common reason a Pi never shows up. |
| └ Hidden SSID | Unticked | Tick it only if your network is hidden |
| └ **Wireless LAN country** | `CO` | Sets which radio channels the Pi may legally use. Use the country where the Pi is: `CO` for Colombia, `US` for the USA. **Don't leave it on `GB`**, which follows European rules. |
| ☑ **Set locale settings** | **Tick it** | If it's unticked, the time zone below is greyed out and **not applied** |
| └ Time zone | `America/Bogota` | Correct local times in logs and ROS timestamps |
| └ Keyboard layout | `latam` | Only for a keyboard plugged into the Pi. Over SSH you type on the PC's keyboard. |

> **Campus or enterprise Wi-Fi** (eduroam, or any network where you log in with a username) can't be set up here. Use your home network, a phone hotspot, or the Ethernet cable ([Guide 3](03-ubuntu-pc-ethernet-connection.md)).

---

## Step 9 — SERVICES tab

| Setting | Choice | Why |
|---|---|---|
| ☑ **Enable SSH** | Ticked | Starts the SSH server, the "door" your PC connects through. Without it you can't log in remotely ("Connection refused"). |
| ◉ **Use password authentication** | Selected | You log in with the username and password from the GENERAL tab |
| ○ Allow public-key authentication only | Not selected | It would block password logins. This Pi is for public use, so the password must keep working. |
| `authorized_keys` box | Empty | Only used with the key-only option |
| **RUN SSH-KEYGEN** button | **Don't click** | It would make a key for this Ubuntu install only. Each PC, and each side of a dual boot, needs its own key, and it's easier to add them after the first login. |

---

## Step 10 — OPTIONS tab

| Option | Recommendation |
|---|---|
| Play sound when finished | Your choice |
| Eject media when finished | ✅ Tick it, so Imager safely releases the card when it's done |
| Enable telemetry | Your choice. It only sends Raspberry Pi anonymous statistics about which OS was downloaded, and doesn't affect the Pi. |

---

## Step 11 — Save and write

1. Click **SAVE**. You're back at "Use OS customisation?".
2. Click **YES** to apply the settings you just saved.
3. Imager warns: **"All existing data on '…' will be erased."** Read the device name and size. If it's the microSD card, click **YES**.
4. Ubuntu may ask for **your PC password** so Imager is allowed to write to the card.
5. Wait while Imager downloads (route A only), then **Writing…**, then **Verifying…**. This takes several minutes.
6. When **"Write Successful"** appears, the card has been ejected and you can take it out.

---

## Step 12 — Check the card before the first boot

**Required with "Use custom", optional with route A.**

1. Put the card back into the reader. Ubuntu mounts its small boot partition, usually named **`system-boot`**, and shows it in Files.
2. Open these two files with a text editor:
   - **`user-data`** should contain `eljohopi` (the hostname and username). The password appears as a long scrambled string, which is a hash and normal.
   - **`network-config`** should contain your Wi-Fi name under `access-points`. It contains **only** the Wi-Fi part, with no `eth0` (cable) section. That's how Imager writes it, and it's why the cable needs one extra file after the first login ([Guide 2, Part 5](02-first-boot-and-pi-setup.md#part-5--fix-the-network)).
3. If your username or Wi-Fi name is missing, the settings weren't applied. Write the card again, picking the OS from Imager's list (route A).
4. Eject the card: click the ⏏ icon next to `system-boot` in Files, then take it out.

> `network-config` contains your Wi-Fi password. Don't copy it into the repo, a chat or a screenshot.

---

## Step 13 — (Shared PCs) Remove the saved settings

Imager saves your settings, **including the Wi-Fi password**, on the PC so it can fill them in next time. On a PC that isn't yours, delete them afterwards:

```bash
rm -f ~/.config/Raspberry\ Pi/Imager.conf
```

---

## What's next

Put the card into the Pi, but **don't power it on yet.** First prepare the PC's cable connection ([Guide 3](03-ubuntu-pc-ethernet-connection.md), Parts 1–4), then follow [Guide 2 — First boot and setting up the Pi](02-first-boot-and-pi-setup.md).

> **Expect one extra fix at the first login.** With Imager 1.8.5 and Ubuntu 26.04.1, the Pi's first boot leaves the network partly unconfigured: the cable has no settings, and the network service may not start. Guide 2 explains the fix step by step.

---

## Troubleshooting

| Problem | What to do |
|---|---|
| "Ubuntu Server 26.04.1 LTS (64-bit)" isn't in the list | Use route B ("Use custom") with the file from Step 2 |
| The card doesn't appear in CHOOSE STORAGE | Take it out and put it back in, or try another reader or port. Check that `lsblk` shows it. |
| Writing or verifying fails | Try again. If it fails twice, the card may be faulty or fake; use another one. `sudo dmesg \| tail -30` shows card errors. |
| "Error opening device" / permission error | Enter your PC password when asked, then try again |
| `user-data` doesn't mention your username | The settings weren't applied. Rewrite using route A. |
| After booting, the Pi runs Ubuntu 24.04 | The wrong image was written. Rewrite and check the main screen in Step 5. Also unplug any SSD or USB drive from the Pi, because it may be starting from that instead. |
