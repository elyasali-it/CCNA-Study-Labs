# Part X: IP Services

This part covers five chapters that don't fit neatly into one theme but all live under the "IP Services" umbrella of the CCNA blueprint: device management protocols, NAT, QoS, first hop redundancy protocols, and SNMP/FTP/TFTP.

## Device Management Protocols

### Syslog

I know how Cisco devices report on themselves. When something happens on a router or switch, IOS generates a log message and by default sends it straight to anyone connected on the console port. Anyone connected over Telnet or SSH doesn't automatically see these messages — they have to run `terminal monitor` first, and the device also needs `logging monitor` enabled globally.

I know how to keep a copy of those messages instead of just watching them scroll by. `logging buffered` stores messages in RAM so I can pull them up later with `show logging`. `logging host <address>` sends a copy to a syslog server using UDP, which is the option most production networks use so all their devices report to one place.

I know the eight severity levels, from 0 (emergency) to 7 (debug). Lower numbers mean more severe. I can set a level by keyword or by number on each logging destination (`logging console`, `logging monitor`, `logging buffered`, `logging trap`), and IOS sends messages at that level and every level more severe than it. So `logging console 4` sends warnings and anything worse to the console, but not the informational or debug-level noise.

I know that `debug` commands are their own thing. Debug output always comes in at severity 7, and it can hammer the CPU if I leave it running or if a lot of users are watching it, so I use `debug` carefully on live equipment and turn it off with `no debug` when I'm done.

### Network Time Protocol (NTP)

I know why timestamps matter across a network — if I'm troubleshooting an issue that touched three routers and their clocks aren't synced, the log messages from each device won't line up and I'll waste time chasing events that didn't actually happen in that order.

I know how to configure NTP. `ntp server <address>` puts a router in client/server mode: it syncs its own clock to that server, then can turn around and serve time to other devices. `ntp master <stratum>` makes a router an NTP server only, using its own internal clock as the source — no upstream server needed.

I know what stratum means. It's a number representing how many hops a device is from the original time source, and lower is better. A device with `ntp master` sets its own stratum. Every device downstream adds 1 to whatever stratum it learned from its server. Cisco treats stratum 16 as unusable — the device won't trust that time.

I know how to check the settings and verify sync. `clock timezone` and `clock summer-time` are configured before setting the time itself, because they change how the device interprets the time I'm about to enter. `show ntp status` tells me whether the local device is synchronized and what its reference clock is. `show ntp associations` shows every server the device is trying to sync with and whether that association succeeded.

### CDP and LLDP

I know how to discover what's connected to a router or switch without needing IP connectivity to that neighbor. Cisco Discovery Protocol (CDP) and Link Layer Discovery Protocol (LLDP) both run at Layer 2, so they work even before IPv4 or IPv6 is configured on the link.

I know what these protocols tell me about a neighbor: its hostname, its IP address, which of its interfaces connects to which of mine, what kind of device it is (router vs. switch), and its platform and software version. That's enough information to build or confirm a network diagram just from CLI output.

I know the difference between CDP and LLDP. CDP is Cisco-only. LLDP is a standard, so it works with non-Cisco gear too. LLDP separates transmitting and receiving as two independently configurable functions per interface; CDP doesn't split those apart. LLDP also only lists a neighbor's enabled capabilities, while CDP lists everything the neighbor supports whether it's turned on or not.

I know the commands: `show cdp neighbors` and `show lldp neighbors` give me a one-line summary per neighbor. Adding `detail` (or using `show cdp entry <name>` / `show lldp entry <name>` for one specific neighbor) gives me the full picture, including platform and IOS version. I can also disable either protocol globally with `no cdp run` / `no lldp run`, or per interface.

I know one practical use for this beyond documentation: IP phones use CDP or LLDP-MED to learn which VLAN is the voice VLAN from the switch, so I don't have to hardcode that on the phone itself.

## Network Address Translation (NAT)

I know what problem NAT solves. Every device on the internet needs a unique public address, but the internet doesn't have enough IPv4 addresses for that to work at scale. NAT lets a whole private network share a small number of public addresses, which is one of the main reasons IPv4 lasted as long as it did.

I know the vocabulary Cisco uses. Inside local is the private address a host actually has configured on it. Inside global is the public address that same host appears to have once its traffic crosses the NAT router. NAT keeps a translation table mapping one to the other.

I know the three variations and when each one gets used:

**Static NAT** maps one private address to one public address permanently, configured with `ip nat inside source static <local> <global>`. It doesn't save any public addresses since it's still one-to-one, but because the mapping already exists, connections from the internet can reach that inside host — which is exactly why I'd use static NAT for a server.

**Dynamic NAT** also maps one-to-one, but it builds the mapping automatically the first time a matching inside host sends a packet, pulling the next available address from a configured pool. It still doesn't save addresses, and because the entry only gets created when an inside host initiates traffic, connections from the internet can't reach an inside host that hasn't talked to them first — that's a built-in security benefit.

**PAT (Port Address Translation)**, also called NAT overload, is what almost every real network actually runs. It maps many inside hosts to one — or a few — public addresses by also translating the port number, not just the IP address. Since the port field has 16 bits, PAT can support over 65,000 concurrent connections through a single public IP. I configure it the same way as dynamic NAT but add the `overload` keyword to the end of the `ip nat inside source list` command.

I know the basic configuration checklist, and it's the same shape for all three types: mark the inside interface with `ip nat inside`, mark the outside interface with `ip nat outside`, and then define what gets translated. For static, that's the static mapping command. For dynamic and PAT, that's an ACL identifying which inside addresses are eligible, plus either a pool (`ip nat pool`) or, for the single-address version of PAT, just the outside interface itself.

I know how to verify and troubleshoot it. `show ip nat translations` lists the active mappings. `show ip nat statistics` shows hits, misses, and how full a dynamic pool is — a rising misses counter on the pool means I've run out of addresses to hand out. The most common mistakes I check for: inside and outside marked backward on the interfaces, the static command listing local and global addresses in the wrong order, an ACL that matches the wrong (post-translation) address, and forgetting the `overload` keyword when I actually wanted PAT instead of dynamic NAT.

## Quality of Service (QoS)

I know what QoS actually manages: bandwidth (link capacity), delay (one-way or round-trip), jitter (the variation in delay between consecutive packets), and loss (the percentage of packets that never arrive). A networking device's QoS tools decide who gets a bigger share of these when there isn't enough to go around.

I know that different traffic types need different things. Interactive voice needs low delay, low jitter, and low loss, but not much bandwidth per call — Cisco's guideline is 150ms one-way delay, 30ms jitter, and under 1% loss. Video needs similar delay and jitter numbers but a lot more bandwidth. Regular data traffic (web browsing, file transfers) tolerates delay and jitter fine, but a burst of it can crowd out voice and video if nothing manages that.

I know the four main tools and how they fit together:

**Classification and marking** happens first. Classification is comparing a packet's header fields against some criteria, exactly like an ACL does. Marking changes a field in the header — most commonly IP DSCP (6 bits, 64 possible values, works end-to-end since it's in the IP header) or Ethernet CoS (3 bits, only exists on 802.1Q trunk links). The point of marking early is that every device downstream can then apply QoS with a simple, cheap comparison instead of redoing complex matching at every hop.

**Queuing** decides which packet in an output queue gets sent next when the interface is busy. Round-robin scheduling cycles through queues in order, and weighted round robin (like CBWFQ) lets me give one queue a bigger share of bandwidth than another. Low Latency Queuing (LLQ) adds a true priority queue on top of that — LLQ always services that queue first, which is what voice and video actually need since they can't tolerate waiting behind a full round-robin rotation. To keep a priority queue from starving everything else, I cap how much traffic it's allowed to carry.

**Shaping and policing** both watch the bit rate of traffic and compare it to a configured rate. Policing discards packets that exceed the rate (or re-marks them). Shaping queues and delays packets that exceed the rate instead of dropping them, smoothing the traffic out over time. I'd use policing at a point where I want to enforce a hard limit — like an ISP capping a customer's incoming rate to their contracted CIR — and shaping where I control both ends of the link and want to avoid triggering the other side's policer.

**Congestion avoidance** works on TCP traffic specifically. TCP grows its sending window until something tells it to slow down. Rather than let a queue fill up completely and hit tail drop — which can cause a lot of TCP connections to slow down all at once — congestion avoidance tools discard a small percentage of packets early, as the queue depth grows past a minimum threshold, so TCP backs off gradually instead of all together.

I know these tools sit in a pipeline on the router: classify, then queue, then (optionally) shape, with congestion avoidance managing what happens as those queues fill.

## First Hop Redundancy Protocols (FHRP)

I know the problem FHRPs solve. A host only has one default gateway setting. If that router goes down and there's no FHRP, every host pointed at it loses the ability to send anything off its own subnet, even if a second router on the same LAN is healthy and could have done the job.

I know the shared model behind every FHRP, regardless of which one I'm using: multiple routers on the same subnet cooperate and present themselves to hosts as a single default router. They share a virtual IP address (VIP) that hosts use as their gateway setting, and that VIP maps to a virtual MAC address. Hosts never have to change anything — their gateway setting and ARP table both stay pointed at the virtual address the whole time, even during a failover.

I know the three protocols and how they compare:

**HSRP** (Hot Standby Router Protocol) is Cisco-proprietary and works active/standby: one router handles all the traffic while the other sits idle, ready to take over. The active router owns the virtual IP and virtual MAC. HSRP priority decides which router becomes active — highest priority wins — and I can enable preemption so that when a higher-priority router comes back online after an outage, it takes the active role back instead of leaving the current active router in charge. HSRP also supports interface or object tracking, which lowers a router's priority automatically if something it depends on (like a WAN link) goes down, so a healthier router can preempt it.

**VRRP** is the IETF's standardized version, functionally very close to HSRP, but it uses the terms master and backup instead of active and standby, and its default settings (preemption on by default, for one) differ slightly from HSRP's.

**GLBP** is Cisco-proprietary like HSRP but adds real load balancing. Instead of one active router handling everything, GLBP elects one router as the active virtual gateway (AVG), which answers ARP requests but alternates which virtual MAC address it hands out to different hosts. Every router in the group is an active virtual forwarder (AVF) for its own virtual MAC, so traffic actually splits across all the routers instead of all flowing through just one.

I know how to spot each one's default multicast address and other identifying details if I'm looking at packet captures or trying to tell them apart: HSRPv2 uses 224.0.0.102, VRRP uses 224.0.0.18, and GLBP also uses 224.0.0.102 but with a different message format.

## SNMP, FTP, and TFTP

### SNMP

I know the basic model: an SNMP manager (running on a Network Management Station, or NMS) polls SNMP agents running on each managed device. Every agent maintains a Management Information Base (MIB) — a structured database of variables describing that device's configuration, status, and counters.

I know the two main things a manager can do. An SNMP Get (or GetNext, or GetBulk for retrieving several variables efficiently in one request) reads a variable's current value. An SNMP Set writes a new value to a variable, which is how an NMS can reconfigure a device remotely through SNMP rather than the CLI.

I know that devices can also talk first. A Trap or Inform message lets an agent notify the NMS the moment something happens, rather than waiting for the next poll. Traps are fire-and-forget over UDP, so there's no guarantee one arrives. Informs add an acknowledgment from the NMS, so the agent knows to resend if it doesn't hear back — more reliable, at the cost of a little more overhead.

I know how SNMP versions differ on security. SNMPv1 and SNMPv2c both authenticate with a community string sent in clear text — read-only community for Gets, read-write community for Gets and Sets. SNMPv3 replaces that with real usernames, hashed passwords, and optional encryption, which is the version I'd want for anything beyond a lab.

### FTP and TFTP

I know both protocols move files using a client/server model, but they solve different problems. FTP is a full-featured file transfer protocol — navigate directories, list files, create or remove directories, get and put files — running over TCP for reliable, ordered delivery. It uses a separate control connection (commands) and data connection (the actual file), and can run in active mode (server initiates the data connection back to the client — often blocked by firewalls) or passive mode (client initiates both connections, which works through firewalls much more reliably).

I know TFTP is deliberately stripped down: no authentication, no directory browsing, just Get and Put, running over UDP on port 69 with a checksum to confirm each transfer completed cleanly. That simplicity is the point — it's a lightweight tool for quick, temporary file moves in a controlled environment, most commonly for pushing a new IOS image onto a router or switch.

I know the practical use case tying this to router/switch management: the IOS itself is just one file sitting in the device's flash file system. To upgrade, I copy a new IOS file into flash from a TFTP or FTP server with the `copy` EXEC command, and IOS walks me through confirming the server address, the source filename, and whether there's enough free space before it actually transfers.

I know how to check what's in flash. `show flash:` lists every file with its size, in file-number order. `dir` shows the contents of whatever directory I'm currently in, and `cd`/`pwd` let me navigate the file system the same way I would on a regular OS.

I know how to confirm an IOS file hasn't been tampered with. Cisco publishes an MD5 and SHA512 hash for every IOS image it releases. Running `verify /md5` or `verify /sha512` against the local file recalculates that hash on the router, and I compare it to the published value — if they match, the file is intact.

## Troubleshooting Scenario

A user calls saying they can reach internal resources but nothing on the internet. I check their default gateway and it's still answering pings, so the FHRP virtual IP is up. I check NAT next with `show ip nat translations` and see no entries at all for their address, so I look at `show ip nat statistics` and the misses counter on the dynamic pool is climbing — the pool ran dry.

While I'm digging into that, I notice the syslog server hasn't logged anything from this router in over an hour, even though I know changes were made. I check `show ntp status` and the router shows unsynchronized. Its clock has drifted enough that timestamps on any log messages it did send wouldn't line up with the rest of the network anyway, which would have made this harder to diagnose if I'd needed to correlate events across devices.

I free up the NAT pool and get NTP re-pointed at a working server. Traffic to the internet starts flowing again, and once the clock is back in sync, the log messages start landing on the syslog server with timestamps I can actually trust for the next issue that comes up.
