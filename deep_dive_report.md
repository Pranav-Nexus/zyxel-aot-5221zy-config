# Technical Deep-Dive Report: Zyxel AOT-5221ZY Router

This report presents a comprehensive technical deep-dive into the configuration settings, remote management protocols, VoIP systems, Wi-Fi layout, and DNS routing mechanics of the **Zyxel AOT-5221ZY** router (customized for Bharti Airtel).

> [!IMPORTANT]
> All sensitive, personal, and system credentials have been masked for security purposes before committing this report.

---

## 1. Device & Vendor Overview
- **Device Model**: Zyxel AOT-5221ZY (Wi-Fi 6 Dual-Band GPON ONT Router).
- **Firmware Customizer**: Customized by **Embedur Systems** (a networking firmware partner for Indian ISPs). This is indicated by specific firewall rule tags like `EMBEDUR_TCP`.
- **Default Credentials**: 
  - Admin login: `admin` / `[MASKED_ROUTER_PASSWORD]` (Stored in `LoginPrivilegeMgmt`).

---

## 2. Remote Management & Telemetry
Airtel maintains extensive, real-time control over this router using two concurrent management protocols:

### A. TR-069 (CWMP)
- **ACS URL**: `http://tms.airtelbroadband.in:8103` (Airtel's Auto-Configuration Server).
- **ACS Credentials**: `airtelacs` / `airtelacs`.
- **Periodic Inform**: Enabled (`PeriodicInformEnable = 1`), configured to check in with Airtel every 24 hours (`86400` seconds).
- **STUN Server**: `tms.airtelbroadband.in:8104` (used to maintain connection states through NAT).
- **Inbound Access**: Firewall rules explicitly permit TCP port `7547` (`Default_TR069`) from the WAN interface to receive Connection Requests from Airtel.
- **Security Note**: `CheckCertificateCN = 0` (The router does not validate the Common Name (CN) of SSL certificates when establishing remote connections, which is vulnerable to MITM interception).

### B. TR-369 (User Services Platform / USP) via MQTT
The router runs a modern USP client that maintains a persistent connection to Airtel's **Corteca Smart Home Management Platform**:
- **MQTT Broker**: `verne-shard3.corteca.airtelbroadband.in` (Port `443` over TLS).
- **MQTT Credentials**: Client ID and Username are set to the router's WAN MAC address (`XX:XX:XX:XX:XX:XX`), password: `[MASKED]`.
- **MQTT Topic Subscription**: `devices/usp/[MASKED_MAC_ADDRESS]/request/#` (allows Airtel's backend to push instant real-time telemetry queries and commands to the router).
- **Inbound Access**: Firewall rules explicitly permit TCP port `8883` (`EMBEDUR_TCP`) from the WAN for MQTT TLS traffic.

---

## 3. Wi-Fi (WLAN) Configuration
The Wi-Fi 6 subsystem has **8 separate SSID slots** split across 2.4GHz and 5GHz bands:
- **SSID 1 (2.4GHz)**: Active. SSID name: `Airtel_Home_WiFi` (User Defined). Security: WPA/WPA2/WPA3 (Pre-shared key: `[MASKED]`).
- **SSID 5 (5GHz)**: Active. SSID name: `Airtel_Home_WiFi` (User Defined). Security: WPA/WPA2/WPA3 (Pre-shared key: `[MASKED]`).
- **SSIDs 2, 3, 4 (2.4GHz)**: Disabled. Default names like `Airtel_Zerotouch_2`, `3`, `4`.
- **SSIDs 6, 7 (5GHz)**: Disabled. Default names like `Airtel_Zerotouch_5G_2`, `3`.
- **SSID 8 (Mesh/Backhaul)**: Active. SSID name: `[MESH_BACKHAUL_SSID]`. Default system passphrase: `[MASKED]` (used for EasyMesh or ISP provisioning).

---

## 4. Voice over IP (VoIP) SIP Credentials
Airtel routes fixed-line landline voice service over a dedicated virtual network interface:
- **VLAN/QoS tagging**: VoIP traffic is isolated on VLAN `660` (interface `nas0_660`).
- **SIP Outbound Proxy**: `10.5.109.197` (with standby proxy `10.5.109.213`).
- **SIP Registrar & Domain**: `mu.ims.airtel.in`.
- **SIP User URI**: `+91[MASKED_PHONE_NUMBER]@mu.ims.airtel.in` (the landline directory number).
- **SIP Authentication**: 
  - Username: `+91[MASKED_PHONE_NUMBER]@mu.ims.airtel.in`
  - Password: `[MASKED_SIP_PASSWORD]` (stored in plaintext).

---

## 5. DNS Configuration Deep Dive & Diff
To understand why IPv6 DNS queries resolve correctly to Cloudflare while IPv4 DNS queries route to Airtel, we performed a diff of modified settings against the factory defaults:

| Parameter Path | Factory Default | Current (Working) Config | Role in DNS Routing |
| :--- | :--- | :--- | :--- |
| `LANDevice.i1.LANHostConfigManagement.X_5067F0_DNSRelayEnable` | `<absent>` (Disabled) | `0` (Disabled) | When `1`, the router acts as a DNS Proxy and tells DHCP clients to use its own address (`192.168.1.1`) as the IPv4 DNS server. |
| `WANDevice.i1.WANConnectionDevice.i1.WANPPPConnection.i1.X_5067F0_DNSOverride` | `<absent>` (Disabled) | `1` (Enabled) | When `1`, the router's DNS proxy overrides the ISP-assigned DNS servers with the user-defined `DNSServers` tag. |
| `WANDevice.i1.WANConnectionDevice.i1.WANPPPConnection.i1.DNSOverrideAllowed` | `<absent>` (Disabled) | `1` (Enabled) | Complements the override setting. When `1`, it allows overriding the ISP DNS. |
| `LANDevice.i1.X_5067F0_IPv6LANHostConfigManagement.IPv6DNSWanConnection` | `WANPPPConnection.1` | `""` (Empty) | Disables fetching DNS dynamically from the WAN for IPv6. |
| `StatefullDHCPv6.DNSServerType` | `1` (Dynamic) | `2` (UserDefined) | Forces the DHCPv6 server to lease custom IPv6 addresses directly to clients. |
| `LANHostConfigManagement.X_5067F0_DNS.i1.ServerTypeSelect` | `DNSProxy` | `UserDefined` | Forces the DHCP server to distribute user-defined DNS IPs instead of the router proxy IP. |
| `LANHostConfigManagement.X_5067F0_DNS.i1.X_5067F0_Enable` | `<absent>` (Disabled) | `1` (Enabled) | Activates the custom DNS selection configuration node on the LAN interface. |

### How IPv6 is configured:
The router has `DNSServerType` set to `2` (UserDefined) and `IPv6DNSWanConnection` set to empty. This bypasses WAN-assigned IPv6 DNS entirely and forces the router to distribute Cloudflare's IPv6 addresses directly to DHCPv6 client interfaces.

### How IPv4 is configured:
By setting `DNSOverrideAllowed` to `1` and `X_5067F0_DNSOverride` to `1` on the active PPPoE interface, the router overrides Airtel-negotiated DNS servers and resolves all forwarded DNS queries through Cloudflare (`1.1.1.1` / `1.0.0.1`). Setting `ServerTypeSelect` to `UserDefined` and enabling the setting node ensures the DHCP service distributes custom IPs, though the WAN override resolves any queries forwarded through `192.168.1.1`.

---

## 6. Security Vulnerabilities Identified
1. **Plaintext Credentials**: All SIP credentials, Wi-Fi keys, TR-069 logins, and admin passwords are stored in cleartext inside the XML config.
2. **Weak / Disabled SSL Checks**: `X_5067F0_CheckCertificateCN` is set to `0`, making TR-069 management updates susceptible to man-in-the-middle attacks.
3. **Open Management Ports**: The router opens MQTT (`8883`) and Connection Request HTTP (`7547`) ports directly to the public WAN interface.
4. **Persistent Telemetry**: The USP agent over MQTT allows the ISP to query the local network state and make configuration changes instantly in real-time.
