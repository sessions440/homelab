# GL-SFT1200 (Opal) — Access Point Mode Reference

## 1. Initial Setup: Configure SSID & Password, Then Switch to AP Mode

> **Key rule:** Set the SSID and password **before** switching to Access Point mode.

1. Connect to the Opal's default Wi-Fi (SSID on the bottom label, password `goodlife`) or via Ethernet to a LAN port.
2. Browse to `192.168.8.1` and set your admin password.
3. Go to **Wireless** and, for each band you want enabled (2.4 GHz / 5 GHz):
   - Set **Wi-Fi Name (SSID)** to match your main router's SSID.
   - Set **Wi-Fi Security** to the same encryption (e.g., WPA2-Personal).
   - Set **Wi-Fi Key** to the same password as your main router.
   - Click **Apply**.
4. Go to **Network** → **Network Mode** → select **Access Point** → **Apply**.
5. Plug an Ethernet cable from your main router's **LAN port** into the Opal's **WAN port**.

The Opal now broadcasts the same SSID/password as your main network and hands out no IPs of its own (DHCP/NAT disabled).

---

## 2. Updating SSID or Password After the Device Is in AP Mode

`192.168.8.1` is no longer reachable. Two options:

### Option A — Find the Opal's IP via your main router

1. Log into your main router's admin panel.
2. Open the **DHCP lease table** / connected devices list.
3. Identify the Opal by its **MAC address** (on the bottom label) or hostname.
4. Note its assigned IP (e.g., `192.168.1.50`).
5. Connect your computer to the network (Wi-Fi or Ethernet to a LAN port on the Opal).
6. Browse to that IP, log in, go to **Wireless**, update SSID/password, **Apply**.

> ⚠️ If you change the SSID/password you'll be disconnected immediately. Do this over **Ethernet** or have the new credentials ready.

### Option B — Temporarily revert to Router mode

1. **Hold the reset button for 4 seconds** → Opal reverts to Router mode.
2. Connect to its Wi-Fi (the SSID you previously set) or via Ethernet.
3. Browse to `192.168.8.1`, change SSID/password under **Wireless**.
4. Go to **Network** → **Network Mode** → **Access Point** → **Apply**.
5. Reconnect the Ethernet cable to the **WAN port**.

---

## 3. Reset / Recovery

| Action                                | How                                  |
| ------------------------------------- | ------------------------------------ |
| Revert to Router mode (keep settings) | Hold **Reset** button **4 seconds**  |
| Factory reset (erase all settings)    | Hold **Reset** button **10 seconds** |

After a factory reset the device returns to default SSID (on the label) / password `goodlife` at `192.168.8.1`.

---

## 4. Tip: Reserve a Static IP

In your main router's DHCP settings, **reserve an IP** for the Opal's MAC address. This gives you a fixed address to browse to whenever you need to manage it in AP mode.
