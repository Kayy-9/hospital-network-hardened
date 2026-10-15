# Project: Network Hardening (Cisco Packet Tracer)

Hardening the simulated hospital network from [Project 1](../hospital-network-packet-tracer). The original design had working VLANs, inter-VLAN routing, and guest isolation. This project adds a management VLAN, encrypted remote access, port security, and other protections, then tests that the network still works and that the blocks still hold.

> This is a lab simulation. The password used is a throwaway lab value, and no real systems or data are involved.

---

## Goals

- Give device administration its own isolated network.
- Replace open access to the switch and router with authenticated, encrypted (SSH) access, limited to one management PC.
- Stop unknown devices from using the switch's access ports.
- Close smaller gaps (trunk negotiation, spanning-tree tampering, information leaks).
- Re-test everything and document what I found.

## Starting point

The finished Project 1 network (`hospital-network.pkt`), saved as a separate copy (`hospital-network-hardened.pkt`) so the original stays intact.

| VLAN | Name | Subnet |
|------|------|--------|
| 10 | ADMIN | 10.10.10.0/24 |
| 20 | DOCTORS | 10.10.20.0/24 |
| 30 | NURSES | 10.10.30.0/24 |
| 40 | GUEST | 10.10.40.0/24 |
| 50 | SERVERS | 10.10.50.0/24 |
| **99** | **MANAGEMENT (new)** | **10.10.99.0/24** |

New devices:

| Device | Address | Connected to |
|--------|---------|--------------|
| `MGMT-PC` | 10.10.99.10 (gateway 10.10.99.1) | `HOSP-SW1` `Fa0/6` |
| `HOSP-SW1` management interface | 10.10.99.2 | VLAN 99 |
| `HOSP-R1` subinterface `gig0/0.99` | 10.10.99.1 | Trunk |
| `ROGUE-PC` (test only) | 10.10.10.99 | Temporarily on `Fa0/1` |

---

## What I changed

### 1. Management VLAN

```
! Switch
vlan 99
 name MANAGEMENT
interface vlan 99
 ip address 10.10.99.2 255.255.255.0
 no shutdown
ip default-gateway 10.10.99.1
interface fa0/6
 no shutdown
 switchport mode access
 switchport access vlan 99
interface gig0/1
 switchport trunk allowed vlan 10,20,30,40,50,99

! Router
interface gig0/0.99
 encapsulation dot1q 99
 ip address 10.10.99.1 255.255.255.0
```

Limiting the trunk's allowed VLANs means no other VLAN can cross it.

### 2. Blocking the guest network from the management VLAN

```
ip access-list extended GUEST-RESTRICT
 45 deny ip 10.10.40.0 0.0.0.255 10.10.99.0 0.0.0.255
```

Rule `45` sits before the final `permit ip any any` on line 50. ACLs are read top to bottom, so a rule placed after the `permit` would never match.

### 3. Secure logins and SSH

```
service password-encryption
enable secret <lab password>
username admin secret <lab password>
ip domain-name hospital.local
crypto key generate rsa        (1024-bit modulus)
ip ssh version 2
ip access-list standard MGMT-ONLY
 permit host 10.10.99.10
 deny any log
line vty 0 15
 login local
 transport input ssh
 access-class MGMT-ONLY in
 exec-timeout 5 0
line console 0
 login local
 exec-timeout 5 0
banner motd #Authorized access only. Activity is logged.#
```

Remote access is SSH only (no Telnet), requires a username and password, is limited to `MGMT-PC`, and idle sessions end after 5 minutes. The same configuration was applied to the switch and the router.

### 4. Port security

```
interface range fa0/1-6
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
```

Each access port learns the first device that sends traffic and shuts itself down if a different one appears. The trunk (`Gig0/1`) is left out, since port security is meant for access ports.

### 5. Smaller protections

```
! Switch
interface range fa0/1-6
 spanning-tree portfast
 spanning-tree bpduguard enable
 no cdp enable
interface gig0/1
 switchport nonegotiate

! Router
no ip http server
no ip http secure-server
interface gig0/0
 no cdp enable
```

- **PortFast with BPDU guard:** An access port should only ever have an end device. If someone connects a switch, the port shuts down, so nobody can tamper with the spanning-tree topology.
- **CDP off on access ports:** CDP advertises the device model and software version to whatever is plugged in.
- **`nonegotiate` on the trunk:** Stops a rogue device from negotiating its way into a trunk.
- **Web management off on the router:** Unused, since management is over SSH.

---

## Testing and results

Tests were run from each PC's command prompt. A first ping that times out while ARP resolves is normal and was repeated.

| Test | Expected | Result |
|------|----------|--------|
| Internal VLANs (Admin, Doctors, Nurses) reach the servers | Replies | Pass |
| `MGMT-PC` reaches the router (10.10.99.1) and switch (10.10.99.2) | Replies | Pass |
| `GUEST-PC` reaches Admin, Servers, management | Blocked | Pass |
| SSH from `MGMT-PC` to the switch | Login succeeds | Pass |
| SSH from `ADMIN-PC` to the switch | Refused | Pass |
| Unknown device on a secured port (`Fa0/1`) | Port shuts down | Pass |
| Configuration review | Hashed passwords, SSH v2, allowed VLANs limited | Pass |

### Details worth recording

**A gap I found before fixing it.** After creating the management VLAN, I tested from `GUEST-PC` and it could reach the router's management address (`10.10.99.1`) and the switch (`10.10.99.2`). The Project 1 ACL only listed the four original subnets and ended with `permit ip any any`, so the new VLAN was open to guests. Adding rule 45 fixed it, and the same pings were then blocked while `MGMT-PC` could still reach both addresses. `TTL=254` on the switch replies showed the traffic had been routed through the router, which confirmed the guest could really cross VLANs.

**Port security.** After connecting `ROGUE-PC` to `Fa0/1`, the port showed `Secure-shutdown`, a violation count of 1, and `err-disabled`. The violation's source MAC was different from the sticky MAC learned from `ADMIN-PC`. To recover, I reconnected `ADMIN-PC` and ran `shutdown` then `no shutdown` on the port. It worked immediately, because the sticky entry still held `ADMIN-PC`'s MAC.

**Trunk.** `show interfaces gig0/1 switchport` showed `Negotiation of Trunking: Off` and trunking VLANs `10,20,30,40,50,99`, and the trunk stayed up.

**SSH.** `show ip ssh` reported SSH enabled, version 2.0, with a 120-second authentication timeout and 3 retries. The login banner displayed before the password prompt.

**Passwords.** The running configuration showed `enable secret` and the `admin` user as `secret 5` hashes, with `service password-encryption` on, so no password is stored in plain text.

---

## Troubleshooting log

| Problem | Cause | Fix |
|---------|-------|-----|
| Guest could ping the management VLAN | The existing guest ACL didn't cover the new subnet | Added rule 45 before the final `permit` |
| BPDU guard didn't appear in `show spanning-tree interface fa0/2 detail` | This Packet Tracer view shows PortFast but didn't list BPDU guard | Checked the running configuration for the `bpduguard` line instead |
| `Fa0/1` went `err-disabled` during the rogue test | Intended behaviour of port security | `shutdown` then `no shutdown` after reconnecting the original device |

---

## What I learned

- A new network segment starts with no protection. Adding the management VLAN opened a gap that the existing ACL didn't cover, so every change needs its own before-and-after test.
- ACL rule order matters, and a numbered rule can be inserted in the right place instead of rebuilding the list.
- Port security with sticky MAC addresses protects ports with almost no manual work, but recovering a shut-down port is a manual step.
- A feature that is configured may not show up in every `show` command, so check more than one source (`show` output and the running configuration).

## Limitations

- **Management VLAN reachability.** Only guests are blocked from the management VLAN at layer 3. Other internal VLANs can still ping `10.10.99.1` and `10.10.99.2`. Only SSH login is restricted to `MGMT-PC`. A stricter design would also block the other VLANs from the management subnet.
- **Sticky port security can be bypassed** by a rogue device that copies (spoofs) the real device's MAC address. It stops casual plug-ins and is one layer of a defence, not a complete one.
- **Password hashing.** `secret 5` is the older MD5-based type that this Packet Tracer version uses. Newer devices support stronger types.
- **Shared password.** The `enable secret` and the `admin` user use the same lab password. Production should use separate, unique credentials.
- **Small RSA key.** The 1024-bit key is the lab default. Modern guidance recommends 2048 bits or more.
- **No central logging or AAA.** There is no log server or RADIUS/TACACS+ in this lab, so the "Activity is logged" banner isn't backed by a log collection system.
- **ACLs are stateless**, as in Project 1.
- **Native VLAN left as VLAN 1** on the trunk, to avoid a mismatch with the router subinterfaces.

## Possible next steps

- Block all other VLANs from the management subnet except `MGMT-PC`.
- Add a syslog server and send logs from the switch and router to it.
- Use AAA with a RADIUS server for logins.
- Move unused ports to a dead-end VLAN.
- Add DHCP snooping and Dynamic ARP Inspection.

---

## Files

- `hospital-network-hardened.pkt`: the Packet Tracer file with all changes applied

*This builds on Project 1 (secure hospital network design).*
