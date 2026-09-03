# NexoTech Solutions — Segmented Network Design

A Cisco Packet Tracer project applying what I've learned through the Cisco Networking Academy curriculum to a realistic scenario — VLANs, inter-VLAN routing, DHCP, ACLs, and NAT, all built for a fictional IT consultancy I called NexoTech Solutions.

## About NexoTech

NexoTech is a small IT consultancy (~30-40 employees) that builds custom software for clients and handles some managed IT support. I picked this scenario on purpose — a company like this handles client data, runs internal servers, and has developers writing code, so there's a real reason to segment the network properly rather than just adding VLANs for the sake of it.

## Topology

![Topology](https://github.com/user-attachments/assets/81be82e4-c47f-4e18-b1e2-6c288db12851)

## VLAN Design

| VLAN | Department | Subnet |
|---|---|---|
| 10 | Management/Admin | 10.0.10.0/24 |
| 20 | IT/Sysadmin | 10.0.20.0/24 |
| 30 | Development | 10.0.30.0/24 |
| 40 | Servers/DevOps | 10.0.40.0/24 |
| 50 | Sales & Support | 10.0.50.0/24 |
| 60 | Guest/Wi-Fi | 10.0.60.0/24 |

IT gets access to everything since they run the network. Dev needs Servers (to deploy/test code) but nothing from Sales or Management. Servers is the most locked-down VLAN, since it holds client-related infrastructure. Sales stays cut off from Dev and Servers entirely — if a Sales laptop got phished, there's nowhere sensitive for it to reach. Guest is fully isolated except for internet access, same as any office Wi-Fi.

## Access Control

Built on least privilege — each department only reaches what it actually needs — so that if one VLAN gets compromised, the damage doesn't spread.

**Management** has access to IT, Dev, Servers, and Sales. Blocked from Guest.

**IT** has access to everything (Mgmt, Dev, Servers, Sales) since they administer the whole network. Blocked from Guest.

**Development** has access to Servers (to deploy and test code) and IT (for support). Blocked from Management and Sales.

**Servers** can be reached by Mgmt, IT, and Dev. Blocked from initiating traffic toward Sales or Guest.

**Sales** has access to IT (for support). Blocked from Management, Dev, and Servers.

**Guest** has internet access only, through NAT. Blocked from every internal VLAN, in both directions — no internal VLAN can reach into Guest either.

**Known limitation:** ideally Servers shouldn't be able to *initiate* connections toward Dev/Mgmt/Sales, only reply to ones they start. I didn't implement this — standard Cisco ACLs can't tell a new connection from a reply, so blocking it outright would've broken the legitimate Dev → Servers access. Properly solving this needs a reflexive ACL or a stateful firewall, which is beyond what I've covered so far.

## How It's Built

- **Routing:** router-on-a-stick — one trunk link, one sub-interface per VLAN (802.1Q)
- **DHCP:** one pool per VLAN on the router, first 10 addresses in each subnet reserved
- **ACLs:** extended ACLs applied inbound on each restricted VLAN's sub-interface
- **NAT:** PAT/overload on the router's outside interface, so every VLAN shares one public IP for internet access

## Testing It

Same-VLAN and inter-VLAN pings worked as expected once routing was up, confirming router-on-a-stick was working correctly.

![Same-VLAN ping](https://github.com/user-attachments/assets/3fdd9f9d-e120-4bd5-ad7e-c7e5be7bf341)
![Inter-VLAN ping](https://github.com/user-attachments/assets/ebb25844-8d27-4d73-aca1-8575b1257ce6)

Allowed paths worked, and blocked ones actually got blocked: Dev and Mgmt could both reach Servers, Sales could reach IT but was blocked from Servers, and Dev was blocked from Mgmt.

![Sales blocked from Servers](https://github.com/user-attachments/assets/eb64c2e1-ccac-4170-b08a-5686b345b368)
![Dev blocked from Mgmt](https://github.com/user-attachments/assets/2ca64660-783d-4fef-b937-ee1325a3f166)

Guest isolation was the strictest test, and it held — 100% packet loss trying to reach any internal VLAN, in both directions (Mgmt/IT also blocked from reaching into Guest).

![Guest blocked from Servers](https://github.com/user-attachments/assets/1802ee79-4850-4228-872b-8bdcd5a46fb3)

Guest could still reach the internet through NAT, even while fully cut off internally — exactly the behavior you'd want from a real guest Wi-Fi setup.

![Guest reaching internet](https://github.com/user-attachments/assets/d78edae7-8d56-4401-b88a-bb95509fbb37)

I confirmed NAT was actually translating traffic (not just routing it) with `show ip nat translations` — internal addresses like `10.0.10.11` and `10.0.60.12` showed up correctly mapped to the outside address `203.0.113.2`.

![NAT translation table](https://github.com/user-attachments/assets/30aea037-70aa-4fc9-a45a-f0f38a203d89)

## Repo Contents

```
/configs        router and switch running-configs
/screenshots    connectivity + ACL test results
nexotech-network.pkt   full Packet Tracer project file
```

The `.pkt` file is included so anyone can open the actual working network — not just read about it. Free to open with [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) (requires a Cisco Networking Academy account).

## Router Configuration

<details>
<summary>Click to expand</summary>

```
version 15.1
no service timestamps log datetime msec
no service timestamps debug datetime msec
no service password-encryption
!
hostname Router
!
ip dhcp excluded-address 10.0.10.1 10.0.10.10
ip dhcp excluded-address 10.0.20.1 10.0.20.10
ip dhcp excluded-address 10.0.30.1 10.0.30.10
ip dhcp excluded-address 10.0.40.1 10.0.40.10
ip dhcp excluded-address 10.0.50.1 10.0.50.10
ip dhcp excluded-address 10.0.60.1 10.0.60.10
!
ip dhcp pool MGMT_POOL
 network 10.0.10.0 255.255.255.0
 default-router 10.0.10.1
 dns-server 8.8.8.8
ip dhcp pool IT_POOL
 network 10.0.20.0 255.255.255.0
 default-router 10.0.20.1
 dns-server 8.8.8.8
ip dhcp pool DEV_POOL
 network 10.0.30.0 255.255.255.0
 default-router 10.0.30.1
 dns-server 8.8.8.8
ip dhcp pool SERVERS_POOL
 network 10.0.40.0 255.255.255.0
 default-router 10.0.40.1
 dns-server 8.8.8.8
ip dhcp pool SALES_POOL
 network 10.0.50.0 255.255.255.0
 default-router 10.0.50.1
 dns-server 8.8.8.8
ip dhcp pool GUEST_POOL
 network 10.0.60.0 255.255.255.0
 default-router 10.0.60.1
 dns-server 8.8.8.8
!
interface GigabitEthernet0/0
 no ip address
 duplex auto
 speed auto
!
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.0.10.1 255.255.255.0
 ip access-group MGMT_RESTRICT in
 ip nat inside
!
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 10.0.20.1 255.255.255.0
 ip access-group IT_RESTRICT in
 ip nat inside
!
interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 10.0.30.1 255.255.255.0
 ip access-group DEV_RESTRICT in
 ip nat inside
!
interface GigabitEthernet0/0.40
 encapsulation dot1Q 40
 ip address 10.0.40.1 255.255.255.0
 ip nat inside
!
interface GigabitEthernet0/0.50
 encapsulation dot1Q 50
 ip address 10.0.50.1 255.255.255.0
 ip access-group SALES_RESTRICT in
 ip nat inside
!
interface GigabitEthernet0/0.60
 encapsulation dot1Q 60
 ip address 10.0.60.1 255.255.255.0
 ip access-group GUEST_RESTRICT in
 ip nat inside
!
interface GigabitEthernet0/1
 ip address 203.0.113.2 255.255.255.252
 ip nat outside
 duplex auto
 speed auto
!
ip nat inside source list NAT_ACL interface GigabitEthernet0/1 overload
ip classless
!
ip access-list extended SALES_RESTRICT
 deny ip 10.0.50.0 0.0.0.255 10.0.10.0 0.0.0.255
 deny ip 10.0.50.0 0.0.0.255 10.0.30.0 0.0.0.255
 deny ip 10.0.50.0 0.0.0.255 10.0.40.0 0.0.0.255
 permit ip any any
ip access-list extended DEV_RESTRICT
 deny ip 10.0.30.0 0.0.0.255 10.0.10.0 0.0.0.255
 deny ip 10.0.30.0 0.0.0.255 10.0.50.0 0.0.0.255
 permit ip any any
ip access-list extended GUEST_RESTRICT
 deny ip 10.0.60.0 0.0.0.255 10.0.10.0 0.0.0.255
 deny ip 10.0.60.0 0.0.0.255 10.0.20.0 0.0.0.255
 deny ip 10.0.60.0 0.0.0.255 10.0.30.0 0.0.0.255
 deny ip 10.0.60.0 0.0.0.255 10.0.40.0 0.0.0.255
 deny ip 10.0.60.0 0.0.0.255 10.0.50.0 0.0.0.255
 permit ip any any
ip access-list extended MGMT_RESTRICT
 deny ip 10.0.10.0 0.0.0.255 10.0.60.0 0.0.0.255
 permit ip any any
ip access-list extended IT_RESTRICT
 deny ip 10.0.20.0 0.0.0.255 10.0.60.0 0.0.0.255
 permit ip any any
ip access-list standard NAT_ACL
 permit 10.0.10.0 0.0.0.255
 permit 10.0.20.0 0.0.0.255
 permit 10.0.30.0 0.0.0.255
 permit 10.0.40.0 0.0.0.255
 permit 10.0.50.0 0.0.0.255
 permit 10.0.60.0 0.0.0.255
!
end
```

</details>

## Switch Configuration

<details>
<summary>Click to expand</summary>

```
version 15.0
no service timestamps log datetime msec
no service timestamps debug datetime msec
no service password-encryption
!
hostname Switch
!
spanning-tree mode pvst
spanning-tree extend system-id
!
interface FastEthernet0/1
 switchport trunk allowed vlan 10,20,30,40,50,60
 switchport mode trunk
!
interface FastEthernet0/2
 switchport access vlan 10
 switchport mode access
!
interface FastEthernet0/3
 switchport access vlan 10
 switchport mode access
!
interface FastEthernet0/4
 switchport access vlan 20
 switchport mode access
!
interface FastEthernet0/5
 switchport access vlan 20
 switchport mode access
!
interface FastEthernet0/6
 switchport access vlan 30
 switchport mode access
!
interface FastEthernet0/7
 switchport access vlan 30
 switchport mode access
!
interface FastEthernet0/8
 switchport access vlan 40
 switchport mode access
!
interface FastEthernet0/9
 switchport access vlan 40
 switchport mode access
!
interface FastEthernet0/10
 switchport access vlan 50
 switchport mode access
!
interface FastEthernet0/11
 switchport access vlan 50
 switchport mode access
!
interface FastEthernet0/12
 switchport access vlan 60
 switchport mode access
!
interface FastEthernet0/13
 switchport access vlan 60
 switchport mode access
!
interface FastEthernet0/14
 switchport access vlan 40
 switchport mode access
!
end
```

</details>

## Author

Siyanda Maliwa — BTech Computer Engineering student, Cape Peninsula University of Technology
