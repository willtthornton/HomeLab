# Network Setup & Configuration Guide

This guide details the final, stable configuration for the segmented home network. It uses a dual-link design: one untagged link for management and one tagged trunk link for all other VLANs.

---

## 1. Final Architecture & Configuration

### 🗺️ Final Network Diagram

*(Image placeholder for the final network diagram)*

### ⚙️ Final Configuration Guide

#### **Part 1: OpenWRT Router Configuration**

1.  **Initial Setup:**
    * Connect to the router's default Wi-Fi and navigate to `http://192.168.1.1`. Set an admin password.
    * Navigate to **Network → Interfaces**, `Edit` the `lan` interface, and set its **IPv4 address** to `192.168.1.1`. Save and apply. Reconnect to the router at its new IP.
2.  **Create VLAN Devices and Bridges:**
    * Navigate to **Network → Interfaces → Devices**.
    * Configure the default `br-lan` bridge. Ensure its `Bridge ports` list contains only `lan2` (the dedicated management port). `lan1` should **not** be in this bridge.
    * Click `Add device configuration...` and create the VLAN sub-interfaces and their corresponding bridges according to the table below. The process is:
        1.  Create the **VLAN (802.1q)** device (e.g., `lan1.10`) based on the `lan1` physical device.
        2.  Create a **Bridge device** (e.g., `br-main`) and add the corresponding VLAN device to its `Bridge ports`.

| VLAN ID | VLAN Device Name | Bridge Name |
| --- | --- | --- |
| 10 | `lan1.10` | `br-main` |
| 20 | `lan1.20` | `br-gaming` |
| 30 | `lan1.30` | `br-iot` |
| 40 | `lan1.40` | `br-lab` |
3.  **Create Network Interfaces:**
    * Navigate to the **Interfaces** tab and click `Add new interface...`.
    * Create a new interface for each VLAN, using the parameters in the table below. For each, set the **Protocol** to `Static address`, assign the correct **Device** (bridge), and configure the **IPv4 address** and **Netmask**. Then, under the **DHCP Server** tab, set up the DHCP server.

| Interface Name | Device (Bridge) | IPv4 Address | Netmask | DHCP Range |
| --- | --- | --- | --- | --- |
| `MAIN_VLAN` | `br-main` | `192.168.10.1` | `255.255.255.0` | `100-150` |
| `GAMING_VLAN` | `br-gaming` | `192.168.20.1` | `255.255.255.0` | `100-150` |
| `IOT_VLAN` | `br-iot` | `192.168.30.1` | `255.255.255.0` | `100-150` |
| `LAB_VLAN` | `br-lab` | `192.168.40.1` | `255.255.255.0` | `100-150` |
4.  **Configure Wireless SSIDs:**
    * Navigate to **Network → Wireless**.
    * Create an SSID for management assigned to the `lan` network.
    * Create separate SSIDs for your user and IoT devices, assigning them to their respective network interfaces (`MAIN_VLAN`, `IOT_VLAN`).
    * Enable **WPA2-PSK** or **WPA3** encryption for all SSIDs.
5.  **Configure Firewall:**
    * Navigate to **Network → Firewall**.
    * For each new interface (`MAIN_VLAN`, `GAMING_VLAN`, etc.), **assign it to a new firewall zone** (e.g., `main_zone`, `gaming_zone`).
    * For each new zone, set the default policies: **Input:** `REJECT`, **Output:** `ACCEPT`, **Forward:** `REJECT`.
    * For "Allow forward to destination zones," select **`wan`**.
    * Under the **Traffic Rules** tab, add rules for each new zone to allow essential services to the router:
        * **Allow DNS:** Source Zone: `[new_zone]`, Destination Zone: `Device (input)`, Protocol: `TCP+UDP`, Destination Port: `53`, Action: `accept`.
        * **Allow DHCP:** Source Zone: `[new_zone]`, Destination Zone: `Device (input)`, Protocol: `UDP`, Destination Port: `67-68`, Action: `accept`.
    * Click **Save & Apply**.

#### **Part 2: Switch Configuration**

1.  **Initial Setup:**
    * Connect the switch's **Port 4** to the router's `lan2` port. Power it on.
    * Find the switch's IP in the Router DHCP lease list. Log in (default password: `password`).
    * Navigate to `System > Management > IP Configuration` and set a **Static IP Address**:
        * IP Address: `192.168.1.2`
        * Subnet Mask: `255.255.255.0`
        * Default Gateway: `192.168.1.1`
2.  **VLAN Configuration:**
    * Navigate to **VLAN → 802.1Q VLAN → Advanced** and select **Enable**.
    * Under **VLAN Configuration**, create VLANs `10`, `20`, `30`, and `40`.
    * Under **VLAN Membership**, configure each port based on the table below. `T` = **Tagged**, `U` = **Untagged**.
    * Under **Port PVID**, set the Port VLAN ID for each port. The **PVID** dictates which VLAN any incoming *untagged* traffic is assigned to.

| Port | Purpose | VLAN 1 (Mgmt) | VLAN 10 (Main) | VLAN 20 (Game) | VLAN 30 (IoT) | VLAN 40 (Lab) | **PVID** |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | Lab Access | -- | -- | -- | -- | **U** | **40** |
| 2 | Gaming Access | -- | -- | **U** | -- | -- | **20** |
| 3 | Main Access | -- | **U** | -- | -- | -- | **10** |
| 4 | Management Link | **U** | -- | -- | -- | -- | **1** |
| 5 | **VLAN Trunk Link** | **U** | **T** | **T** | **T** | **T** | **1** |
3.  **Apply** all settings and save the configuration on the switch.

---
## 2. Testing & Validation

Final verification was performed to ensure all project goals were met.

| VLAN | Test Action | Expected Result | Status |
| --- | --- | --- | --- |
| **Management** | Connect to management SSID or physical port 4. | Get `192.168.1.x` IP. Access router and switch admin. | ✅ |
| **Main** | Connect to main SSID or physical port 3. | Get `192.168.10.x` IP. Access internet. | ✅ |
| **Gaming** | Connect console to physical port 2. | Get `192.168.20.x` IP. Access internet. | ✅ |
| **IoT** | Connect Speaker to IoT SSID. | Get `192.168.30.x` IP. Access internet. | ✅ |
| **Lab** | Connect Pi to physical port 1. | Get `192.168.40.x` IP. Access internet. | ✅ |
| **Isolation** | From any non-management VLAN (10, 20, 30, 40)... | | |
| | ...ping device in another VLAN (e.g., VLAN 10 -> 20) | **FAIL.** No inter-VLAN communication. | ✅ |
| | ...ping router/switch management IPs (`192.168.1.x`) | **FAIL.** No access to management plane. | ✅ |
| | ...access router/switch web admin pages. | **FAIL.** Web interfaces are inaccessible. | ✅ |
