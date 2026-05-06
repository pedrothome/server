# UniFi Network Segmentation Guide

## Dream Machine SE + U7 Pro — VLAN, Firewall & Wi-Fi Setup

---

## Table of Contents

### Core Setup (Do Now)

1. [Network Architecture Overview](#1-network-architecture-overview)
2. [VLAN & Subnet Design](#2-vlan--subnet-design)
3. [Device Assignment](#3-device-assignment)
4. [Wi-Fi Networks](#4-wi-fi-networks)
5. [Step-by-Step Implementation](#5-step-by-step-implementation)
6. [Firewall Zones & Rules](#6-firewall-zones--rules)
7. [Validation & Testing](#9-validation--testing)

### Optional / Future Enhancements

8. [OPTIONAL: Pi-hole as Network-Wide DNS](#7-dns-configuration-dual-pi-hole)
9. [OPTIONAL: Cloudflare Tunnel for Docker Server](#8-cloudflare-tunnel-considerations)
10. [OPTIONAL: Switch Replacement (TP-Link → UniFi)](#optional-switch-replacement)
11. [OPTIONAL: Ceiling Cameras](#optional-ceiling-camera-setup)
12. [OPTIONAL: NAS Storage](#optional-nas-storage)
13. [Future Expansion Notes](#10-future-expansion-notes)

---

## 1. Network Architecture Overview

```
Internet
   │
   ▼
┌─────────────────────────┐
│   Dream Machine SE      │
│   (Router/Firewall)     │
│   192.168.1.1           │
├─────────────────────────┤
│  Port 1: Management     │  ← VLAN 1 (untagged)
│  Port 2: Servers        │  ← VLAN 20 (untagged) — Docker, HA, Pi-hole
│  Port 3: Trunk to U7   │  ← All VLANs (tagged)
│  Port 4-8: As needed   │
└─────────────────────────┘
         │
         ▼
┌─────────────────────────┐
│      U7 Pro             │
│   (Access Point)        │
├─────────────────────────┤
│  SSID: HomeNet          │  → VLAN 10 (Trusted)
│  SSID: HomeIoT          │  → VLAN 30 (IoT)
│  SSID: HomeGuest        │  → VLAN 40 (Guest)
└─────────────────────────┘
```

### Why 5 Networks (not 4)?

I recommend **5 VLANs** instead of the 4 minimum. A dedicated **Servers** VLAN provides:
- Clean firewall rules (IoT → Plex allowed, IoT → everything else blocked)
- Home Assistant can reach IoT devices without being on the same VLAN
- Cloudflare Tunnel traffic isolated from personal devices
- If a server is compromised, lateral movement to personal devices is blocked

---

## 2. VLAN & Subnet Design

| VLAN ID | Name       | Subnet            | Gateway         | DHCP Range            | Purpose                          |
|---------|------------|-------------------|-----------------|-----------------------|----------------------------------|
| 1       | Management | 192.168.1.0/24    | 192.168.1.1     | 192.168.1.100–199     | Network infrastructure only      |
| 10      | Trusted    | 192.168.10.0/24   | 192.168.10.1    | 192.168.10.100–199    | Personal devices, full access    |
| 20      | Servers    | 192.168.20.0/24   | 192.168.20.1    | 192.168.20.100–199    | Docker, HA, Pi-hole              |
| 30      | IoT        | 192.168.30.0/24   | 192.168.30.1    | 192.168.30.100–199    | Smart home, consoles, TVs        |
| 40      | Guest      | 192.168.40.0/24   | 192.168.40.1    | 192.168.40.100–199    | Visitors, internet-only          |

### Static IP Reservations (Servers VLAN 20)

| Device          | IP Address     | MAC Address | Notes                              |
|-----------------|----------------|-------------|------------------------------------|
| Docker Server   | 192.168.20.10  | (your MAC)  | Plex, Immich, Pi-hole #2 (Docker)  |
| Home Assistant  | 192.168.20.11  | (your MAC)  | Smart home controller              |
| Pi-hole (standalone) | 192.168.20.12  | (your MAC)  | Primary DNS for all VLANs     |

### Static IP Reservations (IoT VLAN 30) — Optional but Recommended

| Device              | IP Address     | Notes                    |
|---------------------|----------------|--------------------------|
| TV 1                | 192.168.30.10  | Needs Plex access        |
| TV 2                | 192.168.30.11  | Needs Plex access        |
| Firestick           | 192.168.30.12  | Needs Plex access        |
| Echo Dot 1–5        | 192.168.30.20–24 | Alexa                  |
| PS5                 | 192.168.30.30  | Gaming                   |
| Nintendo Switch 1   | 192.168.30.31  | Gaming                   |
| Nintendo Switch 2   | 192.168.30.32  | Gaming                   |
| Zigbee Hub          | 192.168.30.40  | HA integration           |
| Vacuum Robot        | 192.168.30.41  | HA integration           |
| Cat Feeder 1        | 192.168.30.42  | HA integration           |
| Cat Feeder 2        | 192.168.30.43  | HA integration           |
| Washer/Dryer        | 192.168.30.44  | HA integration           |
| Air Conditioner 1   | 192.168.30.45  | HA integration           |
| Air Conditioner 2   | 192.168.30.46  | HA integration           |
| Creality K1C        | 192.168.30.50  | 3D printer               |
| Kindle              | 192.168.30.60  | E-reader                 |

---

## 3. Device Assignment

| Device                | VLAN        | Connection | Justification                                    |
|-----------------------|-------------|------------|--------------------------------------------------|
| Dream Machine SE      | Management  | —          | Network infrastructure                           |
| U7 Pro                | Management  | Ethernet   | Network infrastructure                           |
| Docker Server         | Servers     | Ethernet   | Runs Plex, Immich, services                      |
| Home Assistant        | Servers     | Ethernet   | Controls IoT, needs cross-VLAN                   |
| Pi-hole               | Servers     | Ethernet   | DNS for all networks                             |
| iPhone 1              | Trusted     | Wi-Fi      | Personal device, full access                     |
| iPhone 2              | Trusted     | Wi-Fi      | Personal device, full access                     |
| iPad 1                | Trusted     | Wi-Fi      | Personal device, full access                     |
| iPad 2                | Trusted     | Wi-Fi      | Personal device, full access                     |
| Apple Watch 1         | Trusted     | Wi-Fi      | Paired with iPhone                               |
| Apple Watch 2         | Trusted     | Wi-Fi      | Paired with iPhone                               |
| Desktop PC            | Trusted     | Ethernet   | Personal workstation                             |
| Personal Laptop       | Trusted     | Wi-Fi      | Personal device                                  |
| Work Laptop           | Trusted     | Wi-Fi      | Needs server access for dev                      |
| Work Dock             | Trusted     | Ethernet   | Connected to work laptop                         |
| TV 1                  | IoT         | Wi-Fi/Eth  | Media consumption, limited access                |
| TV 2                  | IoT         | Wi-Fi/Eth  | Media consumption, limited access                |
| Firestick             | IoT         | Wi-Fi      | Media consumption                                |
| PS5                   | IoT         | Ethernet   | Gaming, internet access                          |
| Nintendo Switch 1     | IoT         | Wi-Fi      | Gaming, internet access                          |
| Nintendo Switch 2     | IoT         | Wi-Fi      | Gaming, internet access                          |
| Echo Dot ×5           | IoT         | Wi-Fi      | Voice assistant                                  |
| Washer/Dryer          | IoT         | Wi-Fi      | Smart appliance                                  |
| Air Conditioner ×2    | IoT         | Wi-Fi      | Smart appliance                                  |
| Vacuum Robot          | IoT         | Wi-Fi      | Smart appliance                                  |
| Cat Feeder ×2         | IoT         | Wi-Fi      | Smart appliance                                  |
| Zigbee Hub            | IoT         | Ethernet   | Bridge for Zigbee devices                        |
| Creality K1C          | IoT         | Wi-Fi      | 3D printer (known security issues)               |
| Kindle                | IoT         | Wi-Fi      | Limited device, internet-only                    |
| Guest devices         | Guest       | Wi-Fi      | Internet-only, isolated                          |

> **Note on Work Laptop:** If your employer requires network isolation, consider creating a dedicated Work VLAN (VLAN 50, 192.168.50.0/24) in the future. For now, Trusted is fine if you need access to your servers.

---

## 4. Wi-Fi Networks

| SSID Name   | VLAN | Security      | Band    | Hidden | Client Isolation | Notes                                   |
|-------------|------|---------------|---------|--------|------------------|-----------------------------------------|
| HomeNet     | 10   | WPA3 Personal | 2.4+5+6| No     | No               | Trusted devices only                    |
| HomeIoT     | 30   | WPA2 Personal | 2.4+5  | Yes    | No               | IoT devices (WPA2 for compatibility)    |
| HomeGuest   | 40   | WPA2/WPA3     | 2.4+5  | No     | Yes              | Guests, captive portal optional         |

### Wi-Fi Design Decisions

- **HomeNet (WPA3):** Maximum security for personal devices. All modern Apple/PC devices support WPA3.
- **HomeIoT (WPA2, Hidden):** Many IoT devices (Echo Dots, smart appliances) don't support WPA3. Hidden SSID adds minor obscurity. Client isolation is OFF because some IoT devices need local discovery (Echo groups, Chromecast, etc.).
- **HomeGuest (WPA2/WPA3 Transitional):** Guests might have older devices. Client isolation ON prevents guests from seeing each other.
- **No separate gaming SSID:** Consoles on IoT Wi-Fi with proper QoS is sufficient. PS5 should use Ethernet if possible.

### Recommended Wi-Fi Passwords

- **HomeNet:** Strong 20+ character passphrase (you manage few devices)
- **HomeIoT:** Strong but simpler 16+ characters (entered on many devices)
- **HomeGuest:** Rotate monthly, share via QR code or guest portal

---

## 5. Step-by-Step Implementation

### Phase 0: Preparation (Before Making Changes)

> ⚠️ **IMPORTANT:** Have physical access to the UDMSE during this process. If you lose access, you can factory reset via the physical button.

1. **Document current state:**
   - Screenshot all current firewall rules
   - Export your UniFi site backup: **Settings → System → Backup → Download**
   - Note all device IPs and MAC addresses from **Clients** tab
   - Note your Pi-hole's current IP and any devices using it as DNS

2. **Update firmware:**
   - Ensure UDMSE is on UniFi OS 4.0+ (required for Zone-Based Firewall)
   - Ensure U7 Pro firmware is current
   - **Settings → System → Updates → Update All**

3. **Plan maintenance window:**
   - All devices will temporarily lose connectivity during network changes
   - Smart home devices will need to reconnect to new SSID
   - Budget 2-4 hours for full implementation

---

### Phase 1: Create VLANs/Networks

#### Step 1.1: Create Trusted Network (VLAN 10)

1. Go to **Settings → Networks**
2. Click **Create New Network**
3. Configure:
   - **Name:** `Trusted`
   - **Router:** Default (UDMSE)
   - **VLAN ID:** `10`
   - **Gateway IP/Subnet:** `192.168.10.1/24`
   - **DHCP Mode:** DHCP Server
   - **DHCP Range:** `192.168.10.100` – `192.168.10.199`
   - **DNS Server:** `192.168.20.12` (Pi-hole — set after Pi-hole is migrated)
   - **Domain Name:** (leave blank or set `trusted.local`)
   - **Multicast DNS (mDNS):** Enable ✓
   - **IGMP Snooping:** Enable ✓
4. Click **Create**

#### Step 1.2: Create Servers Network (VLAN 20)

1. Click **Create New Network**
2. Configure:
   - **Name:** `Servers`
   - **VLAN ID:** `20`
   - **Gateway IP/Subnet:** `192.168.20.1/24`
   - **DHCP Mode:** DHCP Server
   - **DHCP Range:** `192.168.20.100` – `192.168.20.199`
   - **DNS Server:** `192.168.20.12` (Pi-hole, or `1.1.1.1` temporarily)
   - **Multicast DNS (mDNS):** Enable ✓
   - **IGMP Snooping:** Enable ✓
3. Click **Create**

#### Step 1.3: Create IoT Network (VLAN 30)

1. Click **Create New Network**
2. Configure:
   - **Name:** `IoT`
   - **VLAN ID:** `30`
   - **Gateway IP/Subnet:** `192.168.30.1/24`
   - **DHCP Mode:** DHCP Server
   - **DHCP Range:** `192.168.30.100` – `192.168.30.199`
   - **DNS Server:** `192.168.20.12` (Pi-hole)
   - **Multicast DNS (mDNS):** Enable ✓ (needed for Chromecast/AirPlay discovery)
   - **IGMP Snooping:** Enable ✓
3. Click **Create**

#### Step 1.4: Create Guest Network (VLAN 40)

1. Click **Create New Network**
2. Configure:
   - **Name:** `Guest`
   - **VLAN ID:** `40`
   - **Gateway IP/Subnet:** `192.168.40.1/24`
   - **DHCP Mode:** DHCP Server
   - **DHCP Range:** `192.168.40.100` – `192.168.40.199`
   - **DNS Server:** `1.1.1.1, 8.8.8.8` (public DNS, not Pi-hole — guests shouldn't depend on your infra)
   - **Multicast DNS (mDNS):** Disable ✗
   - **IGMP Snooping:** Enable ✓
   - **Network Isolation:** Enable ✓ (this is the "Guest Network" toggle if available)
3. Click **Create**

---

### Phase 2: Configure Port Profiles on UDMSE

#### Step 2.1: Assign Switch Ports

Go to **UniFi Devices → Dream Machine SE → Port Manager**

**Your current physical layout:**

| Port | Current Device          | New VLAN Profile | Network       | Notes                                |
|------|-------------------------|------------------|---------------|--------------------------------------|
| 1    | U7 Pro (AP)             | Trunk (All)      | All VLANs     | Must carry all VLANs for SSIDs       |
| 2    | Ceiling cable (cameras) | IoT or Cameras   | VLAN 30 (or 60)| Single run to ceiling — see camera section |
| 3    | Home Assistant          | Servers          | VLAN 20       | Untagged Servers                     |
| 4    | Docker Server           | Servers          | VLAN 20       | Untagged Servers                     |
| 5    | Pi-hole (standalone)    | Servers          | VLAN 20       | Untagged Servers                     |
| 6    | Switch → PC + Work Dock | Trusted          | VLAN 10       | See switch replacement below         |
| 7    | Switch → TV, PS5, Switch| IoT              | VLAN 30       | See switch replacement below         |
| 8    | Zigbee Hub              | IoT              | VLAN 30       | Untagged IoT                         |

> **Key insight about your TP-Link switches:** Since Port 6 has ONLY Trusted devices and Port 7 has ONLY IoT devices, unmanaged switches work fine — the UDMSE port sets the VLAN for all traffic on that port. **No replacement needed to implement VLANs.** Upgrading to managed UniFi switches is optional (see [Optional: Switch Replacement](#optional-switch-replacement) below).

#### Step 2.1b: (OPTIONAL FUTURE) UniFi Switch Replacements for TP-Link

**Recommended replacements:**

| Current Switch     | Replacement                | Ports | Price  | Why                                         |
|--------------------|----------------------------|-------|--------|---------------------------------------------|
| TP-Link (Port 6)   | **USW-Flex-Mini**          | 5     | ~$29   | Per-port VLAN, managed, compact, silent     |
| TP-Link (Port 7)   | **USW-Flex-Mini**          | 5     | ~$29   | Per-port VLAN, managed, compact, silent     |

**Alternative options:**

| Model               | Ports | PoE        | Price  | Notes                                     |
|---------------------|-------|------------|--------|-------------------------------------------|
| USW-Flex-Mini       | 5     | No (powered via USB-C or PoE-in) | ~$29  | Best match — tiny, silent, VLAN-aware     |
| **USW-Ultra**       | 8     | 4× PoE+ out | ~$109 | **Best 8-port pick** — see below         |
| USW-Lite-8-PoE      | 8     | 4× PoE out  | ~$109 | 8-port with PoE, slightly older model    |
| USW-Flex            | 5     | 4× PoE out  | ~$99  | PoE powered + PoE out, outdoor-rated     |

**8-port alternative: 2× USW-Ultra (~$109 each, $218 total)**

| Feature             | USW-Flex-Mini (5-port)  | USW-Ultra (8-port)             |
|---------------------|-------------------------|--------------------------------|
| Ports               | 5× 1GbE                 | 8× 1GbE (1× 2.5GbE uplink)   |
| PoE                 | No (PoE-in only)        | 4× PoE+ out (42W budget)      |
| Managed             | Yes                     | Yes                            |
| Per-port VLAN       | Yes                     | Yes                            |
| Power               | USB-C or PoE-in         | Standard AC adapter            |
| Size                | Very compact             | Compact (wall-mountable)       |
| Extra benefit       | Cheapest                 | Power IoT devices via PoE, room to grow |

**Why USW-Ultra is worth considering:**
- 3 extra ports per switch = less likely you'll need to buy another switch later
- PoE+ on 4 ports lets you power future PoE devices (cameras, additional APs, PoE smart devices) without extra adapters
- 2.5GbE uplink on port 1 gives you faster backhaul to the UDMSE (if you use SFP+ or 2.5G ports)
- Still compact and wall-mountable

**If budget is tight:** Go with 2× USW-Flex-Mini now ($58). You can always add a USW-Ultra later if you need more ports or PoE.

**My recommendation: 2× USW-Flex-Mini ($58 total)**

- Same 5-port size as your TP-Links
- Fully managed in UniFi dashboard (auto-adopts)
- Per-port VLAN profiles (assign each port to a specific network)
- Traffic statistics per port
- Powered via USB-C (5V adapter included) or PoE-in on port 1
- Fanless/silent, very compact

#### Step 2.1c: Configure USW-Flex-Mini #1 (Replaces Port 6 TP-Link)

After the switch auto-adopts in UniFi:

Go to **UniFi Devices → USW-Flex-Mini #1 → Port Manager**

| Port | Device         | Profile  | Network  |
|------|----------------|----------|----------|
| 1    | Uplink (UDMSE) | —        | (auto)   |
| 2    | Desktop PC     | Trusted  | VLAN 10  |
| 3    | Work Dock      | Trusted  | VLAN 10  |
| 4    | (available)    | Trusted  | VLAN 10  |
| 5    | (available)    | Trusted  | VLAN 10  |

> Since all devices here are Trusted, you could also set UDMSE Port 6 as "Trusted" (untagged VLAN 10) and leave the Flex-Mini ports on default. Either approach works — per-port profiles on the Flex-Mini gives you flexibility to add a non-Trusted device later.

#### Step 2.1d: Configure USW-Flex-Mini #2 (Replaces Port 7 TP-Link)

Go to **UniFi Devices → USW-Flex-Mini #2 → Port Manager**

| Port | Device           | Profile  | Network  |
|------|------------------|----------|----------|
| 1    | Uplink (UDMSE)   | —        | (auto)   |
| 2    | TV               | IoT      | VLAN 30  |
| 3    | PS5              | IoT      | VLAN 30  |
| 4    | Nintendo Switch  | IoT      | VLAN 30  |
| 5    | (available)      | IoT      | VLAN 30  |

> All devices here are IoT. Same logic applies — you have flexibility to mix VLANs per port if needed in the future (e.g., plugging in a work device temporarily).

**UDMSE Port 6 & 7 profile when using managed switches:**

If you install the USW-Flex-Minis and want per-port VLAN on them, set the UDMSE uplink ports as **Trunk (All)**:

| UDMSE Port | Profile     | Reason                                           |
|------------|-------------|--------------------------------------------------|
| 6          | Trunk (All) | Flex-Mini handles VLAN assignment per port        |
| 7          | Trunk (All) | Flex-Mini handles VLAN assignment per port        |

Alternatively, if all devices on a switch are the same VLAN, you can keep it simple:

| UDMSE Port | Profile     | Reason                                           |
|------------|-------------|--------------------------------------------------|
| 6          | Trusted     | All downstream devices are Trusted (untagged)    |
| 7          | IoT         | All downstream devices are IoT (untagged)        |

> **Choose Trunk if** you might mix VLANs on that switch. **Choose single-VLAN if** you want simplicity and all devices are same VLAN.

#### Step 2.2: Create Port Profiles (if custom needed)

1. Go to **Settings → Profiles → Switch Ports**
2. For the **Servers** profile:
   - **Native Network:** Servers (VLAN 20)
   - **Tagged Networks:** None (untagged only)
3. For the **Trusted** profile:
   - **Native Network:** Trusted (VLAN 10)
   - **Tagged Networks:** None
4. For the **IoT** profile:
   - **Native Network:** IoT (VLAN 30)
   - **Tagged Networks:** None
5. For the **Trunk/All** profile:
   - **Native Network:** Management (VLAN 1)
   - **Tagged Networks:** Trusted (10), Servers (20), IoT (30), Guest (40)

> **Note:** The U7 Pro on Port 1 MUST be a trunk port carrying all VLANs so it can serve multiple SSIDs on different VLANs.

---

### Phase 3: Create Wi-Fi Networks

#### Step 3.1: Create HomeNet (Trusted)

1. Go to **Settings → WiFi**
2. Click **Create New WiFi Network**
3. Configure:
   - **Name/SSID:** `HomeNet`
   - **Password:** (strong 20+ char passphrase)
   - **Network:** `Trusted` (VLAN 10)
   - **Security Protocol:** WPA3 Personal
   - **WiFi Band:** 2.4 GHz + 5 GHz + 6 GHz (if U7 Pro supports)
   - **BSS Transition:** Enable ✓
   - **Hide SSID:** No
   - **Client Device Isolation:** Disable
   - **Multicast Enhancement:** Enable ✓
   - **802.11 DTIM Period:** 3 (better for mobile battery)
4. Click **Create**

#### Step 3.2: Create HomeIoT

1. Click **Create New WiFi Network**
2. Configure:
   - **Name/SSID:** `HomeIoT`
   - **Password:** (strong 16+ char passphrase)
   - **Network:** `IoT` (VLAN 30)
   - **Security Protocol:** WPA2 Personal (compatibility!)
   - **WiFi Band:** 2.4 GHz + 5 GHz
   - **Hide SSID:** Yes (optional, minor security)
   - **Client Device Isolation:** Disable (IoT devices may need to discover each other)
   - **Multicast Enhancement:** Enable ✓
3. Click **Create**

#### Step 3.3: Create HomeGuest

1. Click **Create New WiFi Network**
2. Configure:
   - **Name/SSID:** `HomeGuest`
   - **Password:** (rotate monthly)
   - **Network:** `Guest` (VLAN 40)
   - **Security Protocol:** WPA2/WPA3 Transitional
   - **WiFi Band:** 2.4 GHz + 5 GHz
   - **Hide SSID:** No (guests need to find it)
   - **Client Device Isolation:** Enable ✓ (guests can't see each other)
   - **Guest Policies:** Enable if desired (bandwidth limit, captive portal)
4. Click **Create**

---

### Phase 4: Migrate Devices

> ⚠️ Do this BEFORE enabling firewall rules so you can troubleshoot connectivity.

#### Step 4.1: Migrate Wired Devices

Your cables stay in the same UDMSE ports — you're just changing the port profiles:

| Port | Device                  | Action                                          |
|------|-------------------------|-------------------------------------------------|
| 1    | U7 Pro                  | Change profile to **Trunk (All)**               |
| 3    | Home Assistant          | Change profile to **Servers** (VLAN 20)         |
| 4    | Docker Server           | Change profile to **Servers** (VLAN 20)         |
| 5    | Pi-hole                 | Change profile to **Servers** (VLAN 20)         |
| 6    | Switch (PC + Dock)      | Change profile to **Trusted** (VLAN 10) or Trunk|
| 7    | Switch (TV, PS5, NSwitch)| Change profile to **IoT** (VLAN 30) or Trunk   |
| 8    | Zigbee Hub              | Change profile to **IoT** (VLAN 30)             |

> ⚠️ **Order matters:** Change Port 1 (AP) to Trunk FIRST, then create Wi-Fi networks, then change other ports. If you change Port 1 before the AP has the right VLAN config, you lose wireless management access.

> **If replacing TP-Link switches with USW-Flex-Minis (optional future):** Swap the switches, connect uplinks to Port 6 and 7, wait for adoption in UniFi, then configure per-port profiles as described in Step 2.1c/2.1d.

2. **Set static IPs** (DHCP reservations):
   - Go to **Clients → (device) → Settings → Network**
   - Assign fixed IPs per the table in Section 2
   - Devices will get new IPs from the new VLAN's DHCP range

#### Step 4.2: Migrate Wi-Fi Devices

Reconnect each device to its new SSID:

| Device           | Old SSID  | New SSID  |
|------------------|-----------|-----------|
| iPhones          | (default) | HomeNet   |
| iPads            | (default) | HomeNet   |
| Apple Watches    | (default) | HomeNet   |
| Personal Laptop  | (default) | HomeNet   |
| Work Laptop      | (default) | HomeNet   |
| TVs              | (default) | HomeIoT   |
| Firestick        | (default) | HomeIoT   |
| Echo Dots        | (default) | HomeIoT   |
| Switches         | (default) | HomeIoT   |
| Vacuum           | (default) | HomeIoT   |
| Cat Feeders      | (default) | HomeIoT   |
| AC units         | (default) | HomeIoT   |
| Washer/Dryer     | (default) | HomeIoT   |
| Creality K1C     | (default) | HomeIoT   |
| Kindle           | (default) | HomeIoT   |

> **Tip:** Do this room by room. Smart home devices often need the setup app to change Wi-Fi. Echo Dots require the Alexa app. Cat feeders/vacuum may need their respective apps to reconfigure Wi-Fi.

#### Step 4.3: Verify All Devices Are Online

1. Go to **Clients** in UniFi
2. Confirm each device shows on its correct network/VLAN
3. Test internet access from each VLAN (ping 8.8.8.8)
4. Test DNS resolution (ping google.com)

---

### Phase 5: Configure Firewall Zones & Rules

> This is the most critical phase. The UDMSE with UniFi OS 4.0+ supports **Zone-Based Firewall**.

---

## 6. Firewall Zones & Rules

### Current State (What You Have Now)

Your firewall is currently using the default zone configuration:

| Zone Name  | Networks/Interfaces     | Policy                       |
|------------|-------------------------|------------------------------|
| Interno    | Default                 | Allow All to everything      |
| Externo    | Internet 1, Internet 2  | Allow Return only (inbound)  |
| Gateway    | (system - locked)       | Allow All outbound           |
| VPN        | (system - locked)       | Allow All                    |
| Hotspot    | (none assigned)         | Allow Return / Block inter-zone |
| DMZ        | (none assigned)         | Allow Return / Block inter-zone |

**Current Zone Matrix:**

| Source → Dest | Interno     | Externo         | Gateway         | VPN         | Hotspot     | DMZ         |
|---------------|-------------|-----------------|-----------------|-------------|-------------|-------------|
| Interno       | Permitir Tudo | Permitir Tudo (2) | Permitir Tudo (2) | Permitir Tudo | Permitir Tudo | Permitir Tudo |
| Externo       | Permitir Retorno (3) | Permitir Retorno (3) | Permitir Retorno (5) | Permitir Retorno (3) | Permitir Retorno (3) | Permitir Retorno (3) |
| Gateway       | Permitir Tudo | Permitir Tudo | — | Permitir Tudo | Permitir Tudo | Permitir Tudo |
| VPN           | Permitir Tudo | Permitir Tudo (2) | Permitir Tudo | Permitir Tudo | Permitir Tudo | Permitir Tudo |
| Hotspot       | Permitir Retorno | Permitir Tudo (2) | Permitir Retorno | Permitir Retorno | Bloquear Tudo | Bloquear Tudo |
| DMZ           | Permitir Retorno | Permitir Tudo (2) | Permitir Retorno | Permitir Retorno | Bloquear Tudo | Bloquear Tudo |

**Key observations:**
- You have **2 WAN connections** (Internet 1 + Internet 2) — dual-WAN/failover setup
- All devices are in the single "Interno" zone (flat network, no segmentation)
- The existing (2), (3), (5) numbers are default UniFi policy rule counts — these are system rules you shouldn't touch
- Hotspot and DMZ zones exist but have no networks assigned
- VPN zone is for UniFi's built-in VPN (Teleport, WireGuard site-to-site)

### What Changes

You'll **keep the system zones** (Externo, Gateway, VPN) and **reassign/create zones** for your new VLANs:

| Action           | Zone       | Network Assignment               |
|------------------|------------|----------------------------------|
| **Rename/Reuse** | Interno    | Trusted (VLAN 10) only           |
| **Create New**   | Servidores | Servers (VLAN 20)                |
| **Create New**   | IoT        | IoT (VLAN 30)                    |
| **Rename/Reuse** | Hotspot    | Guest (VLAN 40) — OR create new  |
| Keep as-is       | Externo    | Internet 1, Internet 2           |
| Keep as-is       | Gateway    | (system)                         |
| Keep as-is       | VPN        | (system)                         |
| Keep/Ignore      | DMZ        | (unused, leave empty)            |

> **Note on language:** Your UI is in Portuguese. The guide uses English zone names but the steps are the same. Adjust names as you prefer (e.g., "Servidores" instead of "Servers", "Confiável" instead of "Trusted").

### Zone Definitions

Go to **Settings → Firewall & Security → Zones** (Configurações → Firewall e Segurança → Zonas)

| Zone Name   | Networks Assigned          | Description                    |
|-------------|----------------------------|--------------------------------|
| Interno     | Trusted (VLAN 10)          | Personal devices (reuse existing zone) |
| Servidores  | Servers (VLAN 20)          | Infrastructure servers (CREATE NEW) |
| IoT         | IoT (VLAN 30)              | Smart home & media devices (CREATE NEW) |
| Hotspot     | Guest (VLAN 40)            | Untrusted visitors (reuse existing zone) |
| Externo     | Internet 1, Internet 2     | Internet — dual WAN (keep as-is) |
| Gateway     | UDMSE itself               | Router management (keep as-is) |
| VPN         | (system)                   | Teleport/WireGuard (keep as-is) |
| DMZ         | (none)                     | Unused — leave empty           |

**Steps to configure zones:**

1. **Edit "Interno" zone:**
   - Click on "Interno"
   - Remove "Default" network
   - Add only "Trusted" (VLAN 10) network
   - (After creating VLANs, the "Default" network should be either deleted or used only for Management VLAN 1)

2. **Create "Servidores" zone:**
   - Click "Criar Zona" (Create Zone)
   - Name: `Servidores`
   - Assign network: `Servers` (VLAN 20)

3. **Create "IoT" zone:**
   - Click "Criar Zona"
   - Name: `IoT`
   - Assign network: `IoT` (VLAN 30)

4. **Edit "Hotspot" zone (reuse for Guest):**
   - Click on "Hotspot"
   - Assign network: `Guest` (VLAN 40)
   - The Hotspot zone already has good defaults (Permitir Retorno to internal, Bloquear Tudo inter-zone)
   - Alternatively, create a new "Convidados" zone if you prefer a custom name

> **Why reuse Hotspot for Guest?** The Hotspot zone already has secure defaults: it blocks traffic to other zones and only allows return traffic. This matches exactly what you want for guests — internet-only, no access to internal resources. Saves you from recreating all those policies.

> **Management VLAN 1:** The "Default" network (currently in Interno) should either stay in Interno (if you want management devices to have Trusted-level access) or be left unassigned/in Gateway zone. Since only the UDMSE itself and the U7 Pro are on VLAN 1, this is fine either way.

### Target Zone-to-Zone Policy Matrix (After Configuration)

| Source → Dest | Interno  | Servidores | IoT     | Hotspot | Externo  | Gateway  |
|---------------|----------|------------|---------|---------|----------|----------|
| **Interno**   | Permitir Tudo | Permitir Tudo | Permitir Tudo | Bloquear Tudo | Permitir Tudo | Permitir Tudo |
| **Servidores**| Bloquear Tudo | Permitir Tudo | Permitir Tudo | Bloquear Tudo | Permitir Tudo | Permitir Tudo |
| **IoT**       | Bloquear Tudo | Limited*  | Permitir Tudo | Bloquear Tudo | Permitir Tudo | Bloquear Tudo** |
| **Hotspot**   | Bloquear Tudo | Bloquear Tudo | Bloquear Tudo | Bloquear Tudo | Permitir Tudo | Bloquear Tudo |
| **Externo**   | Permitir Retorno | Permitir Retorno | Permitir Retorno | Permitir Retorno | — | Permitir Retorno |
| **Gateway**   | Permitir Tudo | Permitir Tudo | Permitir Tudo | Permitir Tudo | Permitir Tudo | — |
| **VPN**       | Permitir Tudo | Permitir Tudo | Permitir Tudo | Bloquear Tudo | Permitir Tudo | Permitir Tudo |

\* IoT → Servidores: Only specific ports (Plex 32400, Jellyfin 8096, HA 8123, MQTT 1883, DNS 53)
\** IoT → Gateway: Blocked except DHCP/DNS which UniFi handles automatically

> **Compared to your current setup:** Right now everything is "Permitir Tudo" because all devices are in one "Interno" zone. After segmentation, you'll have controlled access between zones.

### Firewall Rules — Detailed

Go to **Settings → Firewall & Security → Firewall Rules** (Configurações → Firewall e Segurança → Regras de Firewall)

> **Important about your existing rules:** The numbers you see in the current matrix (2), (3), (5) are system-default rules. Do NOT delete those. You'll be adding NEW rules on top of what exists. When you reassign networks to new zones, the default "Permitir Retorno" between External→Internal will automatically apply correctly.

Create rules in this order (rules are evaluated top-to-bottom):

---

#### Rule 1: Allow Established/Related Traffic (AUTO — usually exists)

This rule ensures return traffic for allowed connections works. UniFi typically has this by default.

- **Type:** LAN In
- **Action:** Allow
- **States:** Established, Related
- **Source:** Any
- **Destination:** Any

---

#### Rule 2: Allow Internal → Servers (All)

Trusted devices get full access to all servers.

- **Type:** LAN In
- **Action:** Allow
- **Source:** Network — `Trusted` (192.168.10.0/24)
- **Destination:** Network — `Servers` (192.168.20.0/24)
- **Port:** Any

---

#### Rule 3: Allow Internal → IoT (All)

Trusted devices can manage/access IoT devices (casting, app control).

- **Type:** LAN In
- **Action:** Allow
- **Source:** Network — `Trusted` (192.168.10.0/24)
- **Destination:** Network — `IoT` (192.168.30.0/24)
- **Port:** Any

---

#### Rule 4: Allow IoT → Plex (Docker Server, port 32400)

TVs and Firestick need to stream from Plex.

- **Type:** LAN In
- **Action:** Allow
- **Source:** Network — `IoT` (192.168.30.0/24)
- **Destination:** IP Address — `192.168.20.10` (Docker server)
- **Port:** TCP `32400`

---

#### Rule 5: Allow IoT → Jellyfin (Docker Server, port 8096)

If using Jellyfin instead of/alongside Plex.

- **Type:** LAN In
- **Action:** Allow
- **Source:** Network — `IoT` (192.168.30.0/24)
- **Destination:** IP Address — `192.168.20.10` (Docker server)
- **Port:** TCP `8096`

---

#### Rule 6: Allow IoT → Home Assistant (port 8123)

IoT devices that need to report to HA (some integrations initiate from device).

- **Type:** LAN In
- **Action:** Allow
- **Source:** Network — `IoT` (192.168.30.0/24)
- **Destination:** IP Address — `192.168.20.11` (Home Assistant)
- **Port:** TCP `8123`

---

#### Rule 7: Allow IoT → Home Assistant MQTT (port 1883)

MQTT devices (Zigbee hub via MQTT, sensors) need to reach the MQTT broker on HA.

- **Type:** LAN In
- **Action:** Allow
- **Source:** Network — `IoT` (192.168.30.0/24)
- **Destination:** IP Address — `192.168.20.11` (Home Assistant)
- **Port:** TCP `1883`

---

#### Rule 8: Allow Servers → IoT (All)

Home Assistant needs full access to IoT devices to control them (send commands, poll status).

- **Type:** LAN In
- **Action:** Allow
- **Source:** Network — `Servers` (192.168.20.0/24)
- **Destination:** Network — `IoT` (192.168.30.0/24)
- **Port:** Any

---

#### Rule 9: Allow All VLANs → Pi-hole DNS (Both Instances)

All networks need to resolve DNS via both Pi-holes.

- **Type:** LAN In
- **Action:** Allow
- **Source:** Any (or create a group with Trusted + Servers + IoT)
- **Destination:** IP Group — `Pi-hole-DNS` (`192.168.20.12`, `192.168.20.10`)
- **Port:** TCP/UDP `53`

> Create IP Group `Pi-hole-DNS` in **Settings → Profiles → IP Groups** containing both Pi-hole IPs.

---

#### Rule 10: Allow IoT → Internet (via default route)

IoT devices need internet access (firmware updates, cloud services, voice assistants).

> This is typically allowed by default (LAN → WAN is allow). No explicit rule needed unless you've changed the default WAN-out policy.

---

#### Rule 11: Block IoT → All Private Networks

**CRITICAL** — This blocks IoT from reaching anything NOT explicitly allowed above.

- **Type:** LAN In
- **Action:** Drop
- **Source:** Network — `IoT` (192.168.30.0/24)
- **Destination:** RFC1918 — `192.168.0.0/16, 10.0.0.0/8, 172.16.0.0/12`
- **Port:** Any

> Create an IP Group called `RFC1918` with: `192.168.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12`
> Place this rule AFTER all IoT allow rules (rules 4-7).

---

#### Rule 12: Block Guest → All Private Networks

Guests get internet only.

- **Type:** LAN In
- **Action:** Drop
- **Source:** Network — `Guest` (192.168.40.0/24)
- **Destination:** RFC1918 Group
- **Port:** Any

---

#### Rule 13: Block Servers → Internal

Servers should not initiate connections to personal devices. If a server is compromised, this limits lateral movement.

- **Type:** LAN In
- **Action:** Drop
- **Source:** Network — `Servers` (192.168.20.0/24)
- **Destination:** Network — `Trusted` (192.168.10.0/24)
- **Port:** Any

---

#### Rule 14 (Optional): Block IoT → DNS except Pi-hole

Prevent IoT devices from using hardcoded DNS (bypassing Pi-hole ad blocking).

- **Type:** LAN In
- **Action:** Drop
- **Source:** Network — `IoT` (192.168.30.0/24)
- **Destination:** Any
- **Port:** TCP/UDP `53`
- **Exceptions:** NOT destination `192.168.20.12`

> **Implementation:** Create this as two rules:
> 1. Allow IoT → Pi-hole:53 (Rule 9 above)
> 2. Block IoT → Any:53 (this rule, placed AFTER Rule 9)

---

### Firewall Rules Summary (Order Matters!)

| #  | Action | Source     | Destination              | Port       | Purpose                     |
|----|--------|------------|--------------------------|------------|-----------------------------|
| 1  | Allow  | Any        | Any                      | Any        | Established/Related         |
| 2  | Allow  | Trusted    | Servers                  | Any        | Full server access          |
| 3  | Allow  | Trusted    | IoT                      | Any        | Manage IoT devices          |
| 4  | Allow  | IoT        | 192.168.20.10            | TCP 32400  | Plex streaming              |
| 5  | Allow  | IoT        | 192.168.20.10            | TCP 8096   | Jellyfin streaming          |
| 6  | Allow  | IoT        | 192.168.20.11            | TCP 8123   | HA web interface            |
| 7  | Allow  | IoT        | 192.168.20.11            | TCP 1883   | MQTT broker                 |
| 8  | Allow  | Servers    | IoT                      | Any        | HA controls devices         |
| 9  | Allow  | Any LAN    | Pi-hole-DNS group        | 53         | Pi-hole DNS (both)          |
| 10 | Drop   | IoT        | Any:53 (not Pi-hole)     | 53         | Force Pi-hole DNS           |
| 11 | Drop   | IoT        | RFC1918                  | Any        | Block IoT→private           |
| 12 | Drop   | Guest      | RFC1918                  | Any        | Block Guest→private         |
| 13 | Drop   | Servers    | Trusted                  | Any        | Block server→personal       |

---

### mDNS / Service Discovery

For casting (AirPlay, Chromecast) from Trusted → IoT to work:

1. Go to **Settings → Networks → (each network) → Advanced**
2. Enable **Multicast DNS (mDNS)** on: Trusted, Servers, IoT
3. The UDMSE will act as an mDNS reflector across VLANs

This allows:
- iPhones (Trusted) to discover and cast to TVs (IoT)
- Trusted devices to see Plex server (Servers)
- AirPlay from Trusted to Echo Dots (IoT)

---

## 7. DNS Configuration (Dual Pi-hole)

> ### ⚠️ OPTIONAL — Implement When Ready
>
> This section makes Pi-hole serve DNS for your ENTIRE network. Currently Pi-hole is only used by your personal PC. You can implement VLANs/firewall first and come back to this later.
>
> **For now:** Leave each VLAN's DNS set to `1.1.1.1` and `8.8.8.8` (or the UDMSE gateway). When you're ready to enable network-wide ad blocking, follow these steps.

### Architecture: Primary + Secondary Pi-hole

You have **two Pi-hole instances** for redundancy:

| Instance        | Location              | IP Address     | Role            |
|-----------------|-----------------------|----------------|------------------|
| Pi-hole #1      | Standalone device     | 192.168.20.12  | Primary DNS      |
| Pi-hole #2      | Docker container      | 192.168.20.10  | Secondary DNS    |

> **Why this matters:** If the Docker server goes down for maintenance, Pi-hole #1 still serves DNS. If the standalone Pi-hole dies, Pi-hole #2 on Docker handles it. UniFi DHCP gives clients BOTH IPs.

> **Current state:** You said Pi-hole is only configured for your personal PC. The steps below will make it serve the ENTIRE network across all VLANs.

### Step 7.1: Configure Pi-hole #1 (Standalone Device)

1. **Set static IP:**
   - IP: `192.168.20.12`
   - Subnet: `255.255.255.0`
   - Gateway: `192.168.20.1`
   - DNS: `1.1.1.1` (upstream, NOT itself)

2. **Allow queries from ALL subnets:**
   - SSH into the Pi-hole or use the web admin
   - Edit `/etc/pihole/setupVars.conf`:
     ```
     PIHOLE_INTERFACE=eth0
     DNSMASQ_LISTENING=all
     ```
   - OR in web UI: **Settings → DNS → Interface settings → Permit all origins**
   - This is critical! Without this, Pi-hole will reject queries from 192.168.10.x, 192.168.30.x, etc.

3. **Restart Pi-hole DNS:**
   ```bash
   pihole restartdns
   ```

4. **Add conditional forwarding:**
   - **Settings → DNS → Conditional Forwarding** (or edit `/etc/dnsmasq.d/`):
   - Enable conditional forwarding
   - Local network CIDR: `192.168.0.0/16`
   - IP of DHCP server (router): `192.168.20.1`
   - Local domain (optional): `local`
   - This lets Pi-hole resolve local hostnames (e.g., `desktop-pc.local`)

5. **Verify upstream DNS servers:**
   - **Settings → DNS → Upstream DNS Servers**
   - Recommended: `1.1.1.1` and `8.8.8.8` (or your preferred upstream)
   - Do NOT point upstream to itself or to Pi-hole #2

### Step 7.2: Configure Pi-hole #2 (Docker Container)

Your `docker-compose.mine.yml` already has a Pi-hole container. Update it:

1. **Ensure the container listens on the host IP (192.168.20.10):**
   - Your current port mapping `"53:53/tcp"` and `"53:53/udp"` binds to all interfaces — this is correct
   - Pi-hole #2 will be reachable at `192.168.20.10:53`

2. **Update Pi-hole container environment:**
   ```yaml
   environment:
     - TZ=${TIMEZONE}
     - WEBPASSWORD=${PIE_HOLE_PASSWORD}
     - DNSMASQ_LISTENING=all    # Accept queries from all subnets
     - PIHOLE_DNS_=1.1.1.1;8.8.8.8  # Upstream DNS
   ```

3. **Add conditional forwarding inside the container:**
   Create/mount a custom dnsmasq config:
   ```bash
   # Create file: pihole/etc/dnsmasq.d/05-conditional.conf
   server=/local/192.168.20.1
   rev-server=192.168.0.0/16,192.168.20.1
   ```

4. **Sync both Pi-holes (nebula-sync):**
   Update your `nebula-sync` service with the new IPs:
   ```yaml
   nebula-sync:
     environment:
       - PRIMARY=http://192.168.20.12|${PIE_HOLE_PASSWORD}
       - REPLICAS=http://192.168.20.10:81|${PIE_HOLE_PASSWORD}
       - FULL_SYNC=true
       - RUN_GRAVITY=true
       - CRON=0 * * * *
   ```
   This keeps blocklists, whitelists, and settings in sync between both instances.

### Step 7.3: Configure UniFi DHCP to Use Both Pi-holes

Go to **Settings → Networks** and update DNS on each network:

| Network    | DNS Server 1       | DNS Server 2       | Notes                         |
|------------|--------------------|--------------------|-------------------------------|
| Trusted    | `192.168.20.12`    | `192.168.20.10`    | Both Pi-holes                 |
| Servers    | `192.168.20.12`    | `192.168.20.10`    | Both Pi-holes                 |
| IoT        | `192.168.20.12`    | `192.168.20.10`    | Both Pi-holes                 |
| Guest      | `1.1.1.1`          | `8.8.8.8`          | Public DNS (no dependency)    |

**How to set this in UniFi:**
1. Go to **Settings → Networks → (select network, e.g., Trusted)**
2. Expand **DHCP** section
3. Under **DHCP DNS Server**, select **Manual**
4. Enter: `192.168.20.12` and `192.168.20.10`
5. Repeat for Servers and IoT networks
6. For Guest: enter `1.1.1.1` and `8.8.8.8`

> **Important:** After changing DHCP settings, existing clients keep their old DNS until their DHCP lease renews. Force renewal by:
> - Disconnecting/reconnecting Wi-Fi
> - Running `ipconfig /release && ipconfig /renew` (Windows)
> - Running `sudo dhclient -r && sudo dhclient` (Linux)
> - Toggle Wi-Fi off/on (phones)

### Step 7.4: Remove PC-Only DNS Configuration

Since Pi-hole is currently only configured for your personal PC:

1. **On your Desktop PC:**
   - Remove any manually set DNS (set back to "Automatic" / DHCP)
   - The PC will now get `192.168.20.12` + `192.168.20.10` from DHCP

2. **On your router (current state):**
   - If you set the Default network's DNS to point to Pi-hole for just the PC, this will be replaced by per-VLAN DNS settings above

### Step 7.5: Verify DNS Is Working Network-Wide

After configuration, test from each VLAN:

```bash
# From a Trusted device:
nslookup google.com 192.168.20.12   # Should resolve (Pi-hole #1)
nslookup google.com 192.168.20.10   # Should resolve (Pi-hole #2)

# From an IoT device (e.g., SSH to a Pi or use network tools app):
nslookup google.com                  # Should use Pi-hole via DHCP
nslookup doubleclick.net              # Should be blocked (0.0.0.0)
```

**Check Pi-hole dashboards:**
- Pi-hole #1 (http://192.168.20.12/admin): Should show queries from all VLANs
- Pi-hole #2 (http://192.168.20.10:81/admin): Should show queries from all VLANs
- Both should show clients from 192.168.10.x, 192.168.20.x, 192.168.30.x

### Pi-hole Firewall Consideration

Since both Pi-holes are in the Servers VLAN, all other VLANs need to reach them on port 53.

Update **Firewall Rule 9** to allow traffic to BOTH:

- **Rule 9:** Allow All VLANs → Pi-hole DNS
  - **Source:** Any LAN (Trusted + Servers + IoT)
  - **Destination:** IP Group `Pi-hole-DNS` containing `192.168.20.12` and `192.168.20.10`
  - **Port:** TCP/UDP `53`

> Create an IP Group called `Pi-hole-DNS` in **Settings → Profiles → IP Groups** with:
> - `192.168.20.12`
> - `192.168.20.10`

---

## 8. Cloudflare Tunnel Considerations

> ### ⚠️ OPTIONAL — Future Enhancement
>
> This section covers adding Cloudflare Tunnel external access to the Docker server. Home Assistant already has external access via tunnel. Extending this to other Docker services (Plex, Immich, etc.) can be done at any time without changing the VLAN setup.

### Current Setup

Your docker-compose.mine.yml shows:
- `cloudflared` container running with `TUNNEL_TOKEN`
- External access routed through Traefik's `tunnel` entrypoint (port 8888)
- Google OAuth (`traefik-forward-auth`) protecting external routes
- Services get external access via `*-external` router labels

### What Changes With VLANs

**Nothing breaks.** Here's why:

1. **Cloudflare Tunnel is outbound-only:** The `cloudflared` container initiates an outbound connection to Cloudflare's edge. No inbound ports needed on your router.

2. **Docker networking is internal:** Your Docker services communicate via the `docker-compose-server` bridge network. VLAN changes don't affect container-to-container traffic.

3. **Only requirement:** The Docker server (192.168.20.10) needs outbound internet access. The Servers VLAN allows this by default (LAN → WAN is allowed).

### After VLAN Migration Checklist

1. **Update Docker server's IP references:**
   - If any `.env` file references the old IP (e.g., `192.168.1.x`), update to `192.168.20.10`
   - Check `NET_LOCAL` in the VPN service: change to include all your subnets if needed:
     ```
     NET_LOCAL=192.168.10.0/24,192.168.20.0/24,192.168.30.0/24
     ```

2. **Update Home Assistant configuration:**
   - If HA uses IP-based integrations (not hostnames), update target IPs
   - Update `trusted_proxies` in HA's `configuration.yaml` if you use reverse proxy:
     ```yaml
     http:
       use_x_forwarded_for: true
       trusted_proxies:
         - 192.168.20.0/24  # Docker network / Traefik
         - 172.17.0.0/16    # Docker bridge
     ```

3. **Cloudflare Tunnel DNS records:**
   - No changes needed — tunnel points to internal Docker service names
   - If tunnel config uses IPs, update to `192.168.20.10`

4. **Update `nebula-sync` Pi-hole IPs:**
   - Your compose has `PRIMARY=http://192.168.0.2` — update to new Pi-hole IP
   - Update `REPLICAS` if the secondary Pi-hole also moved

### Home Assistant External Access

Your HA Cloudflare Tunnel will continue to work because:
- HA is on Servers VLAN → has internet access
- Cloudflared (on Docker server) connects outbound to Cloudflare
- External traffic: User → Cloudflare → Tunnel → Traefik → HA container

If HA is NOT on the Docker server (separate device):
- HA still needs internet access (Servers VLAN provides this)
- You'd run cloudflared on HA itself, OR point the tunnel to HA's new IP (192.168.20.11)

---

## 9. Validation & Testing

### Test Plan (Execute After Full Setup)

#### Test 1: Basic Connectivity

| From          | To              | Test                          | Expected       |
|---------------|-----------------|-------------------------------|----------------|
| iPhone/iPad   | google.com      | `ping google.com`             | ✓ Success      |
| iPhone/iPad   | Docker server   | Browse to Plex                | ✓ Success      |
| iPhone/iPad   | TV              | AirPlay/Cast                  | ✓ Success      |
| TV            | Plex            | Open Plex app                 | ✓ Success      |
| TV            | Desktop PC      | Try to access                 | ✗ Blocked      |
| Echo Dot      | Internet        | "Alexa, what time is it?"     | ✓ Success      |
| Echo Dot      | Desktop PC      | —                             | ✗ Blocked      |
| Guest phone   | google.com      | Browse web                    | ✓ Success      |
| Guest phone   | Plex            | Try to browse to Plex IP      | ✗ Blocked      |
| Guest phone   | Other guest     | Try to ping                   | ✗ Blocked      |

#### Test 2: DNS Resolution

| From          | DNS Query       | Expected                       |
|---------------|-----------------|--------------------------------|
| Trusted       | google.com      | Resolves via Pi-hole           |
| IoT           | google.com      | Resolves via Pi-hole           |
| Guest         | google.com      | Resolves via 1.1.1.1           |
| IoT           | (blocked domain)| Pi-hole blocks it              |

#### Test 3: Firewall Verification

From a device on each VLAN, try to access services that should be blocked:

```bash
# From IoT device (TV/Switch), try to reach trusted network:
ping 192.168.10.100  # Should timeout/fail

# From IoT device, try to reach Plex:
curl http://192.168.20.10:32400/web  # Should work

# From Guest device, try to reach any internal:
ping 192.168.20.10  # Should timeout/fail
ping 192.168.10.1   # Should timeout/fail
```

#### Test 4: Cloudflare Tunnel

1. From your phone on mobile data (not Wi-Fi):
   - Access your external Plex URL → Should work (OAuth prompt)
   - Access your external HA URL → Should work
2. Check `cloudflared` container logs:
   ```bash
   docker logs cloudflared
   ```
   Should show active tunnel connection, no errors.

#### Test 5: Home Assistant IoT Control

1. Open HA dashboard
2. Toggle a smart device (light, AC, feeder)
3. Verify it responds
4. Check HA logs for connectivity errors to any integration

---

## 10. Future Expansion Notes

### Adding More Devices

- **New server:** Plug into Port 7 (Servers profile) or add to the switch on Port 2
- **New IoT device:** Connect to HomeIoT SSID, optionally assign static IP
- **New trusted device:** Connect to HomeNet SSID

### Potential Future VLANs

| VLAN ID | Name       | Subnet            | Use Case                              |
|---------|------------|-------------------|---------------------------------------|
| 50      | Work       | 192.168.50.0/24   | Work laptop isolation (if required)   |
| 60      | Cameras    | 192.168.60.0/24   | Security cameras (no internet!)       |
| 70      | Lab        | 192.168.70.0/24   | Testing/development sandbox           |

### Security Cameras VLAN (Recommended Future Add)

> See [Optional: Ceiling Camera Setup](#optional-ceiling-camera-setup) below for the full implementation guide.

If you add cameras:
- VLAN 60, block ALL internet access (cameras don't need cloud)
- Allow only Servers → Cameras (NVR access)
- Allow Trusted → Cameras (live view)
- Block Cameras → everything except NVR

### Ceiling Camera Setup (Port 2 — Up to 4 Cameras via Single Cable)

> ### ⚠️ OPTIONAL — Implement When Ready {#optional-ceiling-camera-setup}
>
> You already have the Ethernet cable run to the ceiling. This section can be implemented independently at any time. The VLAN/firewall core setup works without cameras.

You have a single Ethernet run from UDMSE Port 2 into the ceiling. Here's how to support up to 4 cameras with that one cable:

**Solution: PoE switch in the ceiling**

Install a small PoE switch at the end of the ceiling cable. The switch powers the cameras via PoE AND provides network connectivity — no additional power cables needed for the cameras.

**Recommended ceiling switch:**

| Model               | Ports | PoE Budget | Price  | Why                                           |
|---------------------|-------|------------|--------|-----------------------------------------------|
| **USW-Flex**        | 5     | 4× PoE (46W)| ~$99  | **Best pick** — powered via PoE-in, compact, outdoor-rated |
| USW-Ultra           | 8     | 4× PoE+ (42W)| ~$109| More ports but needs AC power                 |
| USW-Flex-Mini       | 5     | No PoE out  | ~$29  | Only if cameras have own power (rare)         |

**Why USW-Flex is the best choice for ceiling mounting:**

1. **Powered via PoE-in on Port 1** — the single cable from UDMSE Port 2 powers the switch itself (no need for a power outlet in the ceiling!)
2. **4× PoE-out ports** — powers up to 4 cameras directly
3. **Weatherproof/outdoor-rated** — handles ceiling dust/temperature variations
4. **Compact** — easily hidden above ceiling tiles
5. **Managed** — per-port VLAN, visible in UniFi dashboard

**Important: UDMSE Port 2 must provide PoE to power the USW-Flex.**
The UDMSE SE has PoE++ on ports 1-2 (up to 60W per port). This is more than enough to power the USW-Flex + 4 cameras.

**Wiring diagram:**

```
┌─────────────────────┐         Single Ethernet cable          ┌──────────────────┐
│   Dream Machine SE  │         (PoE power + data)             │  USW-Flex        │
│   Port 2 (PoE++)    │─────────────── ceiling ───────────────→│  (in ceiling)    │
│   Profile: Cameras  │                                        │                  │
│   VLAN 60 (untagged)│                                        ├──────────────────┤
└─────────────────────┘                                        │ Port 1: Uplink   │ ← PoE-in powers the switch
                                                               │ Port 2: Camera 1 │ → PoE out
                                                               │ Port 3: Camera 2 │ → PoE out
                                                               │ Port 4: Camera 3 │ → PoE out
                                                               │ Port 5: Camera 4 │ → PoE out
                                                               └──────────────────┘
```

**Configuration steps (when you're ready to add cameras):**

1. **Create Cameras network (VLAN 60):**
   - **Settings → Networks → Create New Network**
   - Name: `Cameras`
   - VLAN ID: `60`
   - Gateway: `192.168.60.1/24`
   - DHCP Range: `192.168.60.100–199`
   - DNS: None needed (cameras won't use internet)
   - mDNS: Disable

2. **Set UDMSE Port 2 profile:**
   - Profile: **Cameras** (VLAN 60 untagged)
   - PoE: **Enable** (this powers the USW-Flex through the cable)

3. **Install USW-Flex in ceiling:**
   - Connect the ceiling cable to USW-Flex Port 1 (uplink/PoE-in)
   - The switch auto-adopts in UniFi within ~60 seconds
   - All other ports default to the same VLAN (Cameras)

4. **Connect cameras:**
   - Run short Ethernet cables from USW-Flex to each camera position
   - Cameras get powered via PoE from the switch
   - Cameras get IP from 192.168.60.x DHCP

5. **Add firewall rules for Cameras VLAN:**

   | # | Action | Source    | Destination | Port | Purpose                     |
   |---|--------|-----------|-------------|------|-----------------------------|
   | A | Allow  | Servers   | Cameras     | Any  | NVR/Frigate records footage |
   | B | Allow  | Trusted   | Cameras     | Any  | Live view from phone/PC     |
   | C | Drop   | Cameras   | RFC1918     | Any  | Block all camera→LAN        |
   | D | Drop   | Cameras   | Any         | Any  | Block camera→internet       |

   > **Why block internet for cameras?** Many cheap cameras phone home to Chinese servers. Blocking internet prevents data exfiltration while still allowing local recording/viewing.

6. **NVR setup (optional — Frigate on Docker server):**
   - Run Frigate container on your Docker server
   - Docker server (Servers VLAN 20) reaches cameras (VLAN 60) via Rule A above
   - Add a firewall rule: Allow Servers → Cameras
   - Frigate accesses camera RTSP streams at `192.168.60.x:554`

**Camera recommendations for PoE (compatible with USW-Flex):**

| Camera             | Resolution | PoE Draw | Price   | Notes                          |
|--------------------|-----------|----------|---------|--------------------------------|
| Reolink RLC-810A   | 4K        | ~10W     | ~$55    | Good value, local recording    |
| Reolink RLC-520A   | 5MP       | ~7W      | ~$40    | Budget pick                    |
| TP-Link Tapo C325WB| 4MP       | ~9W      | ~$50    | Works without cloud            |
| Amcrest IP8M-2496E | 4K        | ~12W     | ~$80    | Great with Frigate/HA          |

> **Power budget check:** USW-Flex provides 46W total PoE. 4 cameras × ~12W max = 48W. This is tight at max draw. In practice, cameras draw 7-10W each, so 4 cameras (28-40W) fits within budget. If you pick lower-power cameras (RLC-520A at 7W), you'll have plenty of headroom.

### Adding a Managed Switch / Upgrading Switches {#optional-switch-replacement}

> ### ⚠️ OPTIONAL — Your TP-Link Switches Work Fine for VLANs
>
> Since all devices on each TP-Link switch are the same VLAN (Port 6 = all Trusted, Port 7 = all IoT), the unmanaged switches work correctly. Replace only when you want per-port control, visibility, or need to mix VLANs on the same switch.

You already have 2 switches that could be replaced (TP-Link → UniFi). If you outgrow those:
1. **USW-Flex-Mini** (5 ports, ~$29) — great for desk/entertainment areas
2. **USW-Ultra** (8 ports, 4× PoE+, ~$109) — if you add PoE devices (cameras, additional APs)
3. **USW-Enterprise-8-PoE** (8 ports, 2.5GbE, ~$179) — if you want multi-gig to your server

Installation:
1. Connect uplink to a Trunk port on UDMSE
2. The switch auto-adopts into UniFi Network
3. Assign port profiles per device on the switch via the UniFi dashboard

### Adding Another Access Point

1. Connect new AP to a Trunk port
2. It auto-adopts and inherits all SSID/VLAN configurations
3. Enable seamless roaming between APs

---

### OPTIONAL: NAS Storage {#optional-nas-storage}

> ### ⚠️ OPTIONAL — Add When Ready
>
> A NAS provides centralized storage for media, backups, Docker volumes, and surveillance footage. It fits perfectly in the Servers VLAN.

#### Recommended NAS Options

| Brand/Model            | Bays | CPU               | RAM     | Price Range | Best For                            |
|------------------------|------|-------------------|---------|-------------|-------------------------------------|
| **UGREEN NASync DXP4800 Plus** | 4 | Intel N100  | 8GB     | ~$400-450   | Best value, Docker support, modern  |
| UGREEN NASync DXP2800  | 2    | Intel N100        | 8GB     | ~$280-320   | Budget pick if 2 bays is enough     |
| Synology DS423+        | 4    | Intel Celeron J4125| 2GB    | ~$500       | Best software/ecosystem, proven     |
| Synology DS223+        | 2    | Intel Celeron J4125| 2GB    | ~$300       | 2-bay Synology                      |
| QNAP TS-464           | 4    | Intel N5095        | 8GB    | ~$450       | Powerful, 2.5GbE, good for Docker   |
| Ubiquiti UNAS Pro      | 4    | (TBD)             | (TBD)  | ~$500?      | UniFi ecosystem (if/when released)  |
| DIY (TrueNAS)         | Any  | Your choice       | 16GB+  | Varies      | Maximum flexibility, ECC RAM        |

#### NAS Placement in Your Network

| Aspect          | Configuration                                          |
|-----------------|--------------------------------------------------------|
| VLAN            | **Servers (VLAN 20)** — same as Docker server          |
| IP              | `192.168.20.13` (static reservation)                   |
| Connection      | Ethernet to UDMSE Port 7 (Servers profile)             |
| Access from     | Trusted (full), Servers (full), IoT (block)            |

#### Use Cases for Your Setup

1. **Media storage for Plex/Jellyfin:**
   - Mount NAS shares via NFS/SMB on Docker server
   - `${DATA_ROOT}` in your docker-compose points to NAS
   - Plex/Sonarr/Radarr read/write media directly from NAS

2. **Docker volume backups:**
   - Automated backup of Docker configs to NAS
   - Versioned snapshots (Synology/UGREEN support this natively)

3. **Surveillance footage (if cameras added):**
   - Frigate NVR on Docker server → stores recordings on NAS
   - Or run Frigate directly on the NAS (if it supports Docker)

4. **Time Machine / PC backups:**
   - Trusted devices back up to NAS via SMB/AFP
   - Keeps backups off your Docker server's disks

5. **Immich photo storage:**
   - Point Immich's storage to NAS mount
   - Photos safe on redundant (RAID) storage

#### NAS Firewall Rules

No additional firewall rules needed — NAS is in Servers VLAN:
- ✓ Trusted → Servers already allowed (Rule 2)
- ✓ IoT → Servers already blocked (Rule 11)
- ✓ Guest → Servers already blocked (Rule 12)

#### My Recommendation

**UGREEN NASync DXP4800 Plus** ($400-450):
- Intel N100 is very power-efficient (great for 24/7 NAS)
- 4 bays = start with 2 drives in RAID 1, expand later
- Native Docker support (could even run some containers on it)
- 2.5GbE networking
- Much newer hardware than similarly-priced Synology

**If you want best software/support:** Synology DS423+ — the DSM software is unmatched for ease of use, but hardware is older and more expensive.

**If Ubiquiti releases UNAS Pro:** Wait and evaluate — tight UniFi ecosystem integration could be compelling, but it's unproven.

**If you want maximum control:** DIY TrueNAS box — most flexible, best for ZFS, but requires more setup and maintenance.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│                    NETWORK QUICK REF                      │
├─────────────────────────────────────────────────────────┤
│ Management : VLAN 1  │ 192.168.1.0/24  │ Infra only    │
│ Trusted    : VLAN 10 │ 192.168.10.0/24 │ Personal      │
│ Servers    : VLAN 20 │ 192.168.20.0/24 │ Docker/HA/DNS │
│ IoT        : VLAN 30 │ 192.168.30.0/24 │ Smart home    │
│ Guest      : VLAN 40 │ 192.168.40.0/24 │ Visitors      │
├─────────────────────────────────────────────────────────┤
│ Wi-Fi: HomeNet (WPA3) │ HomeIoT (WPA2) │ HomeGuest     │
├─────────────────────────────────────────────────────────┤
│ Docker Server : 192.168.20.10 (+ Pi-hole #2 container) │
│ Home Assistant: 192.168.20.11                           │
│ Pi-hole #1    : 192.168.20.12 (standalone)              │
├─────────────────────────────────────────────────────────┤
│ DNS (all VLANs): 192.168.20.12 + 192.168.20.10         │
│ Guest DNS      : 1.1.1.1 / 8.8.8.8                     │
└─────────────────────────────────────────────────────────┘
```

---

## Troubleshooting

| Symptom                              | Likely Cause                        | Fix                                        |
|--------------------------------------|-------------------------------------|--------------------------------------------|
| IoT device can't reach internet      | Firewall rule order wrong           | Ensure RFC1918 block is AFTER allow rules  |
| AirPlay/Cast not discovering         | mDNS not enabled                    | Enable mDNS on both Trusted and IoT VLANs  |
| Pi-hole not resolving for IoT        | Firewall blocking DNS               | Verify Rule 9 (Allow → Pi-hole-DNS group:53)|
| TV can't play Plex                   | Wrong Plex port or server IP        | Check Rule 4, verify server IP is correct  |
| Cloudflare Tunnel down after move    | Docker server lost internet         | Check server gateway (192.168.20.1)        |
| HA can't control IoT devices         | Servers→IoT rule missing            | Verify Rule 8 exists                       |
| Guest can't get DHCP                 | VLAN not tagged on AP uplink        | Verify trunk port carries VLAN 40          |
| Echo Dot offline after Wi-Fi change  | Re-setup needed                     | Use Alexa app to reconnect to HomeIoT      |
| Work VPN broken                      | VPN needs specific ports/protocols  | Add allow rule for work VPN destination    |

---

## Implementation Order Summary

### Phase A: Core (Do Now)

1. ✅ Backup UniFi configuration
2. ✅ Create all VLANs/Networks (Phase 1)
3. ✅ Configure port profiles (Phase 2) — keep TP-Link switches as-is
4. ✅ Create Wi-Fi networks (Phase 3)
5. ✅ Migrate wired devices (change port profiles, no cable moves needed)
6. ✅ Migrate wireless devices to new SSIDs
7. ✅ Verify connectivity WITHOUT firewall rules
8. ✅ Create firewall rules (Phase 5) — one at a time, test after each
9. ✅ Full validation (Section 9)
10. ✅ Take a new backup once everything works

### Phase B: Optional Enhancements (Do Later, Any Order)

| Enhancement                    | Dependency              | Difficulty | Section     |
|-------------------------------|-------------------------|------------|-------------|
| Pi-hole as network-wide DNS   | Core setup complete     | Easy       | Section 7   |
| Docker server external access | Core setup + tunnel     | Medium     | Section 8   |
| Replace TP-Link switches      | None (works with core)  | Easy       | Section 10  |
| Ceiling cameras               | USW-Flex purchase       | Medium     | Section 10  |
| NAS storage                   | None (works with core)  | Medium     | Section 10  |

> Each optional enhancement is independent — do them in any order, at any time.
