# Lokori community network in Cisco Packet Tracer

This lab models the Lokori network design and lets you watch packets move between its servers in Simulation mode. It was built for Packet Tracer 8.x.

Packet Tracer has no MikroTik, Ubiquiti or Starlink devices, so Cisco equivalents stand in for them:

| Real network | In Packet Tracer |
| --- | --- |
| MikroTik RB5009 core router | `HUB-CORE`, a Cisco 2911 router |
| Starlink satellite service | `STARLINK-GW`, a Cisco 1941 router doing carrier NAT |
| 4G backup router | `LTE-GW`, a Cisco 1941 router doing carrier NAT |
| 5 GHz point-to-multipoint radio links | Copper trunk links from `HUB-SW` to each site switch |
| Raspberry Pi content server | `CONTENT-SRV` (web and DNS) |
| Health records and file relay | `FILE-SRV` (FTP) |
| National KHIS and the internet | `INTERNET-SRV` (web) |

## 1. Place the devices

| Name | Packet Tracer device |
| --- | --- |
| HUB-CORE | 2911 router |
| STARLINK-GW, LTE-GW | 1941 router |
| HUB-SW, PRIMARY-SW, HIGH-SW, MARKET-SW, INTERNET-SW | 2960-24TT switch |
| CONTENT-SRV, FILE-SRV, INTERNET-SRV | Server-PT |
| HEALTH-PC | PC-PT |
| PRIMARY-PC, HIGH-PC | PC-PT |
| MARKET-LAPTOP | Laptop-PT |

## 2. Cable them

Use **Copper Straight-Through** for every link (or the automatic connection tool).

| From | Port | To | Port |
| --- | --- | --- | --- |
| HUB-CORE | G0/0 | HUB-SW | Gi0/1 |
| HUB-CORE | G0/1 | STARLINK-GW | G0/0 |
| HUB-CORE | G0/2 | LTE-GW | G0/0 |
| STARLINK-GW | G0/1 | INTERNET-SW | Fa0/1 |
| LTE-GW | G0/1 | INTERNET-SW | Fa0/2 |
| INTERNET-SRV | Fa0 | INTERNET-SW | Fa0/3 |
| HEALTH-PC | Fa0 | HUB-SW | Fa0/1 |
| CONTENT-SRV | Fa0 | HUB-SW | Fa0/2 |
| FILE-SRV | Fa0 | HUB-SW | Fa0/3 |
| HUB-SW | Fa0/21 | PRIMARY-SW | Gi0/1 |
| HUB-SW | Fa0/22 | HIGH-SW | Gi0/1 |
| HUB-SW | Fa0/23 | MARKET-SW | Gi0/1 |
| PRIMARY-PC | Fa0 | PRIMARY-SW | Fa0/1 |
| HIGH-PC | Fa0 | HIGH-SW | Fa0/1 |
| MARKET-LAPTOP | Fa0 | MARKET-SW | Fa0/1 |

## 3. Paste the configurations

Open each router or switch, go to the **CLI** tab, press Enter until you see the `>` prompt, and paste its whole file from `configs/`, in this order:

1. `01-HUB-CORE.txt`
2. `02-HUB-SW.txt`
3. `03-PRIMARY-SW.txt`, `04-HIGH-SW.txt`, `05-MARKET-SW.txt`
4. `06-STARLINK-GW.txt`, `07-LTE-GW.txt`

`INTERNET-SW` needs no configuration.

## 4. Set up the servers and end devices

Servers have no CLI, so set them on the **Desktop > IP Configuration** and **Services** tabs.

| Device | IP address | Mask | Gateway | DNS | Services to turn on |
| --- | --- | --- | --- | --- | --- |
| CONTENT-SRV | 10.10.40.10 | 255.255.255.0 | 10.10.40.1 | 10.10.40.10 | HTTP on. DNS on, with the A records below |
| FILE-SRV | 10.10.40.11 | 255.255.255.0 | 10.10.40.1 | 10.10.40.10 | FTP on. Add user `health`, password `lokori123`, all permissions |
| INTERNET-SRV | 203.0.113.10 | 255.255.255.0 | 203.0.113.1 | 10.10.40.10 | HTTP on. Edit `index.html` to say "National KHIS server (simulated)" |
| HEALTH-PC, PRIMARY-PC, HIGH-PC, MARKET-LAPTOP | DHCP | | | | None |

DNS A records on CONTENT-SRV:

| Name | Address |
| --- | --- |
| content.lokori.local | 10.10.40.10 |
| files.lokori.local | 10.10.40.11 |
| khis.example.org | 203.0.113.10 |

Wait about 30 seconds after cabling for the switch ports to turn green, then click **DHCP** on each PC. Each one should get an address from its own VLAN.

## Address plan

| VLAN | Name | Subnet | Gateway | Used by |
| --- | --- | --- | --- | --- |
| 10 | HEALTH | 10.10.10.0/24 | 10.10.10.1 | Health facility |
| 20 | SCHOOLS | 10.10.20.0/24 | 10.10.20.1 | Primary and high school |
| 30 | HOTSPOT | 10.10.30.0/24 | 10.10.30.1 | Market, chief's office, mission |
| 40 | SERVERS | 10.10.40.0/24 | 10.10.40.1 | Content and file servers |
| 99 | MGMT | 10.10.99.0/24 | 10.10.99.1 | Switch management (.2 to .5) |
| none | Starlink link | 100.64.0.0/30 | | HUB-CORE .2, STARLINK-GW .1 |
| none | 4G link | 100.65.0.0/30 | | HUB-CORE .2, LTE-GW .1 |
| none | Simulated internet | 203.0.113.0/24 | | STARLINK-GW .1, LTE-GW .2, INTERNET-SRV .10 |

## 5. Packet transfer simulations

Switch to **Simulation** mode (bottom right). In **Edit Filters**, show only the protocols for the scenario you are running. Press **Play** or step with **Capture/Forward**, and click any envelope to see its headers at each layer.

### Scenario A: a school pupil opens the local content library (inter-VLAN HTTP)

1. Filters: DNS, HTTP, TCP, ARP.
2. On PRIMARY-PC, open **Desktop > Web Browser** and go to `content.lokori.local`.
3. **Expect:** a DNS query to 10.10.40.10, then a TCP handshake and HTTP from VLAN 20 to VLAN 40. The traffic goes PRIMARY-SW > HUB-SW > HUB-CORE (G0/0.20 in, G0/0.40 out) > HUB-SW > CONTENT-SRV. It never leaves the network, so it works even with no internet.

### Scenario B: the health facility moves a file to the file server (FTP)

1. Filters: FTP, TCP.
2. On HEALTH-PC, open **Desktop > Command Prompt** and run:

   ```
   ftp 10.10.40.11
   ```

   Log in as `health` / `lokori123`, then:

   ```
   dir
   get <one of the files listed>
   put <the same file name>
   quit
   ```

3. **Expect:** an FTP control session and data transfer between VLAN 10 and VLAN 40, routed by HUB-CORE.

### Scenario C: the health facility sends a report to the national server (NAT through Starlink)

1. Filters: ICMP, HTTP, TCP, DNS.
2. On HEALTH-PC, run `ping 203.0.113.10`, then browse to `khis.example.org`.
3. **Expect:** packets go HUB-CORE > STARLINK-GW > INTERNET-SRV. Click the packet at STARLINK-GW: the source address changes from 10.10.10.x to 203.0.113.1. Confirm on STARLINK-GW with:

   ```
   show ip nat translations
   ```

### Scenario D: a hotspot user is blocked from the health network (ACL)

1. Filters: ICMP.
2. On MARKET-LAPTOP, run `ping 10.10.10.21` (HEALTH-PC's DHCP address; check it first with `ipconfig` on HEALTH-PC).
3. **Expect:** the packet stops at HUB-CORE with a red X, dropped by `HOTSPOT-IN`.
4. Then browse from MARKET-LAPTOP to `content.lokori.local` and `khis.example.org`. Both load, because the hotspot may use the content server and the internet.
5. Check the counters on HUB-CORE with:

   ```
   show access-lists
   ```

### Scenario E: the satellite link fails and 4G takes over (failover)

1. On HUB-CORE, run `show ip route` and note the default route via 100.64.0.1.
2. Simulate the Starlink outage:

   ```
   configure terminal
   interface g0/1
   shutdown
   end
   ```

3. Run `show ip route` again: the default route is now via 100.65.0.1.
4. Filters: ICMP. Ping 203.0.113.10 from HIGH-PC. **Expect:** packets now leave through LTE-GW, and INTERNET-SRV sees the source as 203.0.113.2.
5. Restore the link with `no shutdown` on G0/1. Traffic returns to Starlink.

## Quick checks if something fails

| Symptom | Check |
| --- | --- |
| A PC gets a 169.254.x.x address | Its switch port VLAN (`show vlan brief`) and the trunk (`show interfaces trunk`) |
| Pings between VLANs fail | `show ip interface brief` on HUB-CORE: all subinterfaces should be up |
| Names do not resolve | DNS service is on in CONTENT-SRV, and the PC shows DNS 10.10.40.10 |
| The internet cannot be reached | `show ip route` on HUB-CORE, and `show ip nat translations` on the gateway in use |
