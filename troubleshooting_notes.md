# Troubleshooting Journey & Design Notes

This document details the initial plan, the systematic troubleshooting process, and the evolution of the network design that led to the final, stable configuration.

## Executive Summary

This project documents the design and implementation of a secure, segmented home network using **VLANs** to isolate trusted devices, IoT hardware, gaming consoles, and a lab environment. The initial "router-on-a-stick" design proved unstable due to hardware and firmware limitations between the router and the managed switch. Through systematic, advanced troubleshooting—including Layer 2 diagnostics with `arp` and DHCP service log analysis—the root causes were identified. The final, robust architecture utilizes a physically separate management link, ensuring network stability and security. This document details the implementation journey, the final working configuration, and the validation of a highly reliable, multi-VLAN network.

---

## 1. Project Definition & Initial Plan

The primary goal was to move from a flat home network to a segmented architecture to enhance security and improve network management.

### 🎯 Goals

- Implement network segmentation using **VLANs** for enhanced security and organization.
- Isolate **IoT** devices, **gaming** traffic, **lab** experiments, and **trusted** devices into separate broadcast domains.
- Configure inter-VLAN routing and firewall rules with a default-deny policy.
- Gain practical experience with router firmware and **managed switch** configuration.
- Document the entire process for a professional portfolio.

### 💻 Equipment List

| Device | Model/Type | Role |
| --- | --- | --- |
| Primary Workstation | Laptop Computer | User Device |
| Managed Switch | 5-Port Switch | VLAN Management |
| Router | Wireless Router | Gateway, DHCP, Firewall |
| Network Services Hub | Raspberry Pi | Lab Server |
| Gaming Console | Gaming Console | Gaming Device |
| Smart Home Devices | Smart Speaker, Smart Outlets | IoT Devices |
| Other Devices | Additional User Devices | User Devices |

### 📋 VLAN & Subnet Plan

| VLAN ID | Name | IP Subnet | Gateway IP | DHCP Range | Purpose |
| --- | --- | --- | --- | --- | --- |
| `1` | **Management** | `192.168.1.0/24` | `192.168.1.1` | `192.168.1.100-150` | Secure access to network hardware |
| `10` | **Main** | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.100-150` | Trusted personal devices |
| `20` | **Gaming** | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.100-150` | Isolated gaming traffic |
| `30` | **TV** | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.100-150` | Smart TV streaming |
| `40` | **IoT** | `192.168.40.0/24` | `192.168.40.1` | `192.168.40.100-150` | Untrusted smart home devices |

---

## 2. Implementation Journey & Troubleshooting

### Issue 1: Total Network Lockout on Initial Configuration

-   **Symptom:** After applying a "big bang" configuration of all five VLANs, interfaces, and firewall zones, all network connectivity was lost, including access to the router's admin page at `192.168.1.1`, forcing a factory reset.
-   **Diagnosis:** The simultaneous configuration of all segments was too complex to troubleshoot effectively. It was impossible to identify which specific part of the configuration had failed.
-   **Outcome:** The key takeaway was to adopt an incremental approach: configure and validate one VLAN completely before proceeding to the next.

### Issue 2: Single VLAN Bridging Conflict

-   **Symptom:** Adopting the new incremental approach, I attempted to configure only `MAIN_VLAN` (VLAN 10). The result was the same: a complete loss of internet and router admin access, requiring another factory reset.
-   **Diagnosis:** The critical insight came from inspecting the OpenWRT bridge configuration. The `lan1` physical port was simultaneously a member of two different bridges. It was configured to pass **untagged** traffic for the `br-lan` bridge while also being required to handle **tagged** traffic for its `lan1.10` sub-interface. This conflict destabilized the network.
-   **Outcome:** The key learning was that a physical port designated as a **VLAN trunk** must be exclusively dedicated to that role and removed from any default untagged bridge memberships.

### Issue 3: Clients Failing to Receive DHCP Leases

-   **Symptom:** Clients connecting to the `MAIN_VLAN` (VLAN 10) SSID were not receiving IP addresses.
-   **Diagnosis:** I used SSH to access the router and ran `logread -f | grep dnsmasq` to monitor the DHCP service in real-time. No `DHCPDISCOVER` messages appeared, proving the requests were being dropped before reaching the service. This isolated the problem to the firewall.
-   **Outcome:** Any new, restrictive firewall zone requires explicit "allow" rules for core services. I had forgotten to permit DHCP (UDP ports 67-68) and DNS (TCP/UDP port 53) traffic from the new VLAN to the router itself.

### Issue 4: Intermittent and Unrecoverable Loss of Switch Admin Access

-   **Symptom:** At various stages, I would lose the ability to ping or connect to the switch's admin page at `192.168.1.2`, even from a device on the correct management VLAN.
-   **Diagnosis:** Running the `arp -a` command on the router confirmed the switch was not responding to ARP requests:
    ```bash
    $ arp -a
    ? (192.168.1.1) at 0a:1b:2c:3d:4e:5f [ether] on br-lan
    ? (192.168.1.2) at <incomplete> on br-lan
    ```
-   This `<incomplete>` status proved the problem was a complete Layer 2 failure; the switch's management plane had crashed.
-   **Outcome:** Verifying Layer 2 responsiveness with `arp` is a more conclusive diagnostic step than `ping` when a local device is unreachable. This issue pointed to a deeper instability in the switch firmware.

### Issue 5: Final Trunk Instability and Definitive Resolution

-   **Symptom:** The "router-on-a-stick" setup proved unstable. The router would intermittently lose its WAN connection, and access to the switch remained unreliable.
-   **Diagnosis:** I concluded this method was not reliable between these two specific pieces of hardware, likely due to subtle firmware incompatibilities.
-   **Outcome:** I abandoned the "router-on-a-stick" model. A dedicated router port and cable were used exclusively for the untagged **Management VLAN**, while a second, separate router port and cable were used for the **tagged VLAN trunk**. This new topology was immediately stable. The ultimate lesson was that a physically separate management link is an inherently more robust design.
