# Enterprise Network Segmentation: Inter-VLAN Routing (Router-on-a-Stick) & Cisco IOS DHCP

## 1. Project Overview & Objective
This lab demonstrates how to partition a local area network into multiple secure broadcast domains using 802.1Q VLANs on Cisco Catalyst switching, while providing dynamic routing between subnets via a single physical link (Router-on-a-Stick) on a Cisco IOS router. Dynamic IP allocation is handled via centralized router-based DHCP pools, and management services are isolated on a dedicated server VLAN.

## 2. Topology & Addressing Table

| Device | Interface | VLAN | IP Address | Subnet Mask | Default Gateway | Role / Description |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **R1** | G0/1.10 | 10 | 192.168.10.1 | 255.255.255.0 | N/A | Subinterface - Sales Default Gateway |
| **R1** | G0/1.20 | 20 | 192.168.20.1 | 255.255.255.0 | N/A | Subinterface - Engineering Default Gateway |
| **R1** | G0/1.99 | 99 | 192.168.99.1 | 255.255.255.0 | N/A | Subinterface - Management/Server Gateway |
| **SW1** | VLAN 99 | 99 | 192.168.99.2 | 255.255.255.0 | 192.168.99.1 | Switch Virtual Interface (SVI) |
| **PC1** | Eth0 | 10 | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 | Sales Workstation (DHCP Assigned) |
| **PC2** | Eth0 | 20 | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 | Engineering Workstation (DHCP Assigned) |
| **SRV1** | Eth0 | 99 | 192.168.99.10 | 255.255.255.0 | 192.168.99.1 | Infrastructure / Management Server (Static) |

---

## 3. Device Configurations

### Switch Configuration (SW1)
```text
enable
configure terminal
hostname SW1

! Define VLANs
vlan 10
 name Sales
vlan 20
 name Engineering
vlan 99
 name Management
exit

! Configure Trunk to Router
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 no shutdown
exit

! Assign Access Ports to Endpoints
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
 no shutdown
exit

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 20
 no shutdown
exit

interface FastEthernet0/3
 switchport mode access
 switchport access vlan 99
 no shutdown
exit

! SVI for Switch Management
interface vlan 99
 ip address 192.168.99.2 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.99.1
end
write memory
```

### Router Configuration (R1)
```text
enable
configure terminal
hostname R1

! Activate physical trunk interface
interface GigabitEthernet0/1
 no shutdown
exit

! VLAN 10 Subinterface & DHCP
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
exit

ip dhcp excluded-address 192.168.10.1 192.168.10.9
ip dhcp pool POOL_SALES
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
exit

! VLAN 20 Subinterface & DHCP
interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
exit

ip dhcp excluded-address 192.168.20.1 192.168.20.9
ip dhcp pool POOL_ENG
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
exit

! VLAN 99 Subinterface
interface GigabitEthernet0/1.99
 encapsulation dot1Q 99
 ip address 192.168.99.1 255.255.255.0
exit
end
write memory
```

---

## 4. Verification & Testing

1. **Trunking Verification:**
   * Executing `show interfaces trunk` on SW1 confirms interface `Gi0/1` is actively operating in 802.1Q encapsulation with VLANs 10, 20, and 99 permitted and forwarding.
2. **IP Routing State:**
   * Executing `show ip route` on R1 displays directly connected networks for `192.168.10.0/24`, `192.168.20.0/24`, and `192.168.99.0/24` tied to subinterfaces `Gi0/1.10`, `Gi0/1.20`, and `Gi0/1.99`.
3. **End-to-End Connectivity:**
   * `PC1` pings its default gateway `192.168.10.1` (Success).
   * `PC1` pings across VLANs to `PC2` (`192.168.20.10`) via R1 routing (Success).
   * `PC1` and `PC2` ping `SRV1` (`192.168.99.10`) on the Management subnet (Success).
