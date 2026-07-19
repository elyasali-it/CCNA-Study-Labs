# Part V — Device Configs

These are the configs from Chapters 17–19 — static routes, VLAN routing, and DHCP relay. I keep them separate from the main README here so the writeup stays readable and the actual CLI is easy to find and copy when I want to reference it.

## Ch 17 — Static Routes

Basic IP addressing plus a static route pointing to a remote subnet, first by next-hop address, then by outgoing interface as a comparison:

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 10.1.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# interface Serial0/0/0
R1(config-if)# ip address 10.1.4.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
!
! Static route to a remote subnet using the next-hop router's IP
R1(config)# ip route 10.1.2.0 255.255.255.0 10.1.4.2
!
! Same destination, this time using the outgoing interface instead
R1(config)# ip route 10.1.2.0 255.255.255.0 Serial0/0/0
```

Default route (the catch-all for anything not matched by a more specific route):

```
R1(config)# ip route 0.0.0.0 0.0.0.0 10.1.4.2
```

Host route (a single `/32` address routed differently than the rest of its subnet):

```
R1(config)# ip route 10.1.2.50 255.255.255.255 10.1.4.2
```

Floating static route (higher administrative distance so it only activates if the primary — usually dynamic — route disappears; default AD for a normal static route is 1, so anything higher than whatever routing protocol you're comparing against will "float" until it's needed):

```
R1(config)# ip route 10.1.2.0 255.255.255.0 Cellular0/1/0 130
```

## Ch 18 — VLAN Routing

### Router-on-a-Stick (ROAS)

One trunked physical interface, two subinterfaces, one per VLAN. VLAN 10 configured as the native VLAN using the second of the two supported methods (native keyword on the subinterface, rather than putting the native VLAN's IP on the bare physical interface):

```
R1(config)# interface GigabitEthernet0/0.10
R1(config-subif)# encapsulation dot1Q 10 native
R1(config-subif)# ip address 10.1.10.1 255.255.255.0
R1(config-subif)# exit
R1(config)# interface GigabitEthernet0/0.20
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip address 10.1.20.1 255.255.255.0
R1(config-subif)# exit
R1(config)# interface GigabitEthernet0/0
R1(config-if)# no shutdown
```

### Layer 3 Switch with SVIs

Enable routing globally, then give each VLAN its own routed interface:

```
SW1(config)# ip routing
SW1(config)# interface vlan 10
SW1(config-if)# ip address 10.1.10.1 255.255.255.0
SW1(config-if)# no shutdown
SW1(config-if)# exit
SW1(config)# interface vlan 20
SW1(config-if)# ip address 10.1.20.1 255.255.255.0
SW1(config-if)# no shutdown
SW1(config-if)# exit
SW1(config)# interface vlan 30
SW1(config-if)# ip address 10.1.30.1 255.255.255.0
SW1(config-if)# no shutdown
```

### Layer 3 Switch — Routed Port

Strip the switchport role off a physical interface and configure it like a router interface. Makes sense for a point-to-point link between two Layer 3 switches instead of burning a whole VLAN on it:

```
SW1(config)# interface GigabitEthernet1/0/9
SW1(config-if)# no switchport
SW1(config-if)# ip address 10.1.30.1 255.255.255.0
```

### Layer 3 EtherChannel

Two physical routed ports bundled into one PortChannel interface, IP address configured only on the PortChannel — IOS auto-adds `no ip address` to the physical members once they join the channel:

```
SW1(config)# interface TenGigabit1/1/1
SW1(config-if)# no switchport
SW1(config-if)# channel-group 12 mode on
SW1(config)# interface TenGigabit1/1/2
SW1(config-if)# no switchport
SW1(config-if)# channel-group 12 mode on
SW1(config)# interface Port-channel12
SW1(config-if)# no switchport
SW1(config-if)# ip address 10.1.12.1 255.255.255.0
```

## Ch 19 — DHCP

Relay agent config on a router's LAN-facing interface, pointing DHCP broadcasts at a centralized server:

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip helper-address 172.16.2.11
```

Router or switch acting as a DHCP client itself (common on a branch router's WAN-facing interface talking to an ISP, or a switch leasing its management address):

```
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address dhcp
```

Router configured to use DNS, including a locally defined hostname as an alternative to a DNS lookup — useful in a lab where you don't want to stand up a real DNS server just to test `ping hostname`:

```
R1(config)# ip domain lookup
R1(config)# ip domain name example.com
R1(config)# ip name-server 8.8.8.8 8.8.4.4
!
! Or, skip DNS entirely and define the hostname locally
R1(config)# ip host hostB 172.16.2.101
```
