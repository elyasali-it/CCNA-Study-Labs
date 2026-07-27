# Part X: IP Services — Configs

## Syslog

Basic syslog setup, sending severity level 4 (warnings) and worse to both the console/monitor sessions and a syslog server, plus buffering level 4 and worse locally:

```
logging console 7
logging monitor debug
logging buffered 4
logging host 172.16.3.9
logging trap warning
```

Confirming what's configured and what's been logged:

```
R1# show logging
```

Disabling timestamps and switching to sequence numbers instead, useful when I want to see the order messages happened in rather than exact time:

```
R1(config)# no service timestamps
R1(config)# service sequence-numbers
```

## NTP

Setting the time zone and daylight savings before setting the clock itself:

```
R1(config)# clock timezone EST -5
R1(config)# clock summer-time EDT recurring
R1# clock set 12:32:00 19 January 2023
```

Basic client/server chain — R1 syncs to R2, R2 syncs to R3, R3 is the master:

```
! Configuration on R1:
ntp server 172.16.2.2

! Configuration on R2:
ntp server 172.16.3.3

! Configuration on R3:
ntp master 2
```

Verifying:

```
R1# show ntp status
R1# show ntp associations
```

## CDP and LLDP

Enabling LLDP globally, then disabling it in both directions on one interface and one direction on another:

```
lldp run
!
interface gigabitEthernet1/0/17
 no lldp transmit
 no lldp receive
!
interface gigabitEthernet1/0/18
 no lldp receive
```

Verifying:

```
SW2# show cdp neighbors detail
SW2# show lldp neighbors
SW2# show lldp entry R1
```

## NAT

### Static NAT

Two hosts get permanent one-to-one public mappings:

```
interface GigabitEthernet0/0/0
 ip address 10.1.1.3 255.255.255.0
 ip nat inside
!
interface GigabitEthernet0/0/1
 ip address 200.1.1.253 255.255.255.0
 ip nat outside
!
ip nat inside source static 10.1.1.2 200.1.1.2
ip nat inside source static 10.1.1.1 200.1.1.1
```

Verifying:

```
NAT# show ip nat translations
NAT# show ip nat statistics
```

### Dynamic NAT

Same two inside hosts, but now pulling from a pool instead of a fixed mapping:

```
interface GigabitEthernet0/0/0
 ip address 10.1.1.3 255.255.255.0
 ip nat inside
!
interface GigabitEthernet0/0/1
 ip address 200.1.1.253 255.255.255.0
 ip nat outside
!
ip nat pool fred 200.1.1.1 200.1.1.2 netmask 255.255.255.252
ip nat inside source list 1 pool fred
!
access-list 1 permit 10.1.1.2
access-list 1 permit 10.1.1.1
```

### PAT (NAT Overload)

Using the outside interface's own address as the only public address needed:

```
interface GigabitEthernet0/0/0
 ip address 10.1.1.3 255.255.255.0
 ip nat inside
!
interface GigabitEthernet0/0/1
 ip address 200.1.1.249 255.255.255.252
 ip nat outside
!
ip nat inside source list 1 interface GigabitEthernet0/0/1 overload
!
access-list 1 permit 10.1.1.2
access-list 1 permit 10.1.1.1
```

Clearing the dynamic NAT table when I need entries rebuilt from scratch:

```
NAT# clear ip nat translation *
```

## FTP and TFTP

Copying an IOS image from a TFTP server into flash, walking through the interactive prompts:

```
R2# copy tftp: flash:
Address or name of remote host []? 2.2.2.1
Source filename []? c1100-universalk9.17.06.03a.SPA.bin
Destination filename [c1100-universalk9.17.06.03a.SPA.bin]?
```

Same idea using FTP, with the username and password embedded in the URI:

```
R1# copy ftp://wendell:odom@192.168.1.170/c1100-universalk9.17.06.03a.SPA.bin flash:
```

Configuring FTP credentials on the router once, so I don't have to put them in the URI every time:

```
ip ftp username wendell
ip ftp password odom
```

Checking flash contents and confirming the file transferred intact:

```
R2# show flash:
R2# dir
R2# pwd
R2# verify /sha512 flash0:c1100-universalk9.17.06.03a.SPA.bin
```
