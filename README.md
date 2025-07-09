# Enterprise Network Lab: Secure Site-to-Site VPN with GRE, OSPF, HSRP & NAT
This lab demonstrates a real-world enterprise network setup connecting two office locations (Chennai HQ and Mumbai Branch) securely over the public internet. The implementation includes:

---

## Demo Video

 [Watch the Demo](https://github.com/saran04092001/Virtual-Network-Infrastructure-Setup/blob/main/project%201/Virtual-Network-Infrastructure-demo.mp4)

---


## Network Topology Stack

| Layer              | Technology       | Purpose                          | Example Config                          |
|--------------------|------------------|----------------------------------|-----------------------------------------|
| **Access Layer**   | Cisco IOS Switches | VLAN segmentation (10,20,30)     | `switchport access vlan 10`             |
| **Distribution**   | HSRP + OSPF      | Gateway redundancy & routing     | `standby 10 ip 192.168.10.100`          |
| **Core**          | Cisco ISR Router | Inter-VLAN routing & VPN termination | `router ospf 1`                         |
| **Connectivity**   | GRE over IPsec   | Secure site-to-site tunnel        | `tunnel source Serial0/0/0`             |
| **Security**       | NAT/PAT          | Internet access for private IPs   | `ip nat inside source list 10 interface overload` |
| **Monitoring**     | CDP/LLDP         | Device discovery                 | `show cdp neighbors`                    |

---

##  How It Works

1. **User Traffic Initiation**
   - End devices (PCs/servers) in Chennai HQ generate traffic destined for Mumbai Branch servers (HTTP/FTP).

2. **VLAN Segmentation & Routing**  
   - Access switches tag traffic by department (VLANs 10/20/30).
   - Distribution layer switches:
        - Use HSRP (192.168.10.100) as default gateway
        - Route traffic via OSPF-learned paths

3. **Secure Tunnel Establishment**
   - A[Chennai Edge Router] -->|Encapsulates in GRE| B(Public Internet) B -->|Decapsulates| C[Mumbai Edge Router]
     
4. **Internet Access (NAT Process)**
    | Step	| Action	                                 | Example              |
    |-------|------------------------------------------|-----------------------------| 
    | 1	  | Private IP (192.168.10.5) sends request	| src:192.168.10.5 → dst:8.8.8.8 |
    | 2	  | Edge router translates to public IP	    | src:100.1.1.1 → dst:8.8.8.8 |
    | 3	  | Response returns via NAT table	          | dst:100.1.1.1 → 192.168.10.5 |
   

6. **Failover Scenario**  
   - If primary distribution switch fails:
        - HSRP detects failure (3 sec hello timer)
        - Standby switch takes over virtual IP (192.168.10.100)
        - OSPF recalculates routes within 10 seconds
    
---

## 🛠️ Installation

### 1. Clone the Repository
```bash
| **Distribution**    | HSRP             | ```cisco
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
 standby 10 ip 192.168.10.100
 standby 10 priority 110
 standby 10 preempt
``` |
| **Core Routing**    | OSPF             | ```cisco
router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 10.1.1.0 0.0.0.3 area 0
``` |
| **VPN**            | GRE Tunnel       | ```cisco
interface Tunnel1
 ip address 172.16.1.1 255.255.255.252
 tunnel source Serial0/0/0
 tunnel destination 100.1.11.1
``` |
| **NAT**            | PAT Overload     | ```cisco
ip nat inside source list 10 interface Serial0/0/0 overload
access-list 10 permit 192.168.0.0 0.0.255.255
``` |
| **Management**     | SNMP             | ```cisco
snmp-server community MyROCommunity RO
snmp-server host 192.168.10.100 version 2c MyROCommunity
``` |

### **Complete Device Configuration Examples**

**1. Access Switch (Catalyst 2960)**
```cisco
enable
configure terminal
!
vlan 10
 name Engineering
vlan 20
 name Finance
!
interface range FastEthernet0/1-24
 switchport mode access
 switchport access vlan 10
 switchport port-security maximum 3
 switchport port-security
!
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
!
show vlan brief                 # Verify VLAN assignments
show standby brief              # Check HSRP status
show ip ospf neighbor           # Verify OSPF adjacencies
show ip nat translations        # Monitor NAT operations
ping 172.16.1.2 source 192.168.10.1  # Test GRE tunnel
