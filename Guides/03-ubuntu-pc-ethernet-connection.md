# Guide 3 — Connect any Ubuntu PC to the Pi with an Ethernet cable

This guide sets up an Ubuntu PC, from scratch, to reach the Pi through a **direct Ethernet cable**. No Wi-Fi or router is needed. When you're done, `ssh eljohopi@192.168.77.2`, or simply `ssh rpi-cable`, works every time you connect the cable.

> Tested on 2026-11-05 on an Ubuntu 26.04.1 PC (Ethernet port `enp4s0`) with the Raspberry Pi 5 `eljohopi`. It works the same on Ubuntu Desktop 22.04 and 24.04, which also use NetworkManager.
> Time needed: about 10 minutes.
> **Requirement:** the Pi already has its fixed cable address `192.168.77.2` ([Guide 2, Part 5.1](02-first-boot-and-pi-setup.md#51-give-the-cable-its-settings)). If it doesn't yet, you can still get in with [Part 8](#part-8--no-setup-at-all-the-link-local-address) of this guide.

---

## How it works

```
   [Ubuntu PC] =========== Ethernet cable =========== [Raspberry Pi]
  192.168.77.1                                        192.168.77.2
  (you set this once, in Part 3)                      (already set on the Pi)
```

- **Why fixed addresses:** a cable between two computers has no router to hand out addresses. So both ends get a **fixed address in the same range** (`192.168.77.x`, mask `255.255.255.0`), and they can always find each other.
- **The Pi's side never changes:** it is always `192.168.77.2`. Only the PC needs setting up, once.
- **Wi-Fi isn't affected:** the PC keeps using Wi-Fi for the internet as usual. Only traffic for `192.168.77.x` goes through the cable.
- **Optional internet sharing:** the PC can also share its internet with the Pi through the cable (option A in Part 3), so the Pi can run `apt` without Wi-Fi.

### Values used in this guide

| What | Value |
|---|---|
| Pi cable address | `192.168.77.2` |
| PC cable address | `192.168.77.1` |
| Mask | `255.255.255.0` (written `/24`) |
| Gateway | **none** |
| Pi username | `eljohopi` |
| Pi password | `eljohopi` (published on purpose: public-use Pi) |
| Pi SSH fingerprint | `SHA256:Wfg76GKg1mLrQLbDDW9ElpJQCNAgNSca4a+DSPwa5Po` |
| Pi link-local address | `fe80::2ecf:67ff:fe6c:4e6e` |

---

## Part 1 — What you need

- **An Ubuntu PC** with an Ethernet port. A laptop without one needs a **USB-to-Ethernet adapter**.
- **Any normal Ethernet cable** (Cat5e or Cat6). You don't need a crossover cable.
- **The Pi**, powered on and set up as in Guide 2.
- **The Pi's password:** `eljohopi`. It's published on purpose, because this Pi is for public use.
- **Possibly your PC password:** Ubuntu may ask for it when you change network settings.

---

## Part 2 — Find the PC's Ethernet port name

In a terminal on the PC:

```bash
nmcli device status
```

Example output:

```
DEVICE   TYPE      STATE         CONNECTION
wlo1     wifi      connected     <your Wi-Fi>
enp4s0   ethernet  unavailable   --
lo       loopback  connected     lo
```

The line with `TYPE` = **ethernet** is your port. Typical names:

| Name | Meaning |
|---|---|
| `enp4s0`, `enp3s0`, `eno1` | Built-in Ethernet port |
| `enx00e04c68…` (long, with letters and digits) | USB-to-Ethernet adapter |

Save the name in a variable for the following commands (replace `enp4s0` with yours). The variable only lasts in this terminal window:

```bash
PORT=enp4s0
```

Make sure no network on this PC already uses `192.168.77.x`:

```bash
ip -br addr | grep 192.168.77 || echo "OK: 192.168.77.x is free"
```

---

## Part 3 — Create the connection profile (once)

NetworkManager, Ubuntu's network manager, keeps named **profiles**: saved sets of network settings. You create one profile just for the Pi. Your normal network settings stay untouched.

Choose **one** of these:

| | **A. `pi-shared`** (recommended for your own PC) | **B. `pi-direct`** (simplest) |
|---|---|---|
| PC address | `192.168.77.1` | `192.168.77.1` |
| The Pi gets internet through this PC | ✅ Yes | ❌ No |
| SSH to `192.168.77.2` | ✅ | ✅ |
| Hands out addresses (runs a DHCP server) on the port | Yes | No |
| Safe to leave on if this port is plugged into an office or university network | ❌ **No.** It would hand out addresses to other people's computers. | Mostly; it just doesn't work there |

### Option A — `pi-shared` (address + internet sharing)

```bash
nmcli connection add type ethernet ifname "$PORT" con-name pi-shared \
  ipv4.method shared ipv4.addresses 192.168.77.1/24
```

- `ipv4.method shared` makes the PC act like a small router on that port: it hands the Pi an extra address (`192.168.77.10–254`) and shares the PC's internet.
- `ipv4.addresses 192.168.77.1/24` gives the PC address `192.168.77.1`. Without it, NetworkManager would use `10.42.0.1`, and `192.168.77.2` wouldn't be reachable.

### Option B — `pi-direct` (fixed address only)

```bash
nmcli connection add type ethernet ifname "$PORT" con-name pi-direct \
  ipv4.method manual ipv4.addresses 192.168.77.1/24
```

With either option, the expected result is:

```
Connection 'pi-shared' (f22b8242-…) successfully added.
```

> ⚠️ **Run the `add` command only once.** Running it twice creates **two profiles with the same name**, and the wrong one can win after a restart. This happened during setup. To check, and delete an extra one:
>
> ```bash
> nmcli -f NAME,UUID,DEVICE connection show | grep pi-
> nmcli connection delete <UUID-of-the-extra-one>
> ```

### Same thing with the Settings app (option B only)

1. **Settings → Network → Wired →** click **+** (add a profile).
2. **Identity** tab: Name `pi-direct`.
3. **IPv4** tab: **Manual** → Address `192.168.77.1`, Netmask `255.255.255.0`, **Gateway empty**.
4. Click **Add**.

Option A, with its custom `192.168.77.1` address, needs the terminal command above.

### Start automatically or by hand?

By default the profile starts by itself when a cable is connected to that port.

- **The port is only ever used for the Pi:** leave it like that.
- **You also plug this port into normal wired networks** (office, university): turn automatic start off, and switch the profile on yourself when you connect the Pi. This is **important for `pi-shared`**:

  ```bash
  nmcli connection modify pi-shared connection.autoconnect no
  ```

---

## Part 4 — Connect the cable and switch the profile on

1. **Connect the cable** from the PC's Ethernet port to the Pi's Ethernet port. The Pi should be on, with a green LED.
2. **Check that the cable has a link:**

   ```bash
   cat /sys/class/net/$PORT/carrier      # 1 = link OK, 0 = no link
   ```

   With a link, the lights on both Ethernet ports light up or blink.
3. **Switch the profile on** (use `pi-direct` if you chose option B):

   ```bash
   nmcli connection up pi-shared
   ```

   Expected: `Connection successfully activated …`
4. **Check the PC's address on that port:**

   ```bash
   ip -br addr show "$PORT"      # must show 192.168.77.1/24
   ```

---

## Part 5 — Test and log in

```bash
ping -c 3 192.168.77.2
```

Expected: three replies, each in less than 1 ms. Then log in:

```bash
ssh eljohopi@192.168.77.2
```

The first time on this PC, you'll see:

```
The authenticity of host '192.168.77.2 (192.168.77.2)' can't be established.
ED25519 key fingerprint is: SHA256:Wfg76GKg1mLrQLbDDW9ElpJQCNAgNSca4a+DSPwa5Po
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

1. **Compare the fingerprint** with the table at the top. If it matches, type `yes`.
2. **Enter the Pi's password, `eljohopi`.** Nothing appears while you type.
3. **The prompt changes** to `eljohopi@eljohopi:~$`, which means you're on the Pi.

Checks on the Pi:

```bash
ip -br addr show eth0      # 192.168.77.2/24 (+ a 192.168.77.x address with option A)
ping -c 3 ubuntu.com       # option A: replies (internet through the PC); option B: fails, which is expected
```

To leave the Pi:

```bash
exit
```

---

## Part 6 — Make it comfortable (recommended)

### 6.1 A short name: `ssh rpi-cable`

On the PC:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cat >> ~/.ssh/config <<'EOF'

Host rpi-cable
    HostName 192.168.77.2
    User eljohopi
EOF
chmod 600 ~/.ssh/config
```

From now on:

```bash
ssh rpi-cable
```

### 6.2 (Optional) An SSH key for this PC (no more password typing)

```bash
ssh-keygen -t ed25519 -C "$(whoami)@$(hostname)"   # press Enter to accept the file name
ssh-copy-id rpi-cable                               # asks for the Pi's password one last time
ssh rpi-cable                                       # should log in without asking for the Pi's password
```

- **The passphrase question:** a passphrase protects the key if someone copies it from your PC. Leaving it empty is more convenient; setting one is safer.
- **Each PC needs its own key,** and so does each side of a dual boot (Ubuntu and Windows).
- **Password logins stay on** (public-use Pi). `ssh-copy-id` works from any new PC, and PCs without a key can still log in with the password `eljohopi`.

### 6.3 Edit code on the Pi with VS Code (optional)

1. Install the **Remote - SSH** extension in VS Code.
2. Press `F1` → **Remote-SSH: Connect to Host…** → `rpi-cable`.
3. The first time, VS Code installs a small helper on the Pi, which takes a minute or two.
4. Choose **Open Folder** → for example `/home/eljohopi`.

### 6.4 Copy files

```bash
scp myfile.txt rpi-cable:~/          # PC → Pi
scp rpi-cable:~/map.pgm .            # Pi → PC
```

---

## Part 7 — Daily use

1. Connect the cable and power on the Pi. Wait about a minute.
2. If the profile doesn't start by itself, run `nmcli connection up pi-shared` (or `pi-direct`).
3. Connect with `ssh rpi-cable`, or `ssh eljohopi@192.168.77.2`.
4. When you're done, `exit` leaves the Pi. To turn the Pi off: `sudo poweroff`, wait for the red LED, then unplug.
5. **Before plugging this PC's Ethernet port into a normal network**, run `nmcli connection down pi-shared`.

---

## Part 8 — No setup at all: the link-local address

**When to use it:** on a PC where you can't or don't want to create a profile, or when the Pi's cable settings are missing (for example on a freshly written card).

Every Ethernet port gives itself a link-local address (`fe80::…`) as soon as it has a link, with no configuration. Add the PC's port name after `%`:

```bash
ssh eljohopi@fe80::2ecf:67ff:fe6c:4e6e%$PORT
```

- **This Pi's part,** `fe80::2ecf:67ff:fe6c:4e6e`, comes from its hardware ID and never changes. On the Pi, `ip -br addr show eth0` shows it.
- **If you don't know the Pi's link-local address,** list every device on the cable. The address that isn't the PC's own is the Pi:

  ```bash
  ping -6 -c 3 ff02::1%$PORT
  ```

- **If you get "Network is unreachable",** the PC's port isn't active. Switch it on with a link-local-only profile:

  ```bash
  nmcli connection add type ethernet ifname "$PORT" con-name pi-linklocal ipv4.method disabled ipv6.method link-local
  nmcli connection up pi-linklocal
  ```

---

## Part 9 — Change or remove the setup

| Goal | Command (on the PC) |
|---|---|
| Switch the profile off | `nmcli connection down pi-shared` |
| Switch it back on | `nmcli connection up pi-shared` |
| Delete it | `nmcli connection delete pi-shared` |
| See its settings | `nmcli connection show pi-shared \| grep -E 'ipv4.(method\|addresses)\|autoconnect'` |
| Use the port with a different adapter | `nmcli connection modify pi-shared connection.interface-name <new-port>` |

**Changing the address range** is only needed if `192.168.77.x` clashes with a network you use. Change **both** ends, the Pi first, over Wi-Fi:

1. **On the Pi:** edit `/etc/netplan/90-direct.yaml`, set for example `addresses: [192.168.88.2/24]`, then run `sudo netplan apply`.
2. **On the PC:** run `nmcli connection modify pi-shared ipv4.addresses 192.168.88.1/24`, then `nmcli connection up pi-shared`.

---

## Troubleshooting

| What you see | Likely cause | What to do |
|---|---|---|
| `carrier` is `0`, and the port lights are off | Cable not pushed in, Pi off, or a bad cable | Push both ends in fully; check the Pi's LED; try another cable |
| `ping: connect: Network is unreachable` | The profile isn't active, so the PC has no `192.168.77.x` address | `nmcli connection up pi-shared`; check `ip -br addr show $PORT` |
| `Destination Host Unreachable`, or no replies | The Pi is off or still booting, or its cable settings are missing | Wait 2–3 minutes. Try [Part 8](#part-8--no-setup-at-all-the-link-local-address). On the Pi, check `ip -br addr show eth0` and redo Guide 2, Part 5.1 if `192.168.77.2` is missing. |
| `Connection activation failed` for `pi-shared` | The sharing helper (`dnsmasq-base`) is missing | `sudo apt install dnsmasq-base`, then try again |
| `Warning: There is another connection with the name 'pi-shared'` | The `add` command ran twice | Part 3: list the UUIDs and delete the extra profile |
| `ssh: … Connection refused` | The Pi is reachable, but SSH isn't running yet | Wait a minute. On the Pi: `sudo systemctl status ssh`. |
| `Permission denied, please try again` | Wrong username or password | Username and password are both `eljohopi`, in lowercase |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | The Pi's card was rewritten (new identity), or another device is using that address | If you rewrote the card: `ssh-keygen -R 192.168.77.2`, then connect again. Otherwise, find out why first. |
| The Pi has no internet over the cable | Using `pi-direct`; the PC is offline; or the PC's firewall blocks sharing | Use `pi-shared`; check the PC's internet; if `sudo ufw status` says active, run `sudo ufw allow in on $PORT` |
| `eljohopi.local` connects over Wi-Fi instead of the cable | The name resolves to whichever address answers first | Use `192.168.77.2` or `rpi-cable` to make sure you use the cable |
| It worked before, but not with a different USB adapter | A different adapter has a different port name | `nmcli connection modify pi-shared connection.interface-name <new-name>` |
