# Full DNS Server Configuration and Documentation

## Introduction: What is DNS and Why is it Needed?

DNS (Domain Name System) is a system that converts domain names (like www.google.com) into IP addresses (like 142.250.190.68).

Why it's needed:
- Users can access websites using names.
- Enables name-based communication in networks.
- Without DNS, we’d need to remember IPs for all services.

Where it's used:
- Local LANs
- Internal service resolution
- Internet hosting

Who needs it:
- System Administrators
- Network Engineers
- Web and Server Developers

## Server Plan

All servers are running in VMs on Windows 11. Ubuntu 24.04.2 LTS is used within the VMs.

| VM   | Hostname       | IP             | Role       |
|------|----------------|----------------|------------|
| vm-1 | ns1.raihan.dns | 192.168.56.115 | Master DNS |
| vm-2 | ns2.raihan.dns | 192.168.56.116 | Slave DNS  |
| vm-3 | ns3.raihan.dns | 192.168.56.117 | Slave DNS  |

## Hostname and /etc/hosts Setup

Run on each VM:
sudo hostnamectl set-hostname nsX.raihan.dns  # Replace X with 1, 2, or 3

Edit /etc/hosts on each:
192.168.56.115   ns1.raihan.dns ns1
192.168.56.116   ns2.raihan.dns ns2
192.168.56.117   ns3.raihan.dns ns3

## Netplan Network Configuration

Each VM should have a static IP for enp0s8 in /etc/netplan/50-cloud-init.yaml.
Use sudo netplan try && sudo netplan apply to apply.

## DNS Server Software Installation

Run on all servers:
sudo apt update
sudo apt install bind9 bind9utils bind9-doc dnsutils -y

## Master DNS Configuration (ns1)

1. Edit /etc/bind/named.conf.options to allow transfers and queries.
2. Define zones in /etc/bind/named.conf.local.
3. Create zone files in /etc/bind/zones/db.raihan.dns and db.192.

## Slave DNS Configuration (ns2 and ns3)

1. Configure /etc/bind/named.conf.options for read-only DNS.
2. Define slave zones in /etc/bind/named.conf.local pointing to master.

## Restart and Validate Configuration

Run:
sudo named-checkconf
sudo named-checkzone raihan.dns /etc/bind/zones/db.raihan.dns
sudo systemctl restart bind9

## Firewall Settings

Open DNS ports on all servers:
sudo firewall-cmd --add-port=53/tcp --permanent
sudo firewall-cmd --add-port=53/udp --permanent
sudo firewall-cmd --reload

## DNS Testing

Forward Lookup:
dig @192.168.56.115 www.raihan.dns

Reverse Lookup:
dig -x 192.168.56.115

Zone Transfer:
dig @192.168.56.116 raihan.dns AXFR

## Optional: resolvectl (if using systemd-resolved)

sudo resolvectl dns enp0s8 192.168.56.115 192.168.56.116 192.168.56.117

## Master vs Slave Differences

| Feature      | Master DNS          | Slave DNS             |
|--------------|---------------------|------------------------|
| Type         | Authoritative       | Read-only copy         |
| Zone Files   | Stored locally      | Pulled from master     |
| Updates      | Allows updates      | No direct update       |
| Transfer     | Sends zones         | Receives zones         |

## Tips and Best Practices

- Always increase Serial number after edits.
- Use named-checkconf and named-checkzone.
- Keep recursion disabled unless necessary.
- Open TCP and UDP port 53 in firewall.

## Final Notes

This setup enables a production-ready Master-Slave DNS infrastructure. You can later integrate with DHCP, load balancers, or external DNS.
