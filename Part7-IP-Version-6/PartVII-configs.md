# Part VII Configs: IP Version 6

These are the real device configs I pulled out of Part VII. I keep them in their own file so I can grab a command fast without digging through the explanation around it.

## Enabling IPv6 routing (do this first, every time)

IPv4 forwarding is on by default. IPv6 forwarding is not, so this is the one command I'll forget if I don't make it a habit:

```
ipv6 unicast-routing
```

## Static addressing, full 128-bit address

```
interface GigabitEthernet0/0/0
 ipv6 address 2001:DB8:1111:1::1/64
!
interface GigabitEthernet0/0/1
 ipv6 address 2001:db8:1111:12::1/64
```

IOS accepts either uppercase or lowercase hex on input and normalizes to uppercase in the abbreviated form when you look at the running config afterward.

## Static addressing with EUI-64 (letting the router build the interface ID)

```
ipv6 unicast-routing
!
interface GigabitEthernet0/0/0
 mac-address 0200.1111.1111
 ipv6 address 2001:DB8:1111:1::/64 eui-64
!
interface GigabitEthernet0/0/1
 ipv6 address 2001:DB8:1111:12::/64 eui-64
```

Note the prefix only, no host bits, when using `eui-64`. Listing a full address with the `eui-64` keyword still works, IOS will just convert it to the matching prefix before it goes into the running config.

## Dynamic address configuration on router interfaces

```
! Learn address via DHCPv6
interface GigabitEthernet0/0/0
 ipv6 address dhcp
!
! Learn address via SLAAC
interface GigabitEthernet0/0/1
 ipv6 address autoconfig
```

## ipv6 enable (link-local only, no GUA/ULA)

Useful on WAN links where routing only needs a link-local next hop and never needs a routable subnet:

```
interface GigabitEthernet0/0/2
 ipv6 enable
```

## Anycast address

```
interface GigabitEthernet0/0/0
 ipv6 address 2001:1:1:1::1/64
 ipv6 address 2001:1:1:2::99/128 anycast
```

The same 2001:1:1:2::99/128 gets configured identically on every router that should answer for that anycast service; the routing protocol advertises it like any other host route, and the network delivers each packet to whichever instance is closest.

## DHCPv6 relay agent

```
interface GigabitEthernet0/0/0
 ipv6 dhcp relay destination 2001:DB8:1111:2::8
```

Configured on the router interface facing the client subnet, pointing at the remote DHCPv6 server's address on the far side.

## Static network route, outgoing interface only

Only works reliably on serial/point-to-point interfaces, not Ethernet:

```
ipv6 route 2001:DB8:1111:2::/64 S0/0/1
```

## Static network route, next-hop global unicast address

```
ipv6 route 2001:db8:1111:3::/64 2001:db8:1111:13::3
```

IOS does an iterative lookup to figure out the real outgoing interface from this next-hop address, so this style works on Ethernet as well as serial.

## Static network route, next-hop GUA plus outgoing interface (fully specified)

```
ipv6 route 2001:DB8:1111:3::/64 GigabitEthernet0/0/2 2001:DB8:1111:13::3
```

Skips the iterative lookup since the outgoing interface is already given directly.

## Static network route, next-hop link-local address

Requires the outgoing interface every time, an LLA alone will be rejected:

```
ipv6 route 2001:db8:1111:3::/64 GigabitEthernet0/0/2 fe80::ff:fe01:3
```

## Static default route

```
ipv6 route ::/0 GigabitEthernet0/0/1 fe80::ff:fe02:1
```

## Static host route (/128)

```
ipv6 route 2001:db8:1111:d::4/128 GigabitEthernet0/0/1 2001:db8:1111:12::2
```

## Floating static route (backup over cellular, active only if the primary route disappears)

Default AD for a static route is 1, which normally beats a routing protocol. Raise it above the primary route's AD (OSPF is 110) so it only kicks in on failure:

```
ipv6 route 2001:db8:1111:7::/64 cellular0/1/0 130
```

## Quick reference: default IPv6 administrative distances

| Route source | AD |
|---|---|
| Connected | 0 |
| Static | 1 |
| NDP default route | 2 |
| eBGP | 20 |
| EIGRP (internal) | 90 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| EIGRP (external) | 120 |
| iBGP | 170 |
| Unusable | 255 |

## Verification commands I reach for most

```
show ipv6 interface brief
show ipv6 interface GigabitEthernet0/0/0
show ipv6 route
show ipv6 route static
show ipv6 route connected
show ipv6 route local
show ipv6 route <address>
show ipv6 neighbors
show ipv6 routers
ping ipv6 <address>
traceroute ipv6 <address>
```
