# Part 9: Security Services — Configs

## Securing Passwords

Turning on the old-style password encryption so clear-text passwords in the config aren't sitting out in plain view:

```
Switch3# configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Switch3(config)# service password-encryption
Switch3(config)# ^Z
Switch3# show running-config | section line con 0
line con 0
 password 7 070C285F4D06
 login
```

Setting an enable secret, which stores an MD5 hash instead of the actual password:

```
Switch3(config)# enable secret fred
Switch3(config)# ^Z
Switch3# show running-config | include enable secret
enable secret 5 $1$ZGMA$e8cmvkz4UjiJhVp7.maLE1
```

Using the newer, stronger algorithm types instead of the default MD5:

```
R1(config)# enable algorithm-type scrypt secret mypass1
R1# show running-config | include enable
enable secret 9 $9$II/EeKiRK9luxE$fwYuOESHoiil6AWv2wSywkLJ/KNeGj8uK/24B0TVD6
```

Configuring a local username with a hashed secret instead of the older clear-text password option:

```
R1(config)# username test05 algorithm-type scrypt secret cisco
```

## Port Security

Four interfaces on the same switch, each showing a different variation of port security:

```
SW1# show running-config
(Lines omitted for brevity)

interface FastEthernet0/1
 switchport mode access
 switchport port-security
 switchport port-security mac-address 0200.1111.1111
!
interface FastEthernet0/2
 switchport mode access
 switchport port-security
 switchport port-security mac-address sticky
!
interface FastEthernet0/3
 switchport mode access
 switchport port-security
!
interface FastEthernet0/4
 switchport mode trunk
 switchport port-security
 switchport port-security maximum 8
```

Checking a port that's been disabled by a violation:

```
SW1# show port-security interface fastEthernet 0/1
Port Security              : Enabled
Port Status                : Secure-shutdown
Violation Mode              : Shutdown
Aging Time                  : 0 mins
Aging Type                  : Absolute
```

Recovering a port that port security put into err-disabled state:

```
SW1(config-if)# shutdown
SW1(config-if)# no shutdown
```

Letting the switch recover err-disabled ports on its own instead of doing it by hand every time:

```
SW1(config)# errdisable recovery cause psecure-violation
SW1(config)# errdisable recovery interval 30
```

## DHCP Snooping

Core configuration to enable DHCP Snooping on VLAN 11 and trust the port facing the real DHCP server:

```
ip dhcp snooping
ip dhcp snooping vlan 11
no ip dhcp snooping information option
!
interface GigabitEthernet1/0/2
 ip dhcp snooping trust
```

Checking the status and trusted ports:

```
SW2# show ip dhcp snooping
Switch DHCP snooping is enabled
DHCP snooping is configured on following VLANs:
11
DHCP snooping is operational on following VLANs:
11
Insertion of option 82 is disabled
DHCP snooping trust/rate is configured on the following Interfaces:

Interface                   Trusted     Allow option    Rate limit (pps)
-----------------------    -------     ------------    ----------------
GigabitEthernet1/0/2        yes         yes             unlimited
```

Rate-limiting DHCP messages per interface and enabling automatic recovery:

```
errdisable recovery cause dhcp-rate-limit
errdisable recovery interval 30
!
interface GigabitEthernet1/0/2
 ip dhcp snooping limit rate 10
!
interface GigabitEthernet1/0/3
 ip dhcp snooping limit rate 2
```

Viewing the binding table DHCP Snooping builds automatically:

```
SW2# show ip dhcp snooping binding
MacAddress          IpAddress       Lease(sec)  Type            VLAN  Interface
------------------  --------------  ----------  --------------  ----  --------------------------
02:00:11:11:11:11  172.16.2.101   86110       dhcp-snooping   11    GigabitEthernet1/0/3
02:00:22:22:22:22  172.16.2.102   86399       dhcp-snooping   11    GigabitEthernet1/0/4
Total number of bindings: 2
```

## Dynamic ARP Inspection

Core configuration to enable DAI on VLAN 11 and trust the same uplink port:

```
ip arp inspection vlan 11
!
interface GigabitEthernet1/0/2
 ip arp inspection trust
```

A complete configuration combining DHCP Snooping and DAI together, since DAI relies on the DHCP Snooping binding table by default:

```
ip arp inspection vlan 11
ip dhcp snooping
ip dhcp snooping vlan 11
no ip dhcp snooping information option
!
interface GigabitEthernet1/0/2
 ip dhcp snooping trust
 ip arp inspection trust
```

Checking DAI status and violation counts:

```
SW2# show ip arp inspection statistics
Vlan  Forwarded  Dropped  DHCP Drops  ACL Drops
----  ---------  -------  ----------  ---------
11    59         17       17          0
```

Rate-limiting ARP messages per interface, with the optional burst interval DAI supports:

```
errdisable recovery cause arp-inspection
errdisable recovery interval 30
!
interface GigabitEthernet1/0/2
 ip dhcp snooping limit rate 10
 ip arp inspection limit rate 8
!
interface GigabitEthernet1/0/3
 ip dhcp snooping limit rate 2
 ip arp inspection limit rate 8 burst interval 4
```

Turning on the optional deeper checks DAI can run against the Ethernet header itself:

```
SW2(config)# ip arp inspection validate src-mac
SW2# show ip arp inspection
Source Mac Validation      : Enabled
Destination Mac Validation : Disabled
IP Address Validation      : Disabled
```
