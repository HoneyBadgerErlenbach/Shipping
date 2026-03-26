# Pre-configured Camera Shipping

## Overview

Kiki Terminals can be shipped together with ONVIF PoE cameras that are already
configured. The customer plugs the cameras into their router or PoE switch with the
included Ethernet cables, connects the terminal to their home WiFi as usual, and the
cameras appear automatically — no manual IP entry required.

**Supported cameras:** ONVIF-capable PoE cameras only. WiFi cameras are not supported
by this feature (they would require a separate WiFi credential handoff step).

---

## Why IP addresses can't be pre-stored

IP cameras receive their address from the customer's router via DHCP. The address
assigned on the factory bench will be different from the one assigned at the
customer's home. The stable identifier used for matching is the **ONVIF serial number**
returned by `GetDeviceInformation()`, which is fixed per physical device.

---

## Customer Experience

**Terminal without pre-configured cameras (existing behaviour):**
- Home screen shows "No cameras found — Add camera"

**Terminal with pre-configured cameras:**
- Home screen shows "Find my cameras" button
- Customer clicks it after plugging cameras into their network
- Terminal scans the network, matches cameras by serial number, activates them
- Status updates in real time; after completion, cameras appear normally
- Customer can rename cameras immediately via inline edit fields

---

## Factory Operator Workflow

No account creation required. The terminal web UI is accessible without a password
when no account has been set up yet.

1. Connect the terminal and cameras to the same bench network (any switch)
2. Open `http://kiki.local:5000` in a browser — auto-logged in
3. Go to **Cameras** → add each camera the normal way (name, IP, credentials, ONVIF)
4. Click **"Mark for shipping"** on each camera card
   - App captures the ONVIF serial number from the camera
   - Stores the serial, clears the bench IP, flags as pending discovery
5. Run `prepare_for_shipping.sh` — sees cameras configured, preserves them and the
   FERNET key, clears only personal data (recordings, events, account settings)
6. Pack and ship: terminal + cameras + PoE switch + Ethernet cables

---

## How Auto-Discovery Works

After the customer connects the terminal to their home WiFi:

1. Terminal runs ONVIF WS-Discovery (multicast broadcast on the local subnet)
2. For each responding device, fetches its serial number via `GetDeviceInformation()`
3. Compares against pre-configured camera serials stored in the DB
4. On match: updates the camera's IP, clears the pending flag, recorder starts streaming
5. Fallback: if WS-Discovery misses any cameras (some models don't respond to multicast),
   falls back to a TCP port-80 probe across the /24 subnet before retrying serial matching

Works across WiFi/Ethernet boundaries on the same subnet — the terminal stays on WiFi,
cameras are wired, all on the same router/subnet.

---

## Changes Required for Implementation

### Database (`app.py` — `init_db()`)
Two new columns on the `cameras` table, added via the existing `ALTER TABLE` migration
pattern:

```sql
onvif_serial        TEXT    DEFAULT NULL   -- SerialNumber from GetDeviceInformation()
pending_rediscovery INTEGER DEFAULT 0     -- 1 = awaiting IP resolution on customer network
```

### `onvif_helper.py`
New function `get_device_serial(ip, username, password, port=80) -> str | None`:
- Thin wrapper around the existing `test_onvif_connection()` pattern
- Calls `GetDeviceInformation()` and returns `SerialNumber`

### `app.py`
- Schema migration for the two new columns
- `rediscover_pending_cameras()` function (discovery + serial matching + IP update)
- Called from `ap_setup_connect()` background thread after successful WiFi join
- `POST /cameras/find` — manual trigger for the "Find my cameras" button
- `GET /cameras/rediscovery-status` — polling endpoint for real-time UI updates
- `POST /cameras/<id>/mark-for-shipping` — captures serial, sets pending flag, clears IP
- `load_cameras_from_db()` — add `WHERE pending_rediscovery = 0` guard so the recorder
  ignores cameras until their IPs are resolved
- Home screen: pass `has_pending` flag to template

### `prepare_for_shipping.sh`
Make DB deletion and FERNET_KEY regeneration conditional on whether cameras are
pre-configured:

```bash
if cameras exist in DB:
    # Preserve: cameras table, FERNET_KEY in .env
    # Clear: recordings dir, events/detections rows, personal settings
    #        (admin_password_hash, admin_username, notify_email,
    #         network_ssid, notify_last_sent_ts, device_api_key)
else:
    # Existing behaviour: delete storage.db, regenerate FERNET_KEY
```

### Templates
- `templates/index.html` — "Find my cameras" card, shown only when `has_pending=True`
- `templates/cameras.html` — "Searching..." badge for pending cameras; JS polling

---

## What is Not Built

- WiFi camera onboarding
- Continuous background rediscovery (recorder's existing retry/ONVIF logic handles
  runtime IP changes after initial resolution)
- MAC address matching (ONVIF serial is more reliable and already accessible)
- Re-encryption migration (FERNET_KEY is preserved, not regenerated)
