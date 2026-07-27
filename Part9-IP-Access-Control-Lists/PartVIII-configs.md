# Part VIII: Configs

## Standard Numbered ACL

Basic standard ACL matching a host, a range, and a broader subnet, then applied inbound on an interface:

```
R2# configure terminal
R2(config)# access-list 1 permit 10.1.1.1
R2(config)# access-list 1 deny 10.1.1.0 0.0.0.255
R2(config)# access-list 1 permit 10.0.0.0 0.255.255.255
R2(config)# interface gigabitethernet 0/0/1
R2(config-if)# ip access-group 1 in
```

Verifying it:

```
R2# show ip access-lists
Standard IP access list 1
    10 permit 10.1.1.1 (107 matches)
    20 deny 10.1.1.0, wildcard bits 0.0.0.255 (4 matches)
    30 permit 10.0.0.0, wildcard bits 0.255.255.255 (10 matches)
```

## Standard ACL with Remarks and Logging

Using a remark for documentation and adding the log keyword so IOS generates a message whenever the line matches:

```
access-list 2 remark This ACL permits server S1 traffic to host A's subnet
access-list 2 permit 10.2.2.1 log
```

Sample log message from that line matching a packet:

```
R1# show running-config
access-list 2 remark This ACL permits server S1 traffic to host A's subnet
access-list 2 permit 10.2.2.1 log
!
interface G0/0/0
 ip access-group 2 out

R1#
Feb 4 18:30:24.082: %SEC-6-IPACCESSLOGNP: list 2 permitted 0 10.2.2.1 -> 10.1.1.1, 1 packet
```

## Named Standard ACL

Creating a named ACL, adding permit/deny lines in ACL configuration mode, and applying it outbound:

```
R2# configure terminal
R2(config)# ip access-list standard Hannah
R2(config-std-nacl)# remark A sample ACL, originally five lines
R2(config-std-nacl)# permit 10.1.1.2
R2(config-std-nacl)# deny 10.1.1.1
R2(config-std-nacl)# deny 10.1.3.0 0.0.0.255
R2(config-std-nacl)# deny 10.1.2.0 0.0.0.255
R2(config-std-nacl)# permit any
R2(config-std-nacl)# interface GigabitEthernet0/0/1
R2(config-if)# ip access-group Hannah out
```

Deleting one line by sequence number, and deleting one line by retyping the exact command with no:

```
R2# configure terminal
R2(config)# ip access-list extended Hannah
R2(config-std-nacl)# no deny 10.1.2.0 0.0.0.255
R2(config-std-nacl)# no 20
```

Inserting a new line at a specific sequence number so it lands between two existing lines:

```
R2# configure terminal
R2(config)# ip access-list extended Hannah
R2(config-std-nacl)# 40 deny 10.1.2.0 0.0.0.255
R2(config-std-nacl)# 20 deny 10.1.1.1
```

## Extended Named ACL for Web Traffic

Matching HTTP and HTTPS to two different destinations, applied outbound on the WAN interface:

```
R1# configure terminal
R1(config)# ip access-list extended branch_WAN
R1(config-ext-nacl)# remark Example ACL to match HTTP/S
R1(config-ext-nacl)# permit tcp 10.1.4.0 0.0.1.255 10.2.16.0 0.0.3.255 eq 80
R1(config-ext-nacl)# permit tcp 10.1.4.0 0.0.1.255 10.2.16.0 0.0.3.255 eq 443
R1(config-ext-nacl)# permit tcp 10.1.4.0 0.0.1.255 host 10.2.32.1 eq 80
R1(config-ext-nacl)# permit tcp 10.1.4.0 0.0.1.255 host 10.2.32.1 eq 443
R1(config-ext-nacl)# interface gigabitethernet0/0/1
R1(config-if)# ip access-group branch_WAN out
```

Verifying it, showing IOS converting port 80 to the www keyword automatically:

```
R1# show access-list
Extended IP access list branch_WAN
    10 permit tcp 10.1.4.0 0.0.1.255 10.2.16.0 0.0.3.255 eq www (18 matches)
    20 permit tcp 10.1.4.0 0.0.1.255 10.2.16.0 0.0.3.255 eq 443 (416 matches)
    30 permit tcp 10.1.4.0 0.0.1.255 host 10.2.32.1 eq www
    40 permit tcp 10.1.4.0 0.0.1.255 host 10.2.32.1 eq 443
```

## Extended ACL Matching HTTP/3 (QUIC over UDP)

Adding a UDP line so an ACL built for classic HTTP/HTTPS also matches HTTP/3 traffic:

```
R2# show running-config | section access-list
ip access-list extended DC_WAN
 10 permit tcp 10.2.16.0 0.0.3.255 eq www 10.1.4.0 0.0.1.255
 20 permit tcp 10.2.16.0 0.0.3.255 eq 443 10.1.4.0 0.0.1.255
 25 permit udp 10.2.16.0 0.0.3.255 eq 443 10.1.4.0 0.0.1.255
 30 permit tcp host 10.2.32.1 eq www 10.1.4.0 0.0.1.255
 40 permit tcp host 10.2.32.1 eq 443 10.1.4.0 0.0.1.255
 45 permit udp host 10.2.32.1 eq 443 10.1.4.0 0.0.1.255
```

## Permitting DNS Traffic

One-stage approach, permitting all DNS regardless of source or destination:

```
R1# show running-config | section access-list
! ACEs below are part of extended ACL Branch_Common
50 permit udp any any eq domain
60 permit tcp any any eq domain
```

Two-stage approach, permitting DNS only to the known DNS servers, then denying all other DNS traffic:

```
R1# show running-config | section access-list
! Except from extended ACL Branch_Common, replacing previous example's ACEs
110 permit udp any host 10.2.32.1 eq domain
120 permit udp any host 10.2.32.2 eq domain
130 permit tcp any host 10.2.32.1 eq domain
140 permit tcp any host 10.2.32.2 eq domain
! The next lines mimic example 8-1's ACEs, denying DNS (UDP and TCP)
150 deny udp any any eq domain
160 deny tcp any any eq domain
```

## Permitting ICMP

One-stage approach, permitting all ICMP:

```
210 permit icmp any any
```

Two-stage approach, permitting ICMP only within the private network, denying everything else:

```
220 permit icmp 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255
230 deny icmp any any
```

Narrower approach, permitting only Echo and Echo Reply within the private network:

```
ip access-list extended icmp_Echo_network_10
250 permit icmp 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255 echo
260 permit icmp 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255 echo-reply
270 deny icmp any any
```

## Permitting OSPF

One-stage approach, permitting all OSPF:

```
310 permit ospf any any
```

Two-stage approach, permitting OSPF only from a known neighbor, denying everything else:

```
320 permit ospf host 10.1.12.2 any
330 deny ospf any any
```

## Permitting DHCP Traffic to a DHCP Server

One-stage approach:

```
240 permit udp any any eq bootps
```

Two-stage approach, permitting to the known server then denying all other DHCP messages:

```
250 permit udp any host 10.2.16.1 eq bootps
260 deny udp any any eq bootps
```

## DHCP with the ip helper-address Command

On the router doing the helper function, the inbound ACL has to match the pre-helper addresses, since the ACL sees the packet before the helper rewrites source and destination:

```
R1# show running-config
interface GigabitEthernet0/0/0
 ip address 10.1.4.254 255.255.254.0
 ip helper-address 10.2.16.1
 ip access-group R1_Common in
!
! ACL excerpt: permit packets with unusual source/destination addresses to DHCP server
250 permit udp host 0.0.0.0 host 255.255.255.255 eq bootps
260 deny udp any any eq bootps
```

## Permitting SSH and Telnet, Checking Both Directions

Matching both source and destination port with range, since return traffic uses the well-known port as the source port instead of the destination port:

```
410 permit tcp any any range 22 telnet
420 permit tcp any range 22 telnet any
```

Two-stage version, restricting to the private network on both sides:

```
450 permit tcp 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255 range 22 telnet
460 permit tcp 10.0.0.0 0.255.255.255 range 22 telnet 10.0.0.0 0.255.255.255
470 deny tcp any any range 22 telnet
480 deny tcp any range 22 telnet any
```

## vty ACL for Router Access

Restricting inbound SSH/Telnet to the router itself, limited to a known management subnet:

```
line vty 0 15
 transport input all
 access-class IT_only in
!
ip access-list standard IT_only
 remark matches packets sourced from subnet 10.1.1.0/24 only; implied deny any
 10 permit 10.1.1.0 0.0.0.255
```

Outbound vty ACL, filtering based on the destination the router itself is connecting out to:

```
! Configuration excerpt first
line vty 0 15
 transport input all
 access-class R2_WAN out
!
ip access-list standard R2_WAN
 10 permit host 10.1.12.1
```

## Resequencing ACL Numbers

Renumbering an existing ACL's sequence numbers, starting at 100 and incrementing by 20:

```
R1# configure terminal
R1(config)# ip access-list resequence acl_01 100 20
R1(config)# do show access-list acl_01
Extended IP access list acl_01
    100 permit ip 10.1.4.0 0.0.1.255 any
    120 permit ip 10.2.4.0 0.0.1.255 any
    140 permit ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255
```

## Two ACLs on One Interface, One Direction (IOS XE Common ACL)

Applying a common ACL alongside a regular ACL in the same direction, IOS XE only:

```
R1# configure terminal
R1(config-if)# interface gigabitEthernet 0/0/1
R1(config-if)# ip access-group common common_all unique_01 out
R1(config-if)# do show ip interface g0/0/1
GigabitEthernet0/0/1 is up, line protocol is up
  Internet address is 10.1.12.1/24
  Outgoing Common access list is common_all
  Outgoing access list is unique_01
  Inbound Common access list is not set
  Inbound access list is not set
```
