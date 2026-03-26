# Pre-configured Camera Shipping

## Overview

Kiki Terminals can be shipped together with ONVIF PoE cameras that are already
configured. The customer plugs the cameras into their router or PoE switch with the
included Ethernet cables, connects the terminal to their home WiFi as usual, and the
cameras appear automatically — no manual IP entry required.

**Supported cameras:** ONVIF-capable PoE cameras only. WiFi cameras are not supported
by this feature (they would require a separate WiFi credential handoff step that isn't
implemented).

**Why PoE only:** PoE cameras connect via Ethernet directly to the customer's router or
a PoE switch. They don't need to join a WiFi network — the Kiki Terminal and all cameras
end up on the same subnet automatically once the terminal connects to home WiFi.

---

## Why IP addresses can't be pre-stored

IP cameras receive their address from the customer's router via DHCP. The address
assigned on the factory bench will be different from the one assigned at the
customer's home. The stable identifier used for matching is the **ONVIF serial number**
returned by `GetDeviceInformation()`, which is fixed per physical device and survives
factory resets and firmware updates.

---

## Customer Experience

**Terminal without pre-configured cameras (unchanged behaviour):**
- Home screen shows "No cameras found — Add camera" button

**Terminal with pre-configured cameras:**
- Home screen shows a "Cameras have been pre-configured" card with a **"Find my cameras"**
  button (shown only when cameras with `pending_rediscovery=1` exist)
- Customer plugs cameras into their PoE switch or router, then clicks the button
- Terminal scans the network, matches cameras by serial number, activates them
- Status updates in real time; page refreshes automatically on completion
- Cameras appear on the Cameras page and begin recording normally
- Customer can rename cameras via the existing camera settings

**Cameras page:**
- Pending cameras show a yellow **"Searching…"** badge next to their name
- Their IP is blank; the info line shows the serial number and "Awaiting network discovery"
- Once matched, the badge disappears and the camera behaves normally

---

## Factory Operator Workflow

No account creation is required. The terminal web UI is auto-logged in when no
password has been set up yet.

### Step-by-step

**1. Connect everything to the bench network**

Plug the terminal and all cameras into the same Ethernet switch. Both the Pi and
the cameras need to be on the same subnet (any /24 will do).

**2. Open the terminal web UI**

Navigate to `http://<terminal-ip>:5000` in a browser. You will be auto-logged in.
If you see a login page, use the admin credentials you set during a prior session.

**3. Add each camera via the Cameras page**

Go to **Cameras** in the nav. Click **Add Camera** and fill in:
- **Name** — the name the customer will see (e.g. "Front Door", "Garden")
- **IP** — the camera's current bench IP
- **Username / Password** — the camera's ONVIF credentials
- **ONVIF** — tick the ONVIF checkbox, click **Auto-detect profiles**, select a profile

The system will test the connection before saving.

**4. Mark each camera for shipping**

On each saved camera card, click the **"Mark for shipping"** button (teal-outlined,
top-right of the card).

What this does:
- Connects to the camera's current bench IP via ONVIF
- Reads `GetDeviceInformation().SerialNumber`
- Stores the serial in the `onvif_serial` column
- Sets `pending_rediscovery = 1`
- Clears the stored IP (sets it to `''`) so no stale address ships

A confirmation dialog appears: click **OK** after verifying the serial number shown.
The camera card immediately shows the "Searching…" badge — this is expected.

> If "Mark for shipping" returns an error, the camera is not reachable at its stored IP.
> Fix the IP in the camera settings first, then retry.

**5. Run `prepare_for_shipping.sh`**

```bash
sudo ./prepare_for_shipping.sh
```

Because cameras are pre-configured, the script:
- **Preserves** the `cameras` table (with serials and pending flags)
- **Preserves** the `FERNET_KEY` in `.env` (required to decrypt camera passwords)
- **Clears** personal data only:
  - `admin_password_hash`, `admin_username`
  - `notify_email`, `network_ssid`, `notify_last_sent_ts`, `device_api_key`
  - `camera_rediscovery_status`
  - All rows in `events`, `detections`, `recordings`, `notification_emails`
  - The `recordings/` and `clip_cache/` directories

Without pre-configured cameras the script behaves as before: full DB wipe + new
FERNET_KEY.

**6. Verify before packing**

Check the Cameras page — all cameras should show the "Searching…" badge with their
serial number visible. The Cameras page confirms serials are stored and the terminal
is ready to ship.

**7. Pack and ship**

Include in the box:
- Kiki Terminal (Raspberry Pi)
- Each pre-configured PoE camera
- PoE switch (or PoE injectors if using the customer's own router)
- Ethernet cables (one per camera + one for the terminal if wired)
- Setup instructions card

---

## How Auto-Discovery Works (Technical Detail)

### Trigger

Discovery runs automatically in a background thread as soon as the customer's
terminal successfully joins their home WiFi (`ap_setup_connect` → `do_connect()`).
It can also be triggered manually via the **"Find my cameras"** button on the home
screen.

### Discovery algorithm

```
1. Query DB for cameras WHERE pending_rediscovery=1 AND onvif_serial IS NOT NULL

2. Run ONVIF WS-Discovery (multicast, timeout=8s)
   → returns list of {ip, name, manufacturer, model} for responding devices

3. TCP port-80 subnet probe (fallback for cameras that don't respond to WS-Discovery)
   → get local IP, derive /24 prefix
   → probe all 254 IPs in parallel (ThreadPoolExecutor, 10 workers, 0.3s timeout)
   → add any IP responding on port 80 that wasn't found by WS-Discovery

4. For each pending camera × each candidate IP:
   → call get_device_serial(ip, username, password, onvif_port)
   → compare returned SerialNumber against stored onvif_serial
   → on match: UPDATE cameras SET ip=<matched_ip>, pending_rediscovery=0

5. Write status to settings:
   → "complete"  if all cameras matched
   → "partial"   if some cameras matched
   → "failed"    on exception
```

### Recorder guard

`load_cameras_from_db()` in `recorder.py` includes:
```sql
WHERE pending_rediscovery=0 OR pending_rediscovery IS NULL
```
Pending cameras are invisible to the recorder until matched, preventing
connection-refused errors on empty IPs.

### Polling

The "Find my cameras" button POSTs to `/cameras/find` (starts background thread,
returns immediately). The page then polls `/cameras/rediscovery-status` every 5
seconds. On `"complete"` the page reloads; on `"partial"` or `"failed"` an inline
message is shown and the button re-enables.

---

## Code Reference

| File | What was added |
|------|----------------|
| `onvif_helper.py` | `get_device_serial(ip, user, pass, port) -> str \| None` |
| `app.py` | DB migrations for `onvif_serial`, `pending_rediscovery`; `rediscover_pending_cameras()`; routes: `POST /cameras/find`, `GET /cameras/rediscovery-status`, `POST /cameras/<id>/mark-for-shipping`; updated `home()` and `cameras()` routes |
| `recorder.py` | `pending_rediscovery` guard in `load_cameras_from_db()` |
| `prepare_for_shipping.sh` | Conditional DB clear based on camera count |
| `templates/home.html` | "Find my cameras" card + polling JS |
| `templates/cameras.html` | "Searching…" badge, "Mark for shipping" button |

---

## What is Not Built

- WiFi camera onboarding (cameras must be PoE/wired)
- Continuous background rediscovery loop (the recorder's existing ONVIF reconnect
  logic handles runtime IP changes after initial resolution)
- MAC address matching (ONVIF serial is more reliable and already accessible)
- Re-encryption migration (FERNET_KEY is preserved, not regenerated)
- Automatic serial capture without a live camera (operator must have camera connected)
