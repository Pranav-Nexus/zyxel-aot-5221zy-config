# Router Configuration Modification Log

This document records the modifications made to the Zyxel AOT-5221ZY router configuration.

## 1. Summary of Changes

| Modification Date | Target Param / Feature | Modification | Impact |
| :--- | :--- | :--- | :--- |
| **June 30, 2026** | `X_5067F0_Isolation` (SSID 1 & SSID 5) | Added `<X_5067F0_Isolation>0</X_5067F0_Isolation>` | Disables AP isolation on 2.4 GHz and 5 GHz bands |
| **July 5, 2026** | `X_5067F0_IgmpSnoopingEnable`<br>`X_5067F0_IGMPSnoopingEnable` (Bridge `br0`) | Added both tags set to `0` under `<IPInterface><i1>` | Disables IGMP Snooping on LAN/WLAN bridge to allow mDNS/SSDP discovery |
| **June 30, 2026** | `DNSServers` | Preserved `1.1.1.1,1.0.0.1` | Ensures DNS queries continue using Cloudflare |

---

## 2. Configuration Diff

### July 5, 2026 - Disabling IGMP Snooping on bridge `br0`
```diff
         <IPInterface>
           <i1>
             <X_5067F0_Name PARAMETER="configured" TYPE="string" LENGTH="15">br0</X_5067F0_Name>
+            <X_5067F0_IgmpSnoopingEnable PARAMETER="configured" TYPE="boolean">0</X_5067F0_IgmpSnoopingEnable>
+            <X_5067F0_IGMPSnoopingEnable PARAMETER="configured" TYPE="boolean">0</X_5067F0_IGMPSnoopingEnable>
             <IPInterfaceAddressingType PARAMETER="configured" TYPE="string" LENGTH="15">Static</IPInterfaceAddressingType>
```

### June 30, 2026 - Disabling WLAN AP Isolation
```diff
   <LANDevice>
     <i1>
       <WLANConfiguration>
         <!-- 2.4 GHz SSID Configuration -->
         <i1>
           <SSID PARAMETER="configured" TYPE="string" LENGTH="32">Airtel_Home_WiFi</SSID>
+         <X_5067F0_Isolation PARAMETER="configured" TYPE="boolean">0</X_5067F0_Isolation>
           <X_5067F0_Enable11v PARAMETER="configured" TYPE="boolean">1</X_5067F0_Enable11v>
         </i1>
         ...
         <!-- 5 GHz SSID Configuration -->
         <i5>
           <SSID PARAMETER="configured" TYPE="string" LENGTH="32">Airtel_Home_WiFi</SSID>
+         <X_5067F0_Isolation PARAMETER="configured" TYPE="boolean">0</X_5067F0_Isolation>
           <X_5067F0_Enable11v PARAMETER="configured" TYPE="boolean">1</X_5067F0_Enable11v>
         </i5>
```

---

## 3. Step-by-Step Actions Taken (July 5, 2026)

### Step 1: Config Editing
* Located the bridge interface `br0` definition under the `<LANDevice>` -> `<LANHostConfigManagement>` -> `<IPInterface>` -> `<i1>` node.
* Injected both case variations of the IGMP Snooping enable tags:
  * `<X_5067F0_IgmpSnoopingEnable PARAMETER="configured" TYPE="boolean">0</X_5067F0_IgmpSnoopingEnable>`
  * `<X_5067F0_IGMPSnoopingEnable PARAMETER="configured" TYPE="boolean">0</X_5067F0_IGMPSnoopingEnable>`

### Step 2: Push Configuration to Router
* Ran the restore functionality on `zyxel_romfile_tool.py` using administrator credentials:
  ```powershell
  $env:PATH += ";C:\Program Files\Git\usr\bin"; echo yes | python zyxel_romfile_tool.py full-restore -i modified_router_config.xml -u admin -p 'YOUR_ROUTER_PASSWORD'
  ```
* **Process**:
  1. The XML configuration was successfully validated and encrypted.
  2. The script authenticated with the router and uploaded the new `.cfg` file.
  3. The router accepted the payload and rebooted to apply the settings.
