# Part VI Configs: OSPF

Real configuration snippets from the OSPF chapters, pulled out here since there's enough of them to be worth their own reference instead of buried in prose.

## Traditional configuration with the network command

Single broad network command sweeping in every interface on the router:

```
router ospf 1
 network 10.0.0.0 0.255.255.255 area 0
```

More granular, one network command per interface with an exact-match wildcard:

```
router ospf 1
 network 10.1.13.3 0.0.0.0 area 0
 network 10.1.23.3 0.0.0.0 area 0
```

Wildcard mask logic — 0 means match exactly, 255 means ignore that octet:

```
network 10.1.0.0 0.0.255.255 area 0      ! matches addresses beginning with 10.1
network 10.0.0.0 0.255.255.255 area 0    ! matches addresses beginning with 10
network 0.0.0.0 255.255.255.255 area 0   ! matches all addresses
network 10.1.13.0 0.0.0.255 area 0       ! matches addresses beginning with 10.1.13
network 10.1.13.1 0.0.0.0 area 0         ! matches exactly 10.1.13.1
```

## Interface-style configuration (newer method, no wildcard math)

```
router ospf 1
!
interface GigabitEthernet0/0/0
 ip ospf 1 area 0
!
interface GigabitEthernet0/0/1
 ip ospf 1 area 0
```

Migrating an existing network-command config over to interface style:

```
router ospf 1
 no network 10.0.0.0 0.255.255.255 area 0
!
interface GigabitEthernet0/0/0
 ip ospf 1 area 0
interface GigabitEthernet0/0/1
 ip ospf 1 area 0
```

## Setting the router ID manually

Directly, so it doesn't drift after a reload:

```
router ospf 1
 router-id 1.1.1.1
 network 10.1.0.0 0.0.255.255 area 0
```

Or via a loopback, which stays up/up regardless of physical interface state:

```
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
```

To force a router to pick up a newly configured RID without a reload:

```
clear ip ospf process
```

## Multiarea placement

```
router ospf 1
 network 10.1.1.1 0.0.0.0 area 0
 network 10.1.12.1 0.0.0.0 area 23
 network 10.1.13.1 0.0.0.0 area 23
 network 10.1.14.1 0.0.0.0 area 4
```

## Network type

Forcing point-to-point on what's physically an Ethernet WAN link:

```
interface GigabitEthernet0/0/1
 ip ospf network point-to-point
```

Reverting to the Ethernet default explicitly:

```
interface GigabitEthernet0/0/0
 ip ospf network broadcast
```

## Influencing DR/BDR elections

```
interface GigabitEthernet0/0
 ip ospf priority 99
```

Priority of 0 removes a router from DR/BDR eligibility entirely — used deliberately, or accidentally on every router on a segment, which is a known failure mode (nobody becomes DR, neighbors stall at 2WAY/DROTHER):

```
interface GigabitEthernet0/0/3
 ip ospf priority 0
```

## Passive interfaces

Make one interface passive directly:

```
router ospf 1
 passive-interface GigabitEthernet0/0/0
```

Flip the default to passive, then explicitly un-passive the interfaces that face other routers — cleaner on a router with a lot of LAN interfaces:

```
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/0/1
 no passive-interface GigabitEthernet0/0/2
 no passive-interface GigabitEthernet0/0/3
```

## Default route origination

On the internet-edge router, static default plus OSPF advertisement of it:

```
ip route 0.0.0.0 0.0.0.0 192.0.2.1
!
router ospf 1
 network 10.0.0.0 0.255.255.255 area 0
 router-id 1.1.1.1
 default-information originate
```

`always` keeps advertising the default even without a local default route currently present:

```
default-information originate always
```

## Cost

Setting cost directly on an interface:

```
interface GigabitEthernet0/0/1
 ip ospf cost 5
```

Raising the reference bandwidth so faster links stop tying at cost 1 — needs to be the same value on every router in the domain:

```
router ospf 1
 auto-cost reference-bandwidth 100000
```

(100000 = 100,000 Mbps = 100 Gbps as the new reference point, meaning a 10-Gig link calculates to a cost of 10, a 1-Gig link to 100, instead of everything above 100 Mbps tying at 1.)

## Hello and Dead intervals

Setting both explicitly per interface — has to match the neighbor on the other end of each link:

```
interface GigabitEthernet0/0/1
 ip ospf hello-interval 5
!
interface GigabitEthernet0/0/2
 ip ospf dead-interval 20
!
interface GigabitEthernet0/0/3
 ip ospf hello-interval 5
 ip ospf dead-interval 15
```

## Shutting OSPF down without removing configuration

Whole process, on the router:

```
router ospf 1
 shutdown
```

Bring it back:

```
router ospf 1
 no shutdown
```

Single interface only, leaving the rest of the OSPF process running:

```
interface GigabitEthernet0/0/0
 ip ospf shutdown
```

Bring that interface back:

```
interface GigabitEthernet0/0/0
 no ip ospf shutdown
```

## Verification commands worth having memorized

```
show ip ospf neighbor
show ip ospf neighbor detail
show ip ospf interface brief
show ip ospf interface [type number]
show ip ospf database
show ip ospf
show ip protocols
show ip route ospf
show ip route [address]
```
