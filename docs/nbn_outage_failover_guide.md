# NBN Outage Failover & Recovery Guide (`<HOST_SERVER>`)

This guide documents the exact steps to fail over `<HOST_SERVER>`, your `<VM_NAME>` Multipass virtual machine, and your second Ethernet server to your iPhone Personal Hotspot during an NBN outage, keep Docker and Cloudflare Tunnels (`<YOUR_DOMAIN>`) online, and revert everything cleanly when NBN is restored.

---

## Hardware & Network Reference
* **Primary Host:** `<HOST_SERVER>` (Ubuntu Server)
* **WiFi Interface (WAN during outage):** `<WIFI_IFACE>` (e.g., Realtek `rtl8192ee` PCIe adapter)
* **Ethernet Interface (LAN to 2nd Server):** `<LAN_IFACE>` (Normally `<HOST_LAN_IP>/24`)
* **Hotspot Subnet (iOS Default):** `172.20.10.0/28` (Gateway: `172.20.10.1`)
* **Multipass VM:** `<VM_NAME>` (Zone: `<ZONE_NAME>`, Bridge: `<VM_BRIDGE>` on `<VM_BRIDGE_IP>/24`)
* **Network Manager:** `NetworkManager` (`nmcli`) enabled via `/etc/netplan/00-installer-config.yaml`

---

## Phase 1: Connect `<HOST_SERVER>` to the iPhone Hotspot

### 1. Prepare the iPhone Hotspot
1. Open **Settings > Personal Hotspot** on your iPhone.
2. Toggle **Allow Others to Join** ON.
3. Toggle **Maximise Compatibility** ON (forces 2.4 GHz so the Realtek `rtl8192ee` card can see it reliably).
4. **Keep the iPhone screen unlocked on the Personal Hotspot page** until `<HOST_SERVER>` finishes connecting (iOS stops broadcasting the SSID as soon as the screen locks).

### 2. Connect `<WIFI_IFACE>` using `nmcli`
Run these commands on `<HOST_SERVER>` to scan and connect to your hotspot:

```bash
sudo nmcli device wifi rescan ifname <WIFI_IFACE>
sudo nmcli device wifi connect "<HOTSPOT_SSID>" password "<HOTSPOT_PASSWORD>" ifname <WIFI_IFACE>
```

### 3. Clear Dead NBN Routes (If `ping 8.8.8.8` Fails)
If `<LAN_IFACE>` is still plugged into your dead NBN router, Ubuntu will try to route traffic through `<ROUTER_GATEWAY_IP>` instead of your WiFi card. Delete the dead default route:

```bash
sudo ip route del default via <ROUTER_GATEWAY_IP>
ping -c 4 8.8.8.8
```

---

## Phase 2: Share Internet to the 2nd Server via `<LAN_IFACE>`

By configuring `<HOST_SERVER>`'s `<LAN_IFACE>` port as `<ROUTER_GATEWAY_IP>/24` in `shared` mode, `<HOST_SERVER>` impersonates your home router. This provides DHCP, DNS, and NAT routing to your second server without needing to change the second server's IP configuration or Cloudflare Tunnel targets.

### 1. Connect the Ethernet Cable
1. Unplug the dead NBN router from your switch (or plug an Ethernet cable directly from `<HOST_SERVER>`'s `<LAN_IFACE>` port into the second server's Ethernet port).

### 2. Enable Internet Sharing on `<LAN_IFACE>`
Run these commands on `<HOST_SERVER>` to set up DHCP and routing on `<ROUTER_GATEWAY_IP>/24`:

```bash
CONN=$(nmcli -g GENERAL.CONNECTION device show <LAN_IFACE>)
sudo nmcli con mod "$CONN" ipv4.method shared ipv4.addresses <ROUTER_GATEWAY_IP>/24 ipv4.gateway ""
sudo nmcli con up "$CONN"
```

### 3. Bypass Docker's Firewall & Enable NAT Masquerading
Docker sets the default `iptables` `FORWARD` policy to `DROP`, which blocks routed traffic from the second server. Open the forward chain and ensure NAT masquerading is active over `<WIFI_IFACE>`:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -P FORWARD ACCEPT
sudo iptables -t nat -I POSTROUTING 1 -s <LAN_SUBNET>/24 -o <WIFI_IFACE> -j MASQUERADE
```

### 4. Wake Up and Verify the 2nd Server Connection
Unplug and reconnect the Ethernet cable on the second server (so it requests a fresh DHCP lease or updates ARP), then verify it appears on `<LAN_IFACE>`:

```bash
# Check active ARP neighbours on <LAN_IFACE>
ip neigh show dev <LAN_IFACE>

# Check issued DHCP leases (if the 2nd server uses DHCP)
sudo cat /var/lib/NetworkManager/dnsmasq-<LAN_IFACE>.leases
```

---

## Phase 3: Fix Host Docker & Cloudflare Tunnels (`<YOUR_DOMAIN>`)

When switching from Ethernet to a WiFi hotspot, Docker bridges and Cloudflare Tunnels can stall or hit subnet and protocol conflicts.

### 1. Check for Docker Subnet Collisions (`172.20.0.0/16`)
iPhone hotspots always use `172.20.10.0/28`. If Docker creates a custom bridge network on `172.20.0.0/16`, it will collide with your iPhone hotspot route and break routing on `<HOST_SERVER>`.

```bash
# Check if a Docker bridge is hijacking 172.20.x.x
ip -4 addr show | grep 172.20

# If an inactive br-xxxx bridge is using 172.20.0.1/16, delete it directly:
# sudo ip link delete br-<BRIDGE_ID>

# Prune unused conflicting Docker networks and restart Docker
docker network prune -f && sudo systemctl restart docker
```

### 2. Refresh Host Cloudflare Tunnel
```bash
sudo systemctl restart cloudflared 2>/dev/null
```

---

## Phase 4: Fix `<VM_NAME>` (Multipass) & Inner Cloudflare Tunnel

Multipass places `<VM_NAME>` in `<ZONE_NAME>` on the `<VM_BRIDGE>` virtual bridge (`<VM_BRIDGE_IP>/24`). Switching host interfaces or restarting the Multipass daemon while the VM is running can detach the virtual bridge (`NO-CARRIER`) and break DNS or QUIC tunnel connections inside the VM.

### 1. Ensure UFW Allows `<VM_BRIDGE>` & Isolates Local Subnets (On `<HOST_SERVER>`)
These rules allow the VM to receive DHCP/SSH on `<VM_BRIDGE>` and route outbound internet traffic over both `<WIFI_IFACE>` (hotspot) and `<LAN_IFACE>` (NBN), while blocking access to private LAN subnets:

```bash
sudo ufw allow in on <VM_BRIDGE>
sudo ufw allow out on <VM_BRIDGE>
sudo ufw route deny in on <VM_BRIDGE> out on <LAN_IFACE> to 192.168.0.0/16
sudo ufw route deny in on <VM_BRIDGE> out on <LAN_IFACE> to 10.0.0.0/8
sudo ufw route deny in on <VM_BRIDGE> out on <WIFI_IFACE> to 172.16.0.0/12
sudo ufw route allow in on <VM_BRIDGE> out on <WIFI_IFACE>
sudo ufw route allow in on <VM_BRIDGE> out on <LAN_IFACE>
```

### 2. Cold Reboot `<VM_NAME>` to Rebind `<VM_BRIDGE>` (On `<HOST_SERVER>`)
If `multipass shell <VM_NAME>` fails with `No route to host`, force stop the VM, restart the Multipass daemon, and perform a clean boot so the `tap` interface reattaches to `<VM_BRIDGE>`:

```bash
multipass stop --force <VM_NAME>
sudo snap restart multipass
multipass start <VM_NAME>
multipass list
```

*(Fallback: If `ip link show <VM_BRIDGE>` still shows `NO-CARRIER` after starting the VM, manually attach the `tap` interface)*:

```bash
for tap in $(ip -o link show type tap | awk -F': ' '{print $2}'); do
  sudo ip link set "$tap" master <VM_BRIDGE>
  sudo ip link set "$tap" up
done
sudo ip link set <VM_BRIDGE> up
```

### 3. Fix DNS & Cloudflare Tunnel Inside `<VM_NAME>`
Enter the VM shell from `<HOST_SERVER>`:

```bash
multipass shell <VM_NAME>
```

Inside `ubuntu@<VM_NAME>:~$`, ensure DNS points to Cloudflare and Google (rather than forwarding to the offline `<ROUTER_GATEWAY_IP>` router) and force `cloudflared` to use IPv4 over HTTP/2 (`Type=simple`) so the mobile hotspot does not drop UDP QUIC or IPv6 handshakes:

```bash
# 1. Set persistent public DNS servers inside the VM
sudo sed -i 's/^#*DNS=.*/DNS=1.1.1.1 8.8.8.8/' /etc/systemd/resolved.conf
sudo sed -i 's/^#*FallbackDNS=.*/FallbackDNS=1.0.0.1 8.8.4.4/' /etc/systemd/resolved.conf
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved

# 2. Configure cloudflared for HTTP/2 and IPv4 edge connections
sudo sed -i 's/Type=notify/Type=simple/' /etc/systemd/system/cloudflared.service
sudo sed -i 's|ExecStart=.*|ExecStart=/usr/bin/cloudflared --no-autoupdate tunnel --protocol http2 --edge-ip-version 4 run --token-file /etc/cloudflared/token|' /etc/systemd/system/cloudflared.service
sudo systemctl daemon-reload
sudo systemctl restart cloudflared

# 3. Verify the tunnel is active and registered
sudo systemctl status cloudflared --no-pager
```

*(Note: If `/etc/cloudflared/token` is ever missing or empty after a reinstall, recreate it with `echo "<YOUR_TUNNEL_TOKEN>" | sudo tee /etc/cloudflared/token > /dev/null && sudo chmod 600 /etc/cloudflared/token`)*

---

## Phase 5: Revert Back to Normal When NBN is Restored

Once your NBN connection is back online and you plug `<HOST_SERVER>` and your second server back into your normal router or switch:

### 1. Restore `<LAN_IFACE>` to Normal Client Mode
```bash
CONN=$(nmcli -g GENERAL.CONNECTION device show <LAN_IFACE>)

# Option A: If <HOST_SERVER> normally gets <HOST_LAN_IP> via Router DHCP reservation
sudo nmcli con mod "$CONN" ipv4.method auto ipv4.addresses "" ipv4.gateway ""
sudo nmcli con up "$CONN"

# Option B: If <HOST_SERVER> uses a manual static IP on <LAN_IFACE> (uncomment below if needed)
# sudo nmcli con mod "$CONN" ipv4.method manual ipv4.addresses <HOST_LAN_IP>/24 ipv4.gateway <ROUTER_GATEWAY_IP> ipv4.dns "1.1.1.1,8.8.8.8"
# sudo nmcli con up "$CONN"
```

### 2. Disconnect the WiFi Hotspot
```bash
sudo nmcli device disconnect <WIFI_IFACE>
```

### 3. Restart Docker, Multipass & Verify Routes
```bash
sudo systemctl restart docker
multipass stop --force <VM_NAME>
sudo snap restart multipass
multipass start <VM_NAME>
ip route
```
Confirm `default via <ROUTER_GATEWAY_IP> dev <LAN_IFACE>` is back at the top of `ip route`, then unplug and reconnect the Ethernet cable on your second server so it switches back to the main router.

---

## Appendix: Fresh OS Install / Emergency Notes

If you ever reinstall Ubuntu Server from scratch on `<HOST_SERVER>`, remember these three hardware quirks we solved:
1. **Ubuntu Server does not include `NetworkManager` by default:** Plug your iPhone in via USB cable, run `sudo ip link set <usb_interface> up`, and install it via `sudo apt update && sudo apt install -y network-manager iw`.
2. **Netplan Renderer:** Set `/etc/netplan/00-installer-config.yaml` (or `50-cloud-init.yaml`) to use NetworkManager:
   ```yaml
   network:
     version: 2
     renderer: NetworkManager
   ```
3. **Realtek `rtl8192ee` + iOS Hotspot PMF Bug:** Raw `wpa_supplicant` hangs when authenticating with an iPhone hotspot on the `rtl8192ee` driver unless Protected Management Frames are disabled (`ieee80211w=0` inside `/etc/wpa_supplicant.conf`). Using `NetworkManager` (`nmcli`) handles this negotiation automatically.