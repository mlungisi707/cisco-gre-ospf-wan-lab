# Enterprise WAN Architecture: GRE Tunneling over OSPF

![Network Topology Diagram](screenshots/01-gre-wan-ospf-topology.png)

## Project Overview
This project demonstrates the implementation and verification of a **GRE (Generic Routing Encapsulation) Tunnel** overlay across a simulated physical WAN serial link. The network simulates a production environment connecting a corporate **Headquarters (HQ)** and a remote **Branch Office**. Dynamic path discovery and routing table propagation are achieved by running **OSPFv2 (Open Shortest Path First)** directly over the virtual point-to-point tunnel interface.

This lab focuses on structural WAN deployment mechanics, interface configuration parsing, and core control-plane troubleshooting techniques required for network engineering roles.

**Tool:** Cisco Packet Tracer (`enterprise-wan-gre-ospf.pkt`)

---

## Production Deployment Context

### Why Engineers Use GRE Tunnels
* **Multicast Transport Support:** Unlike traditional point-to-point IPsec tunnels which natively drop multicast traffic, GRE encapsulates multicast streams. This allows dynamic routing protocols (like OSPF or EIGRP) to form adjacencies and exchange routing updates across the WAN automatically.
* **Virtual Point-to-Point Topology:** GRE simplifies complex public transport networks. It wraps payload packets inside a new delivery header, making geographically separated routers appear as though they are connected directly via a single virtual cable.

### Real-World Production Scenario
A retail corporation has its primary data center at the **Headquarters** and retail systems at a **Branch Office**. Both locations operate internal private subnets (`192.168.10.0/24` and `192.168.20.0/24`) which cannot traverse the public internet. By establishing a GRE tunnel across their public internet facing links, private corporate data is encapsulated at the local gateway, routed transparently over the public infrastructure, and decapsulated at the remote gateway for delivery.

GRE on its own does not encrypt or authenticate traffic. In production it is normally paired with IPsec (GRE over IPsec) when the underlying network is untrusted; this lab focuses on the tunnel and routing behaviour only.

---

## Topology & Network Addressing Scheme

### Physical Topology
* **Hosts:** 2 PCs at HQ, 2 PCs at Branch.
* **Gateways:** Cisco 819HGW Integrated Services Routers.
* **WAN Transport:** Point-to-point Serial connection between `Serial0` interfaces.

### IP Addressing Table

| Device | Interface | IP Address | Subnet Mask | Purpose / Description |
| :--- | :--- | :--- | :--- | :--- |
| **HQ-Router** | Serial0 | 203.0.113.1 | 255.255.255.252 | Public WAN Interface (DCE) |
| **HQ-Router** | Vlan1 | 192.168.10.1 | 255.255.255.0 | HQ Internal Default Gateway |
| **HQ-Router** | Tunnel0 | 10.0.0.1 | 255.255.255.252 | Virtual GRE Tunnel Interface |
| **Branch-Router**| Serial0 | 203.0.113.2 | 255.255.255.252 | Public WAN Interface (DTE) |
| **Branch-Router**| Vlan1 | 192.168.20.1 | 255.255.255.0 | Branch Internal Default Gateway |
| **Branch-Router**| Tunnel0 | 10.0.0.2 | 255.255.255.252 | Virtual GRE Tunnel Interface |
| **HQ-PC1** | FastEthernet0| 192.168.10.10 | 255.255.255.0 | Enterprise Internal Host |
| **HQ-PC2** | FastEthernet0| 192.168.10.20 | 255.255.255.0 | Enterprise Internal Host |
| **Branch-PC1** | FastEthernet0| 192.168.20.10 | 255.255.255.0 | Enterprise Internal Host |

---

## Configuration Implementation

### 1. HQ-Router Configuration
```cisco
! Configure Physical WAN Link
interface Serial0
 ip address 203.0.113.1 255.255.255.252
 clock rate 64000
 no shutdown
 exit

! Configure Internal LAN Gateway
interface Vlan1
 ip address 192.168.10.1 255.255.255.0
 no shutdown
 exit

! Establish Virtual GRE Tunnel Overlay
interface Tunnel0
 ip address 10.0.0.1 255.255.255.252
 tunnel source Serial0
 tunnel destination 203.0.113.2
 exit

! Configure OSPF Routing Control Plane
router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 10.0.0.0 0.0.0.3 area 0
 exit
```

### 2. Branch-Router Configuration
```cisco
! Configure Physical WAN Link
interface Serial0
 ip address 203.0.113.2 255.255.255.252
 no shutdown
 exit

! Configure Internal LAN Gateway
interface Vlan1
 ip address 192.168.20.1 255.255.255.0
 no shutdown
 exit

! Establish Virtual GRE Tunnel Overlay
interface Tunnel0
 ip address 10.0.0.2 255.255.255.252
 tunnel source Serial0
 tunnel destination 203.0.113.1
 exit

! Configure OSPF Routing Control Plane
router ospf 1
 network 192.168.20.0 0.0.0.255 area 0
 network 10.0.0.0 0.0.0.3 area 0
 exit
```

---

## Technical Command Reference

* `interface Tunnel0`: Instantiates a virtual, logical point-to-point interface within the Cisco IOS software execution space.
* `tunnel source Serial0`: Binds the tunnel initiation point to the specified physical exit interface configuration.
* `tunnel destination [IP]`: Defines the remote public address endpoint where the wrapped GRE transport envelope must be addressed.
* `router ospf [Process-ID]`: Initializes the OSPF link-state routing daemon in internal system memory.
* `network [Net-ID] [Wildcard-Mask] area [ID]`: Specifies which interfaces fall into the OSPF parsing scope. The wildcard mask acts as a precise evaluation filter (binary 0 forces verification, binary 1 bypasses it).

---

## Verification & Troubleshooting Methodology

This section outlines the deterministic verification architecture used to confirm data pathing and control plane state integrity. Each step includes the evidence captured from the lab.

### 1. Underlying WAN Transport Verification
Execute a layer 3 ICMP check across the physical link to validate underlying provider transit functionality.
```cisco
Branch-Router# ping 203.0.113.1
! Expected Result: Success rate is 100 percent (5/5)
```
![Branch-Router pings HQ WAN address](screenshots/03-branch-wan-ping.png)

* **Result:** 5/5 replies, round-trip min/avg/max = 1/22/45 ms. The underlay is healthy, which is the only requirement GRE has before the tunnel can come up.

### 2. GRE Tunnel Interface Verification
Confirm the tunnel interface is up and bound to the correct source and destination.
```cisco
HQ-Router# show interface Tunnel0
```
![HQ-Router Tunnel0 status](screenshots/02-hq-tunnel-status.png)

* **Result:** `Tunnel0 is up, line protocol is up`, tunnel source 203.0.113.1 (Serial0), destination 203.0.113.2, tunnel protocol/transport **GRE/IP**. The tunnel transport MTU is 1476 bytes, which leaves room for the GRE and outer IP headers (24 bytes) on a 1500 byte path. Keepalives are not set.

### 3. Control Plane Verification (OSPF Adjacency Status)
Confirm that the dynamic routing protocol has established a stable state across the logical GRE infrastructure.
```cisco
Branch-Router# show ip ospf neighbor
```
![Branch-Router OSPF neighbor](screenshots/04-branch-ospf-neighbor.png)

* **Analysis Criteria:** The neighbor state must transition to **FULL**. This indicates complete link-state database synchronization has occurred through the GRE tunnel layer.
* **Result:** Neighbor ID 203.0.113.1 (HQ-Router) is `FULL/ -` on **Tunnel0** via 10.0.0.1. Priority 0 and no DR/BDR role is expected, because the tunnel is treated as a point-to-point network.

### 4. RIB (Routing Information Base) Inspection
Verify that remote enterprise LAN prefixes are actively populated in the routing table database.
```cisco
HQ-Router# show ip route
```
![HQ-Router routing table](screenshots/05-hq-routing-table.png)

* **Analysis Criteria:** The table must display a prefix designated with the identifier **O** (OSPF), pointing explicitly out of the logical interface `Tunnel0`.
* **Result:** `O    192.168.20.0/24 [110/1001] via 10.0.0.2, 00:10:55, Tunnel0`. The metric of 1001 is the tunnel's default 100 Kbit/s bandwidth (cost 1000) plus 1 for the Branch LAN interface. The connected routes for 10.0.0.0/30 (Tunnel0), 192.168.10.0/24 (Vlan1) and 203.0.113.0/30 (Serial0) are also present.

### 5. End-to-End Enterprise Datapath Validation
Initiate an ICMP flow directly from host workstation **HQ-PC1** to host workstation **Branch-PC1** across the production boundary.
```cmd
C:\> ping 192.168.20.10
! Expected Result: Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```
![HQ-PC1 pings Branch-PC1](screenshots/06-pc-end-to-end-ping.png)

* **Result:** 4/4 replies, 0% loss. The reply TTL of 126 shows the packet crossed two routers (the PC starts at 128), confirming the traffic is routed through HQ-Router and Branch-Router.

### 6. GRE Encapsulation Analysis (Packet Tracer Simulation Mode)
Simulation mode shows the tunnel at packet level. The capture below is taken at HQ-Router for a reply travelling from the Branch back to HQ-PC2.

![GRE PDU encapsulation at HQ-Router](screenshots/07-gre-pdu-encapsulation.png)

* **In Layers (WAN side):** The frame arrives on `Serial0` as an HDLC frame carrying the outer delivery header, source 203.0.113.2 to destination 203.0.113.1. This is the GRE packet crossing the WAN.
* **Out Layers (LAN side):** After the outer header is removed, the original packet leaves `FastEthernet1` in a new Ethernet II frame, with the inner header 192.168.20.10 to 192.168.10.20 (the real host addresses).
* **Takeaway:** The WAN only ever sees the 203.0.113.x addresses, while the private LAN addresses travel inside the tunnel.

---

## WAN Architecture Troubleshooting Framework

When explaining this architecture during technical recruitment reviews, the diagnostic procedure is broken down systematically across the layers:

1. **Layer 1 & 2 Verification:** Run `show interface Serial0` to verify the hardware and line protocols are up/up. If down, troubleshoot physical cabling, clocking configurations, or public carrier outages.
2. **Layer 3 Tunnel Overlay Validation:** Execute `show interface Tunnel0`. Verify that the tunnel source and destination metrics match exactly on both peers. Ensure that firewall ACLs (Access Control Lists) on the public interface are not dropping IP Protocol 47 (GRE traffic).
3. **Control Plane Diagnosis:** If the virtual tunnel interface is up but end-to-end user communication fails, analyze the control plane via `show ip ospf neighbor`. If stuck in a state other than FULL, check for mismatched OSPF areas, subnet mask conflicts on the tunnel interface, or hello/dead interval discrepancies.

### Observed Behaviour
During testing, the OSPF adjacency dropped to DOWN when `Serial0` went down and returned to FULL shortly after the serial link came back. This shows that the tunnel and its OSPF neighbour depend entirely on the health of the underlying WAN path.

---

## Repository Contents

| File | Description |
| :--- | :--- |
| `enterprise-wan-gre-ospf.pkt` | Packet Tracer lab file |
| `screenshots/` | Numbered evidence screenshots (01 to 07) |
