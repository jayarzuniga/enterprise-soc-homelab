1. Introduction to Pfsense and Suricata

1.1 What is Pfsense?
pfSense is a free, open-source firewall and router distribution based on FreeBSD. It provides enterprise-grade network security featuring the following:
- Stateful paket filtering firewall
- Network Address Translation (NAT)
- Virtual Private Network (VPN) support (IPsec, OpenVPN, WireGuard)
- Traffic shaping and Quality of Service (QoS)
- Intrusion Detection/Prevention via Suricata and Snort
- Detailed logging and reporting
- Multi-WAN capabilities
- VLAN support for network segmentation

1.2 What is Suricata
Suricata is a high-performance, open-source Network IDS/IPS and Network Security Monitoring engine. It provides:
- Real-time intrusion detection and prevention
- Protocol analysis and file extraction
- Multi-threaded packet processing for high performance
- Lua scripting for custom detection logic
- Compatible with Emerging Threats and Snort rules
- JSON logging for SIEM integration (perfect for Wazuh)
- TLS/SSL certificate logging
- DNS request and response logging
- HTTP transaction logging

1.3 Objectives for this SOC Homelab
- Simulate enterprise network architecture with VLANs and segmentation
- Practice firewall rule creation and management
- Detect network-based attacks in real-time with Suricata
- Monitor and analyze network traffic patterns
- Learn professional network security skills
- Integrate with Wazuh SIEM for centralized logging and correlation
- Test security controls against attack simulations

2. SOC Lab System Architecture

2.1 Creating your Network Topology
For me I got this topology:
**Internet** → **ISP Router** (192.168.0.1) → **TP-Link Router** → **pfSense Firewall**(Virtual Maching) → **Segmented Networks** (Such as Servers, Workstations)

|      Network|          Subnet|                                         Purpose|

|------------:|---------------:|-----------------------------------------------:|

|          WAN|     192.168.0.x|          Internet connection via TP-Link router|
|          DMZ|  192.168.3.0/24| Public-facing servers (Wazuh, NextCloud, Proxy)|
|     INTERNAL| 192.168.10.0/24|                   User workstations and clients|
|       SERVER| 192.168.20.0/24|  Backend services (AD, databases, file servers)|
|          SOC| 192.168.30.0/24|    Security monitoring tools (Zeek, dashboards)|
|     ATTACKER| 192.168.50.0/24|                  Attack simulation (Kali Linux)|

