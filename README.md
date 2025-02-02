# Self-Hosting with a Static IPv4 via VPS NAT Bridge

## Problem

Do you need a static IPv4 for your self-hosted services but only have a Carrier-Grade NAT (CGNAT) IPv4 from your ISP? If your router shows an IPv4 similar to `100.64.x.x`, then you are sharing your IPv4 address with other customers. However, your ISP has assigned you an IPv6 subnet.

## Solution

Set up a NAT bridge using an inexpensive VPS (€1/month) or a free Oracle FreeTier VPS to forward all IPv4 traffic. The VPS acts as a gateway, and traffic is encrypted using a WireGuard VPN tunnel.

---

## Setup Guide

### On the VPS

#### 1. Update the system
```bash
apt-get update && apt-get upgrade -y
```

#### 2. Install PiVPN
```bash
curl -L https://install.pivpn.io | bash
```

#### 3. Find the network interface name
```bash
ip a
```

#### 4. Edit `wg0.conf`
```bash
nano /etc/wireguard/wg0.conf
```
Add the following line after `ListenPort` (replace `your_interface` with the actual interface name):
```bash
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o your_interface -j MASQUERADE
```

#### 5. Reboot
```bash
reboot
```

#### 6. Create a WireGuard client profile
```bash
pivpn -a
```
Enter a profile name, e.g., `home_network`.

#### 7. Edit `wg0.conf` again
```bash
nano /etc/wireguard/wg0.conf
```
Add your home network's IP range under `AllowedIPs`:
```bash
AllowedIPs = 10.70.193.2/32, 192.168.8.0/24
```

#### 8. Save the configuration temporarily
```bash
cat /etc/wireguard/configs/home_network.conf
```

#### 9. Enable IP forwarding
```bash
nano /etc/sysctl.conf
```
Change:
```bash
#net.ipv4.ip_forward = 1
```
to:
```bash
net.ipv4.ip_forward = 1
```

#### 10. Create a NAT forwarding script
```bash
nano /ip.sh
```
Add:
```bash
#!/bin/bash
iptables -t nat -A POSTROUTING -j MASQUERADE
```

#### 11. Forward all ports to a local machine (e.g., Nginx Proxy Manager)
```bash
iptables -A PREROUTING -t nat -p tcp --match multiport ! --dports 22,51820 -j DNAT --to-destination 192.168.8.14
iptables -A PREROUTING -t nat -p udp --match multiport ! --dports 22,51820 -j DNAT --to-destination 192.168.8.14
```

#### 12. Forward a single port (e.g., 80 for a web server)
```bash
iptables -t nat -A PREROUTING -p "tcp" --dport "80" -j DNAT --to-destination "192.168.8.14":"80"
```

#### 13. Save iptables rules
```bash
sudo apt install netfilter-persistent
sudo netfilter-persistent save
sudo systemctl enable netfilter-persistent
```

#### 14. Adjust permissions and automate at boot
```bash
chmod +x /ip.sh
crontab -e
```
Add:
```bash
@reboot /ip.sh
```

#### 15. Reboot
```bash
reboot
```

---

### On the Home Server

#### 1. Install required packages
```bash
apt-get update && apt-get upgrade -y && apt-get install -y wireguard openresolv wireguard-dkms
```

#### 2. Edit `wg0.conf` and insert content from the VPS
```bash
nano /etc/wireguard/wg0.conf
```

#### 3. Add before `[Peer]` and adjust interface (`your_interface`):
```bash
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o your_interface -j MASQUERADE
PostUp = echo 1 > /proc/sys/net/ipv4/ip_forward
```

#### 4. Append at the end:
```bash
PersistentKeepalive = 25
```

#### 5. Enable WireGuard
```bash
systemctl enable wg-quick@wg0
```

#### 6. Reboot
```bash
reboot
```

---

### Firewall Adjustments (Example: IONOS VPS)
If using IONOS, allow the WireGuard port in the firewall settings.

---

## Optional Adjustments

### Enable DNS for WireGuard Tunnel

Edit `wg0.conf`:
```bash
DNS = 9.9.9.9
```

Disable stub resolver:
```bash
/etc/systemd/resolved.conf.d/disable-stub.conf
```
Add:
```ini
[Resolve]
DNSStubListener=no
```
Restart systemd-resolved:
```bash
sudo systemctl restart systemd-resolved
```

---

## Disabling VPN Tunnel for Maintenance

#### Turn off PiVPN
```bash
pivpn off
```

#### Turn off WireGuard
```bash
service wg-quick@wg0 stop
```

---

## Secure SSH Access to the VPS

To prevent unauthorized access, use SSH key authentication and dynamically allow only your current IP via DNS lookup.

#### 1. Create a script
```bash
nano /dynamic-ssh-access.sh
```
Paste the provided script. https://github.com/JonaxScript/DynDNS-SSH-Iptables

#### 2. Automate execution with crontab (every 5 minutes):
```bash
*/5 * * * * root /path/to/dynamic-ssh-access.sh
```

---

## Final Notes

Your self-hosted services are now accessible via the VPS IPv4 while traffic remains encrypted through WireGuard. Adjust firewall rules and NAT settings as needed.

