# Prerequisites and General Information

The build guide created for the fact that a pfSense firewall manages the network traffic between Operator, Redirectors and GOAD VMs instead of using Ludus and Wireguard.  
Proxmox host NIC should be in a trunk port in order the lab VMs have their each VLAN tags assigned in proxmox network bridge when we create them.  
Current setup have the Proxmox host NIC in a Management-LAN trunk port connected via the Layer 2 VLAN capable switch (as shown in the diagram) and is at the same subnet with pfSense firewall and the switch.  
This way you can reach the management UI of all network appliances and proxmox host from your Desktop/Laptop with a rule allowing the traffic to LAN subnet from your Desktop/Laptop VLAN, which is convenient for troubleshooting and setup.

---

## 1.1 Create the VLAN interfaces

Interfaces -> Assignments -> VLANs -> Add
Do this four times, creating the below

| Parent interface | VLAN tag | Description |
| ---------------- | -------- | ----------- |
| Lan NIC          | 150      | OPS_C2      |
| Lan NIC          | 160      | REDIRECTORS |
| Lan NIC          | 56       | AD_LAB      |
| Lan NIC          | 100      | HOME        |

The parent interface is the physical pfSense NIC carrying the tagged trunk to your Proxmox mini PC. All four VLANs ride the same trunk.

---

## 1.2 Assign and configure the interfaces

Interfaces -> Assignments -> Interface Assignments
Add each VLAN as a new interface and Save.  
  
Then open each interface, set the below, save and appy changes :

| VLAN - Description | Enable interface | IPv4 Configuration Type | IPv6 Configuration Type | IPv4 Address    |
| ------------------ | ---------------- | ----------------------- | ----------------------- | --------------- |
| OPC_C2             | Check Enable     | Static IPv4             | None                    | 172.23.150.1/24 |
| REDIRECTORS        | Check Enable     | Static IPv4             | None                    | 10.60.160.1/24  |
| AD_LAB             | Check Enable     | Static IPv4             | None                    | 192.168.56.1/24 |
| HOME               | Check Enable     | Static IPv4             | None                    | 10.10.100.1/24  |

GOAD hosts use the domain controllers as their DNS (kingslanding, .10), required for AD. pfSense is their gateway only.
Keep the exact IPv4 Addresses for the AD_LAB, as the DHCP used by the GOAD lab when it will be provisioned is using this subnet for the GOAD lab VMs static IP configuration.  
All other subnets can be changed by your choice.

---

## 1.3 DHCP

Services -> DHCP Server:

- OPS_C2: Check Enable DHCP, Address Pool range: 172.23.150.100 to 172.23.150.120.  Leave .10 to .99 free for the reservations.
- REDIRECTORS: Check Enable DHCP, Address Pool range: 10.60.160.100 to 10.60.160.120. Leave .10 to .99 free for the reservations.
- AD_LAB: Check Enable DHCP (we will disable for the finished lab. GOAD assigns static IPs itself. A temporary exception applies only during Packer template builds, covered when that Part comes up), Address Pool range: 192.168.56.100 to 192.168.56.120. This is needed for the Ubuntu provisioning VM to get an IP when is created.
- HOME: Check Enable DHCP, Address Pool range: 10.10.100.100 to 10.10.100.120.  Optional: Create a reservation for you Desktop/Laptop.

After the C2 and redirector VMs exist (Part 3), add DHCP static mappings by MAC so Kali, Sliver and both redirectors keep fixed addresses:
- Kali: 172.23.150.10
- Sliver: 172.23.150.20
- HTTP/S redirector: 10.60.160.10
- DNS/SSH redirector: 10.60.160.20

---

## 1.4 Firewall aliases

Optional, but useful for the rules we will create.
Firewall -> Aliases -> IP ->  Add:

- RFC1918 (Private networks) - Name: RFC1918 , Description: Internal Networks, Type: Networks, Add: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, Save.
- DESKTOP (Host) - Name: DESKTOP, Description: Desktop, Type: Host, Add: Desktop IP addres and a hostname. Save

---

## 1.5 Firewall rules

Rules are evaluated top down, first match wins, applied on the interface where traffic enters.  
"Real LAN" is covered by the RFC1918 alias minus the specific allows above each block.

### AD_LAB (VLAN 56) tab, most locked down

Goal: VLAN 56 initiates nothing toward VLANs 150, 160, 100 and LAN or WAN. Inbound traffic to VLAN 56 is allowed by the other interfaces rules.

1. **Build time only:** A rule to allow AD_LAB subnets to WAN, any port. Not just 443. The provisioning VM needs DNS/53, HTTP/80 for apt mirrors and HTTPS/443 during install. Remove after both templates are built. You can create a more restrictive rule if you want, instead of **any**.  Place at the top.
2. A rule to block AD_LAB net to RFC1918 (alias). This kills VLAN 56 to VLANs 150, 160, 100 and to LAN in one rule. Enable logging.
3. Optionally allow AD_LAB net to 192.168.56.1 (gateway) for DNS/ICMP to pfSense. The internal DNS is the DCs, not pfSense, so this can be omitted.
4. Implicit deny handles everything else, including WAN, once no allow matches.

If you do not use build time internet on VLAN 56, delete rule 1 and pre-stage packages instead.

### Outbound NAT, required for the build time WAN rule to work

A firewall allow rule alone is not enough for VLAN 56 hosts to reach the internet. pfSense also needs to NAT the 192.168.56.0/24 subnet on the WAN interface, or replies have nowhere to route back to and connections hang.

Firewall, NAT, Outbound.

- If mode is Automatic, check whether a rule already covers 192.168.56.0/24 on WAN. On a manually added VLAN it often does not.
- Switch to Hybrid outbound NAT, save, then add a manual rule.
  - Interface: WAN
  - Source: Network, 192.168.56.0/24
  - Destination: any
  - Translation: WAN address
  - Description: NAT for VLAN 56 AD_LAB, build time
- Apply changes.

Do this before Part 4. It only matters while the build time WAN allow rule is active. Remove both together once both templates are built and the full install is verified.

---

### REDIRECTORS (VLAN 160) tab

Goal: VLAN 160 to VLAN 56 freely (since that is their job). VLAN 160 to VLAN 150 limited and logged, nothing to VLAN 100 or LAN.

1. A rule to allow REDIRECTORS subnet to AD_LAB subnet (192.168.56.0/24), any port. Description: Allow redirectors to targets.
2. A rule to allow and log REDIRECTORS subnet to OPS_C2 subnet, destination port SLIVER_LISTENERS (create alias) only. Description: Redirected sessions back to C2.
3. A rule to block and log REDIRECTORS subnet to RFC1918 (alias). This catches VLAN 160 to 100, to LAN and any VLAN 160 to VLAN 150 not on the allowed listener ports.
4. A rule to allow REDIRECTORS subnet to WAN if your redirectors need to fetch tooling or updates. You can have this rule disabled and enable ony when updating or installing tools.
5. Implicit deny.

---

### OPS_C2 (VLAN 150) tab

Goal: reachable from your desktop, reaches VLAN 56 both directly and via VLAN 160, no "Real LAN" access.

1. A rule to allow OPS_C2 subnet to AD_LAB net (192.168.56.0/24), any. Description: Direct operator path to lab. This is optional, fast path for testing without redirectors.
2. A rule to allow OPS_C2 subnet to REDIRECTORS subnet, any. Description: Operator path via redirectors.
3. A rule to block and log OPS_C2 subnet to RFC1918. This kills VLAN 150 to VLAN 100 and to LAN. The two allows above already matched VLANs 56 and 160.
4. A rule to allow OPS_C2 subnet to WAN for tooling and updates.As above, you can have this rule disabled and enable ony when updating or installing tools.
5. Implicit deny.

---

### VLAN 100 (your desktop) tab, add rules for lab access

On your existing VLAN 100 interface, add these without removing your existing rules.

1. A rule to allow DESKTOP (alias) to OPS_C2 subnet, any. RDP/SSH/web to Kali and Sliver.
2. A rule to allow DESKTOP (alias) to AD_LAB subnet (192.168.56.0/24), any. Direct RDP into DCs for lab admin.
3. A rule to allow DESKTOP (alias) to REDIRECTORS subnet, any. Configure the redirectors.

Return traffic for all of the above is handled automatically by pfSense's stateful engine.

Save and Apply all changes.
