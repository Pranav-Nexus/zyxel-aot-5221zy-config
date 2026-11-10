# Zyxel AOT-5221ZY Router Configuration & Customization Toolkit

A comprehensive security research, configuration management, and reverse-engineering toolkit for the **Zyxel AOT-5221ZY** (Wi-Fi 6 GPON ONT router, customized for Bharti Airtel Xstream Fiber by Embedur Systems).

This repository provides tooling, reference configurations, and deep-dive technical reports to safely backup, decrypt, customize, and restore the router's romfile configuration.

---

## What This Project Does

Airtel's custom firmware for the Zyxel AOT-5221ZY locks down critical router capabilities in the web UI. This toolkit unlocks full user control through romfile reverse engineering:

1. **Custom DNS Enforcement (Bypass ISP DNS)**:
   - Overrides ISP-enforced DNS on both IPv4 and IPv6 connections.
   - Routes queries cleanly to Cloudflare (`1.1.1.1` / `1.0.0.1`), Quad9, Google, or NextDNS without WAN DHCP overrides reverting them.
2. **Disable Wi-Fi AP Isolation (`X_5067F0_Isolation`)**:
   - The stock firmware isolates wireless clients, preventing devices on the same Wi-Fi network from communicating.
   - Disabling AP isolation restores LAN-to-LAN communication (essential for Home Assistant, local SSH, file sharing, and smart home hubs).
3. **Fix Local Multicast Discovery / mDNS / SSDP (`X_5067F0_IgmpSnoopingEnable`)**:
   - Aggressive IGMP snooping on bridge `br0` frequently blocks local mDNS (Bonjour, Zeroconf) and SSDP broadcasts.
   - Disabling IGMP snooping resolves discovery issues for Google Cast / Chromecast, Google Home, Nest, smart speakers, Apple AirPlay, and IoT devices.
4. **Configuration Decryption & Encryption (`zyxel_romfile_tool.py`)**:
   - Automates extraction, decryption, encryption, and restoration of the router's binary romfile (`romfile.cfg`).
   - Uses the reverse-engineered AES-256-CBC cipher key extracted from the firmware shared library (`/lib/MSTC/libCmd.so`).

---

## Repository Contents

* **`zyxel_romfile_tool.py`**: A cross-platform Python CLI utility to download, decrypt, encrypt, restore, and extract firmware configurations.
* **`modified_router_config.xml`**: Sanitized reference configuration showing exact XML settings for DNS override, AP isolation disabled, and IGMP snooping disabled.
* **`original_factory_backup.xml`**: Sanitized reference XML representing clean factory defaults.
* **`original_factory_backup.cfg`**: Sanitized encrypted reference romfile.
* **`deep_dive_report.md`**: Exhaustive technical report covering the router hardware, TR-069 CWMP, TR-369 USP MQTT (Corteca), VoIP SIP routing on VLAN 660, and security vulnerabilities.
* **`modification_report.md`**: Step-by-step diff log detailing each applied configuration modification.

> [!WARNING]
> **Safety Notice**: All `.xml` and `.cfg` configuration files in this repository are **sanitized reference templates**. Never restore another user's raw configuration directly onto your router, as router backups contain device-specific GPON Serial Numbers, MAC addresses, and SIP identifiers. Always back up your **own** router configuration first and apply edits to your own backup.

---

## Prerequisites

1. **Python 3.8+**
2. **OpenSSL**:
   - **Linux**: `sudo apt install openssl`
   - **macOS**: `brew install openssl`
   - **Windows**: Included with Git for Windows. The tool automatically detects OpenSSL under `C:\Program Files\Git\usr\bin\openssl.exe`. Alternatively, add Git's `usr/bin` or OpenSSL to your system `PATH`:
     ```powershell
     $env:PATH += ";C:\Program Files\Git\usr\bin"
     ```

---

## Quick Start: How to Modify Your Router

### 1. Download & Decrypt Your Current Router Configuration
Extract your active router configuration into a readable XML file:
```powershell
python zyxel_romfile_tool.py full-backup -u admin -p 'YOUR_ROUTER_PASSWORD' -o my_config.xml
```

### 2. Apply Custom Modifications to `my_config.xml`

#### A. Enforce Cloudflare / Custom DNS
Under `<WANPPPConnection>`, enable the DNS override flags:
```xml
<DNSOverrideAllowed PARAMETER="configured" TYPE="boolean">1</DNSOverrideAllowed>
<X_5067F0_DNSOverride PARAMETER="configured" TYPE="boolean">1</X_5067F0_DNSOverride>
<DNSServers>
  <i1>
    <ServerIP PARAMETER="configured" TYPE="string" LENGTH="16">1.1.1.1</ServerIP>
    <ServerType PARAMETER="configured" TYPE="string" LENGTH="32">UserDefined</ServerType>
    <X_5067F0_Enable PARAMETER="configured" TYPE="boolean">1</X_5067F0_Enable>
  </i1>
  <i2>
    <ServerIP PARAMETER="configured" TYPE="string" LENGTH="16">1.0.0.1</ServerIP>
    <ServerType PARAMETER="configured" TYPE="string" LENGTH="32">UserDefined</ServerType>
    <X_5067F0_Enable PARAMETER="configured" TYPE="boolean">1</X_5067F0_Enable>
  </i2>
</DNSServers>
```

#### B. Disable Wireless AP Isolation
Under `<LANDevice>` -> `<i1>` -> `<WLANConfiguration>`, add `<X_5067F0_Isolation>0</X_5067F0_Isolation>` to both 2.4 GHz (`<i1>`) and 5 GHz (`<i5>`) radios:
```xml
<SSID PARAMETER="configured" TYPE="string" LENGTH="32">Your_SSID</SSID>
<X_5067F0_Isolation PARAMETER="configured" TYPE="boolean">0</X_5067F0_Isolation>
```

#### C. Disable Bridge IGMP Snooping (Fix mDNS / SSDP Discovery)
Under `<LANDevice>` -> `<LANHostConfigManagement>` -> `<IPInterface>` -> `<i1>` (bridge `br0`):
```xml
<X_5067F0_Name PARAMETER="configured" TYPE="string" LENGTH="15">br0</X_5067F0_Name>
<X_5067F0_IgmpSnoopingEnable PARAMETER="configured" TYPE="boolean">0</X_5067F0_IgmpSnoopingEnable>
<X_5067F0_IGMPSnoopingEnable PARAMETER="configured" TYPE="boolean">0</X_5067F0_IGMPSnoopingEnable>
```

### 3. Re-encrypt & Restore to the Router
Upload the customized configuration directly back to the router (the router will automatically reboot with your changes applied):
```powershell
python zyxel_romfile_tool.py full-restore -i my_config.xml -u admin -p 'YOUR_ROUTER_PASSWORD'
```

---

## Utility Command Reference

```
usage: zyxel_romfile_tool.py [-h] [-r ROUTER] [-u USERNAME] [-p PASSWORD]
                             [-i INPUT] [-o OUTPUT] [-k KEY]
                             {download,decrypt,encrypt,restore,full-backup,full-restore,extract}
```

| Subcommand | Description |
| :--- | :--- |
| `full-backup` | Authenticates, downloads `romfile.cfg`, and decrypts to XML in one step. |
| `full-restore` | Encrypts XML and uploads it to the router, triggering an immediate reboot. |
| `decrypt` | Decrypts an existing `.cfg` binary file to `.xml`. |
| `encrypt` | Encrypts an `.xml` configuration file to a binary `.cfg` romfile. |
| `download` | Downloads raw encrypted romfile from router without decrypting. |
| `restore` | Uploads an encrypted `.cfg` binary file to router. |
| `extract` | Analyzes and unpacks firmware images using `binwalk`. |

---

## Technical Architecture & Cryptographic Analysis

The Zyxel AOT-5221ZY firmware uses OpenSSL symmetric cryptography to protect backup files:

- **Algorithm**: `AES-256-CBC`
- **Key Derivation**: `MD5` (`OpenSSL EVP_BytesToKey`)
- **Key**: Hardcoded in firmware binary `/lib/MSTC/libCmd.so` (`EwhJaD44DfprDOs7OXx9jzAtLg5PKtD8`)
- **Container Format**: Standard OpenSSL salted format beginning with magic bytes `Salted__` (`0x53 0x61 0x6C 0x74 0x65 0x64 0x5F 0x5F`) followed by an 8-byte salt.

Manual equivalent shell commands:
```bash
# Decrypt
openssl aes-256-cbc -md MD5 -k "EwhJaD44DfprDOs7OXx9jzAtLg5PKtD8" -d -in romfile.cfg -out romfile.xml

# Encrypt
openssl aes-256-cbc -md MD5 -k "EwhJaD44DfprDOs7OXx9jzAtLg5PKtD8" -e -in romfile.xml -out romfile.cfg
```

For full details on TR-069, TR-369 USP MQTT telemetry, and SIP voice VLAN configuration, see [deep_dive_report.md](deep_dive_report.md).

---

## License

Educational and research use only. All trademarks belong to their respective owners.
