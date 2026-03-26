# New Raspberry Pi — Preparation & Shipping Guide

## Context
Step-by-step guide for preparing a fresh Raspberry Pi for customer shipping.
Based directly on `install.sh` and `prepare_for_shipping.sh` in the codebase.
End goal: a device that powers on and immediately broadcasts "Kiki-Setup" WiFi
for the customer to complete setup themselves.

---

## What you need before you start
- Raspberry Pi with onboard WiFi (Pi 4 or Pi 5 recommended)
- microSD card (32 GB+) and a card reader
- Power supply and a way to SSH in (ethernet cable or temporary WiFi)
- `TAILSCALE_AUTHKEY` — generate one at tailscale.com → Settings → Keys → Generate Auth Key

---

## Step 1 — Flash the OS

1. Download and open **Raspberry Pi Imager**
2. Choose **Raspberry Pi OS Lite (64-bit)** — no desktop needed
3. Click the gear icon (⚙) and configure:
   - **Hostname:** `kiki`
   - **Enable SSH** with password authentication
   - **Username / password:** `pi` + a strong password
   - **WiFi:** enter your workshop WiFi so the Pi can reach the internet during installation
   - **Locale / timezone:** set appropriately
4. Flash the SD card, insert into the Pi, power on
5. SSH in: `ssh pi@kiki.local` (or find the IP from your router's device list)

---

## Step 2 — Clone the repository

The repo is private — you need a GitHub personal access token (classic, with `repo` scope).

```bash
cd /home/pi
git clone https://<YOUR_GITHUB_TOKEN>@github.com/HoneyBadgerErlenbach/HoneyBadgerCameraSystem.git
cd HoneyBadgerCameraSystem
```

Replace `<YOUR_GITHUB_TOKEN>` with your token (or ask the team lead for it).

---

## Step 3 — Run the installer

```bash
sudo ./install.sh
```

This takes **15–30 minutes**. It will:
- Install all system packages (ffmpeg, NetworkManager, OpenCV deps, QR code tools, etc.)
- Install Tailscale
- Switch network management from `dhcpcd` → `NetworkManager` (required for AP/hotspot mode)
- Create a Python virtual environment under `venv/`
- Install all Python packages from `requirements.txt` (Flask, OpenCV, etc.)
- Print a verification summary — all items should show a version number, not "MISSING"


---

## Step 4 — Set the required secrets

First, generate your Tailscale auth key:

1. Go to **tailscale.com** → log in → **Settings** → **Keys**
2. Click **Generate auth key**
3. Configure it:
   - **Reusable:** No (one-time use per device)
   - **Expiry:** 90 days
   - Tags: none needed
4. Click **Generate key** — copy it immediately (it won't be shown again)

Then open the `.env` file on the Pi:

```bash
nano /home/pi/HoneyBadgerCameraSystem/.env
```

The file already contains a blank `TAILSCALE_AUTHKEY=` line added by the installer.
Paste your key directly after the `=`, so it looks like this:

```
TAILSCALE_AUTHKEY=tskey-auth-xxxxxxxxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**Rules:**
- No quotes, no spaces around the `=`
- One line, key starts with `tskey-auth-`
- Generate a **new key per device** — each key is one-time use

Save and close (`Ctrl+X → Y → Enter`).

> **Note:** Each device automatically generates its own unique API key at runtime — no shared secret needs to be set manually. `KIKI_API_URL` is pre-populated by the shipping script.

---

## Step 5 — Pre-configure cameras *(only when shipping cameras with the terminal)*

Skip this step if the customer is adding their own cameras later.

See **[pre-configured-camera-shipping.md](pre-configured-camera-shipping.md)** for full detail. Summary:

**5a — Prepare each camera (one-time per camera, via manufacturer app)**
- Set the admin password
- Enable ONVIF (Reolink: Settings → Network → Advanced → ONVIF → Enable, port **8000**)
- Assign a static IP in the `192.168.4.x` range (e.g. `192.168.4.10`, `.11`, …)

**5b — Connect to the bench**
- Plug all cameras into a switch
- Power on the Pi — it broadcasts `Kiki-Setup` automatically
- Connect your laptop to `Kiki-Setup` WiFi
- Open `http://192.168.4.1:5000` — you are auto-logged in

**5c — Register each camera in the Kiki UI**

Go to **Cameras → Add Camera** and fill in:
- **Name** — what the customer will see (e.g. "Front Door")
- **IP** — the camera's static bench IP (e.g. `192.168.4.10`)
- **Username / Password** — the camera's ONVIF credentials
- **ONVIF** — tick the checkbox, set port to **8000** for Reolink (80 for most others), click **Auto-detect profiles**, select a profile

**5d — Mark each camera for shipping**

On each saved camera card, click **"Mark for shipping"**:
- The app reads the ONVIF serial number from the live camera
- Stores the serial, clears the bench IP, flags the camera as pending discovery
- A "Searching…" badge appears on the card — this is correct

> If the button returns an error, the camera is not reachable at its stored IP — check the static IP assignment and retry.

After all cameras are marked, all camera cards should show the "Searching…" badge.

---

## Step 6 — Run the shipping preparation script

```bash
sudo ./prepare_for_shipping.sh
```

Type `yes` when prompted. This is fully automated:

| # | What it does |
|---|-------------|
| 1 | Stops any running services |
| 2 | **If cameras are pre-configured:** clears only personal data (account, WiFi credentials, recordings) — preserves the cameras table and `FERNET_KEY`. **Otherwise:** wipes the entire database and generates a fresh `FERNET_KEY` |
| 3 | Compiles Python source to native `.so` binaries with Cython, strips debug symbols, removes `.py` source files and `.git` history |
| 4 | Sets restrictive file permissions (`700` dirs, `600` secrets) |
| 5 | Installs and enables `honeybadger.service` in systemd (auto-starts on boot) |
| 6 | Adds sudoers and polkit rules so the service can manage iptables and NetworkManager |
| 7 | Creates the `.first_boot` marker file |
| 8 | Registers this device's Cloudflare tunnel hostname (`kiki-XXXXXX.kiki-technologies.com`) |
| 9 | Deletes all saved WiFi connections — forces AP mode on the customer's first boot |
| 10 | Connects the device to Tailscale as `kiki-XXXXXX` |
| 11 | Starts the service → AP mode activates → "Kiki-Setup" WiFi appears |

The device ID (`XXXXXX`) is derived from the last 6 characters of the WiFi MAC address.

---

## Step 7 — Verify the device

1. On your phone, open WiFi settings
2. Confirm **Kiki-Setup** appears — it is WPA2-protected
3. The WiFi password is printed on the setup page at `http://192.168.4.1:5000`, and is
   also derived from the device MAC: **`kiki-XXXXXX`** where `XXXXXX` is the last 6 hex
   digits of the WiFi MAC address (uppercase, no colons).
   Example: MAC `dc:a6:32:a1:b2:c3` → password `kiki-A1B2C3`
4. Connect to **Kiki-Setup** using that password
5. The setup page should open automatically as a captive portal
6. If it doesn't open automatically, navigate to `http://192.168.4.1:5000`
7. The WiFi + email setup form should be visible, with the hotspot password displayed on-screen
8. Disconnect your phone from Kiki-Setup

If "Kiki-Setup" does not appear, check logs:
```bash
sudo journalctl -u honeybadger -n 50
```

---

## Step 8 — Shut down, package, and ship

```bash
sudo shutdown -h now
```

- Ensure the SD card is seated firmly
- Box with the power supply cable
- If shipping with cameras: include each camera, the PoE switch, and Ethernet cables (one per camera)
- Attach the printed QR label to the bottom of the device (the label includes the Kiki-Setup WiFi password)

---

## What the customer does on first boot

1. Plugs in the Pi — waits ~60 seconds for it to boot
2. **If cameras were included:** plugs each camera into the PoE switch and connects the switch to their router
3. Sees **"Kiki-Setup"** WiFi on their phone
4. The WiFi password is **`kiki-XXXXXX`** (last 6 hex digits of the device MAC, uppercase) —
   this is printed on the label attached to the bottom of the device
5. Connects to **Kiki-Setup** using that password — setup page opens automatically
6. Selects home WiFi from the list, enters the password and their email address
7. Taps **Connect** — the Pi switches to their home network, "Kiki-Setup" disappears
8. Receives a welcome email with a link to the dashboard on their local network
9. Clicks the link → creates a **username and password**
10. Logs in — a 6-digit verification code is emailed for security
11. **If cameras were pre-configured:** home screen shows **"Find my cameras"** — tap it; the terminal scans the network, matches cameras by serial number, and activates them automatically
12. **If no pre-configured cameras:** adds cameras from the **Cameras** page (ONVIF auto-scan or manual entry)

---

## Troubleshooting quick reference

| Symptom | Fix |
|---------|-----|
| "Kiki-Setup" WiFi doesn't appear | `sudo systemctl restart honeybadger` then check logs |
| Can't connect to "Kiki-Setup" | Password is `kiki-XXXXXX` — check the label on the device bottom |
| Script exits: "DEVICE NOT READY TO SHIP" | `KIKI_API_URL` is blank — set it in `.env` and re-run |
| Tailscale step skipped | Add `TAILSCALE_AUTHKEY` to `.env` before running prep |
| Cloudflare DNS registration warning | Ensure `cloudflared` is installed and the tunnel is authenticated |
| Customer never receives welcome email | Check that `KIKI_API_URL` in `.env` points to the correct cloud server |


