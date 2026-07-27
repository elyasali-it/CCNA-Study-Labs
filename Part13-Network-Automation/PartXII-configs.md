# Part XII: Network Automation - Configs and Examples

## Northbound API Output as a Python Dictionary

A `show` command on a switch gives me text meant for a human to read:

```
SW1# show interfaces gigabit 0/1 switchport
Name: Gi0/1
Switchport: Enabled
Administrative Mode: dynamic auto
Operational Mode: static access
Administrative Trunking Encapsulation: dot1q
Operational Trunking Encapsulation: native
Negotiation of Trunking: On
```

A controller's northbound API returns the same information as a data structure a program can use directly, with no text-parsing required:

```python
>>> interface1
{'trunk-config': 'dynamic auto', 'trunk-status': 'static access'}
```

## Jinja2 Template and YAML Variables (Ansible)

A branch router configuration, with the values unique to this specific device highlighted:

```
hostname BR1
!
interface GigabitEthernet0/0
 ip address 10.1.1.1 255.255.255.0
 ip ospf 1 area 11
!
interface GigabitEthernet0/1
!
interface GigabitEthernet0/1/0
 ip address 10.1.12.1 255.255.255.0
 ip ospf 1 area 11
!
router ospf 1
 router-id 1.1.1.1
```

The Jinja2 template Ansible would use to generate that same configuration for any router playing this role, with the unique values replaced by variables:

```
hostname {{hostname}}
!
interface GigabitEthernet0/0
 ip address {{address1}} {{mask1}}
 ip ospf {{OSPF_PID}} area {{area}}
!
interface GigabitEthernet0/1
!
interface GigabitEthernet0/1/0
 ip address {{address2}} {{mask2}}
 ip ospf {{OSPF_PID}} area {{area}}
!
router ospf {{OSPF_PID}}
 router-id {{RID}}
```

The YAML variables file that supplies BR1's specific values for that template:

```yaml
hostname: BR1
address1: 10.1.1.1
mask1: 255.255.255.0
address2: 10.1.12.1
mask2: 255.255.255.0
RID: 1.1.1.1
OSPF_PID: '1'
area: '11'
```

A short YAML playbook file, showing how readable the format is even without knowing Ansible in depth:

```yaml
---
# This comment line is a place to document this Playbook
- name: Get IOS Facts
  hosts: mylab
  vars:
    cli:
      host: "{{ ansible_host }}"
      username: "{{ username }}"
      password: "{{ password }}"
  tasks:
    - ios_facts:
        gather_subset: all
        provider: "{{ cli }}"
```

## REST API Response Formats (Same Data, Three Serialization Languages)

JSON output from a REST API call to a controller:

```json
"response": {
   "family": "Switches and Hubs",
   "type": "Cisco Catalyst 9000 UADP 8 Port Virtual Switch",
   "macAddress": "52:54:00:01:c2:c0",
   "softwareType": "IOS-XE",
   "softwareVersion": "17.9.20220318:182713",
   "serialNumber": "9SB9FYAFA20",
   "upTime": "30 days, 10:05:18.00",
   "series": "Cisco Catalyst 9000 Series Virtual Switches",
   "hostname": "sw1.ciscotest.com",
   "managementIpAddress": "10.10.20.175",
   "platformId": "C9KV-UADP-8P",
   "role": "CORE"
}
```

The same kind of data returned as XML instead of JSON:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<root>
  <response>
    <family>Switches and Hubs</family>
    <hostname>cat_9k_1</hostname>
    <interfaceCount>41</interfaceCount>
    <lineCardCount>2</lineCardCount>
    <macAddress>f8:7b:20:67:62:80</macAddress>
    <managementIpAddress>10.10.22.66</managementIpAddress>
    <role>ACCESS</role>
    <serialNumber>FCW2136L0AK</serialNumber>
    <series>Cisco Catalyst 9300 Series Switches</series>
    <softwareType>IOS-XE</softwareType>
    <softwareVersion>16.6.1</softwareVersion>
    <type>Cisco Catalyst 9300 Switch</type>
    <upTime>17 days, 22:51:04.26</upTime>
  </response>
</root>
```

## Simple JSON Examples

A JSON object with three key:value pairs:

```json
{
  "1stbest": "Messi",
  "2ndbest": "Ronaldo",
  "3rdbest": "Maradona"
}
```

A JSON snippet listing a router's interfaces, showing two devices each with an array of interface names as the value:

```json
{
  "R1": ["GigabitEthernet0/0", "GigabitEthernet0/1", "GigabitEthernet0/2/0"],
  "R2": ["GigabitEthernet1/0", "GigabitEthernet1/1", "GigabitEthernet0/3/0"]
}
```

A JSON object with two key:value pairs, where each value is itself another nested object:

```json
{
  "Wendells_favorites": {
    "player": "Pedri",
    "team": "Barcelona",
    "league": "La Liga"
  },
  "interface_config": {
    "ip_address": "10.1.1.1",
    "ip_mask": "255.255.255.0",
    "speed": 1000
  }
}
```

## Simple Python Variables

```python
'''
Sample program to multiply two numbers and display the result
'''
x = 3
y = -4
z = 1.247
heading = "The product is "
print(heading, x*y)
```
