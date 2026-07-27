# Part XI: Configs

## VRF Example — Overlapping Subnets from Two Cloud Customers

This is a router-on-a-stick (ROAS) setup where one router, R1, needs to support two different customers whose VMs happen to use the exact same private subnets on the same physical server. Without VRFs, the router wouldn't allow this, since it would see the second customer's subnet as a duplicate of the first.

Customer A uses 10.1.1.0/24 and 10.1.2.0/24 on VLANs 10 and 20. Customer B uses the same two subnets, 10.1.1.0/24 and 10.1.2.0/24, but on VLANs 30 and 40. Each VLAN gets its own subinterface on R1, and each subinterface gets placed into the correct customer's VRF so the router keeps two completely separate routing tables.

```
interface g0/0/0.10
 encapsulation dot1q 10
 ip address 10.1.1.1 255.255.255.0

interface g0/0/0.20
 encapsulation dot1q 20
 ip address 10.1.2.1 255.255.255.0

interface g0/0/0.30
 encapsulation dot1q 30
 ip address 10.1.1.1 255.255.255.0

interface g0/0/0.40
 encapsulation dot1q 40
 ip address 10.1.2.1 255.255.255.0
```

Without VRFs assigned to these interfaces, IOS won't bring up the second pair of interfaces (VLANs 30 and 40) because it sees them as duplicate subnets already in use by the first pair. Assigning each pair of interfaces to its own VRF gives the router a separate routing table per customer, so the same subnet can exist twice on the same router with no conflict. Each VRF also keeps its own routing protocol neighbor relationships, so Customer A's routes and Customer B's routes never mix.
