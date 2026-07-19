# Part V — IPv4 Routing

Parts I through IV got me through Ethernet switching, VLANs, and subnetting math. Part V is where the book finally hands packets to a router and says "your problem now." Chapters 16 through 20 walk from installing a router all the way to troubleshooting a broken route with nothing but ping, traceroute, and a Telnet session. I went into this Part expecting more of the same memorize-and-move-on pace as the earlier ones, and instead it's the first stretch of the book that actually felt like the job I already do every day, just with better vocabulary attached to it.

I'm covering Chapters 16–20 here: Operating Cisco Routers, Configuring IPv4 Addresses and Static Routes, IP Routing in the LAN, IP Addressing on Hosts, and Troubleshooting IPv4 Routing. Device configs that are worth keeping on their own — the ROAS subinterfaces, the SVIs, the routed ports, the L3 EtherChannel — live in `PartV-configs.md` in this same folder instead of cluttering this file.

## Operating Cisco Routers (Ch 16)

The thing that clicked for me here — and it took me re-reading the CLI comparison table twice before it actually landed — is that a router and a switch CLI are basically the same coworker wearing a different uniform. Global config, interface config, `no shutdown`, console/VTY passwords, SSH keys — all of it carries over from the switch chapters. What's actually new is that a router's physical interfaces don't do anything until you tell them to. A switch port comes up ready to forward frames. A router interface sits there with no IP address and a default `shutdown` on it, refusing to do a single thing until you configure `ip address` and `no shutdown`. That default-off behavior is deliberate — Cisco doesn't want a half-configured router accidentally routing garbage onto a live network.

I also finally got the ISR versus Catalyst 8000 naming straight, which is something I'd been quietly confused about for a while and never bothered to look up. ISR (Integrated Services Router) is the older branding — router, switch, VoIP, security features all crammed into one box for a small site. Around 2020 Cisco introduced the Catalyst Edge Platform (8200/8300/8500 series) to replace some of that lineup, and the word "Catalyst" — which used to mean switch, full stop — now also means router if it's an 8000-series box. That's a genuinely annoying naming collision if you're Googling parts for a client site and don't know the history, and I've definitely pulled up the wrong datasheet because of it before.

The interface status codes are the part I actually use every day already, even outside CCNA. Line status and protocol status, both needing to say "up" before anything works:

- **administratively down / down** — somebody typed `shutdown` on it. Fix: `no shutdown`.
- **down / down** — no cable, bad cable, or the far end is dead/shutdown. This is a Layer 1 problem.
- **up / down** — cable's fine, but there's a Layer 2 mismatch, usually a duplex or encapsulation issue (HDLC vs. PPP on serial, most classically).
- **up / up** — working.

That table alone has saved me time on client calls. If a line status/protocol status combo says "up, down," I already know not to waste ten minutes checking a patch cable that was never the issue.

## Configuring IPv4 Addresses and Static Routes (Ch 17)

This chapter is the one that made routing tables stop being an abstract thing I could recite but not actually picture. The book's five-step router forwarding logic is worth internalizing because it's exactly what happens on every hop, every time, whether you're thinking about it or not:

1. Decide whether to even process the frame (FCS check, is the destination MAC mine).
2. De-encapsulate — strip the Ethernet header, keep the IP packet.
3. Match the destination IP against the routing table (longest prefix match wins).
4. Re-encapsulate the packet into a new frame for the outgoing interface.
5. Transmit it out that interface.

Step 3 is where the actual decision-making happens, and honestly it's also where I finally understood *why* connected routes and static routes coexist the way they do instead of one just replacing the other.

**Connected routes** show up automatically the moment an interface has an IP address and is up/up — no configuration needed beyond `ip address` and `no shutdown`. The router does the subnet math itself and adds a route (and a `/32` local route for its own address) without you asking. **Static routes** are the ones I actually type: `ip route destination-subnet mask next-hop-or-exit-interface`. There are four flavors that matter for the exam and for real troubleshooting:

- **Network route** — points at a specific subnet.
- **Default route** — `ip route 0.0.0.0 0.0.0.0 next-hop`, the catch-all for anything that doesn't match a more specific route.
- **Host route** — a `/32` pointing at one single address, useful when one host genuinely needs to take a different path than the rest of its subnet (VIP traffic routing around a firewall, that sort of thing).
- **Floating static route** — a backup route with an artificially high administrative distance so it only gets used when the primary (usually dynamic, usually OSPF in this book) route disappears. This is the "cellular backup kicks in when the fiber dies" pattern, and it's genuinely something I could see building for a client site with a 5G failover router.

The troubleshooting checklist near the end of this chapter is one I'd actually print out: is the subnet ID and mask right, is the next-hop IP actually on a neighboring router, does the outgoing interface exist on *this* router (not the next one over — that's a classic mistake), and is the interface up/up. A static route with a syntax IOS accepts but a destination that's wrong is the single easiest way to break routing without any error message telling you so.

```
Subnet 10.1.1.0/24              Subnet 10.1.4.0/30              Subnet 10.1.2.0/24
   .1                              .1      .2                       .1
 [PC-A]---[SW1]---[R1]=====WAN=====[R2]---[SW2]---[PC-B]
                     ip route 10.1.2.0 255.255.255.0 10.1.4.2
```

## IP Routing in the LAN (Ch 18)

This is the chapter where the book lays out four different ways to get packets moving between VLANs, and honestly all four show up in real networks depending on the size of the site:

- **Router-on-a-stick (ROAS)** — one physical router interface, trunked to a switch, carved into subinterfaces (one per VLAN) using `encapsulation dot1q`. Cheap, works fine for a branch office with a handful of VLANs, but every inter-VLAN packet has to leave the switch, hit the router, and come back — a real bottleneck if traffic volume is high.
- **Layer 3 switch with SVIs** — a switch that also routes. You give it `ip routing` globally and an `interface vlan X` (an SVI) for each VLAN, and now inter-VLAN routing happens inside the switch's ASIC instead of over a wire to an external router. This is what most of my client sites actually run once they outgrow the two-VLAN branch-office setup.
- **Layer 3 switch with routed ports** — instead of an SVI, you strip a physical port of its "switchport-ness" entirely with `no switchport` and give it an IP address directly, like it's a router interface. Makes sense for a single point-to-point link between two Layer 3 switches where there's no reason to have a whole VLAN just for that link.
- **Router with embedded switch ports** — small branch routers (the ISR1K in the book) that have a handful of built-in switched ports alongside routed ports. It's a Layer 3 switch and a WAN router squeezed into one chassis, which is basically what I'd spec for a five-person satellite office.

The native VLAN detail on ROAS tripped me up the first time through, so I want to note it here the way I'd actually explain it to someone at work instead of the way the book phrases it: an 802.1Q trunk tags every VLAN except one — the native VLAN — which rides untagged. On the router side you either configure the native VLAN's IP directly on the physical interface (no subinterface, no encapsulation command) or you use a subinterface with `encapsulation dot1q 10 native`. Either way, both ends of the trunk have to agree on which VLAN is native, or you get one of those bugs where things *mostly* work and nobody can explain why one VLAN is flaky. I can already picture the "why is only VLAN 20 acting weird" ticket this would eventually cause.

Troubleshooting ROAS specifically comes down to a short list I'd actually run through on a call: does the router have `encapsulation dot1q` on the right subinterface, does the switch side have that same VLAN allowed and untrunked, does the subinterface have an IP, does the native VLAN match on both ends, and is anything shut down that shouldn't be. Because neither side can see the other's misconfiguration directly — no status light tells you "the switch forgot about VLAN 20" — you end up checking both devices even when the router config looks perfect.

Full configs for ROAS, SVI, routed port, and Layer 3 EtherChannel are in `PartV-configs.md`.

```
        Trunk (802.1Q)
[SW1]==================[R1]
 VLAN10  VLAN20      Gi0/0.10 -> 10.1.10.1/24
                      Gi0/0.20 -> 10.1.20.1/24
```

## IP Addressing on Hosts (Ch 19)

DHCP is the chapter I already half-knew from doing onboarding automation at work, but seeing the actual packet flow filled in gaps I didn't know I had. DORA — Discover, Offer, Request, Acknowledge — and the reason it needs four messages instead of two comes down to a detail I hadn't thought about: the client doesn't have an IP address yet, so it can't just quietly accept an offer. It broadcasts the Discover to `255.255.255.255` from `0.0.0.0`, and if more than one DHCP server responds with an Offer, the Request message is actually broadcast too, specifically so every server on the segment can see which offer got accepted and the losers can withdraw their reservation.

The part that's genuinely useful for my day job is `ip helper-address`. Broadcasts don't cross routers by default, which is correct behavior for most broadcast traffic but is exactly the problem when your DHCP server lives in a data center and your clients are three routers away at a branch office. `ip helper-address` on the router's LAN-facing interface tells that router: catch DHCP broadcasts, rewrite the destination to the DHCP server's real IP, and forward it as a normal unicast packet. The return trip works because the DHCP server sends its reply back to the router's own IP (the one that appears as the source of the relayed request), and the router — recognizing it's a relay agent for that conversation — flips the destination back to broadcast so it reaches the actual client.

Routers and switches can be DHCP clients themselves too, which is the same `ip address dhcp` command whether it's on a switch's management VLAN interface or a router's WAN-facing interface talking to an ISP. When a router learns its address this way, it can also learn a default route from the DHCP-supplied gateway — that route shows up in `show ip route` with an administrative distance of 254, which is IOS's way of flagging "this default route came from DHCP, not from a static command."

For actually reading host settings, I keep landing back on the same four values no matter the OS — IP address, mask, default gateway, DNS servers — just retrieved differently:

| OS | Command |
|---|---|
| Windows | `ipconfig /all` |
| macOS | `ifconfig` + `networksetup -getinfo <interface>` |
| Linux | `ip address` + `ip route` (or the older `ifconfig` / `netstat -rn`) |

When a Windows client fails DHCP entirely, it self-assigns an APIPA address in the `169.254.x.x` range with a `/16` mask and just gives up on ever having a real gateway or DNS server — that's the single fastest visual tell that DHCP never completed, versus a client that completed DHCP but got handed *wrong* settings by a misconfigured server (which looks totally normal in `ipconfig /all` except the gateway or DNS entry is pointing at something that doesn't exist).

```
Branch subnet 10.1.1.0/24                  HQ subnet 10.1.12.0/24
   [PC]----[R1]============WAN============[R2]----[DHCP Server]
            ^ ip helper-address 10.1.12.2
```

## Troubleshooting IPv4 Routing (Ch 20)

This chapter is where ping and traceroute stopped being "commands I already know" and turned into "the two tools I'd actually reach for first on a call," which sounds obvious written out but wasn't obvious to me until I saw the mechanics behind them. The distinction that matters: ping tells you *whether* something's reachable, traceroute tells you *how far* a packet got before it died.

Ping rides on ICMP echo request/reply, and a standard ping from a router uses the outgoing interface's IP as the source — which means a working ping from R1's LAN interface doesn't actually prove the *reverse* route works, only the forward one. That's what extended ping is for: I can force the source address to something specific, like the LAN subnet's IP, and now the far end's reply has to find its way back to that subnet specifically. If the standard ping to a host works but an extended ping sourced from the far subnet fails, that's a strong signal the problem is a missing or wrong route on the return path, not the forward one.

Traceroute's mechanic is the part I actually enjoyed learning. It sends a burst of packets with TTL=1, and the first router in the path decrements TTL to 0, discards the packet, and sends back an ICMP Time Exceeded message — which is how traceroute learns hop #1's address. Then it sends TTL=2, gets a Time Exceeded from hop #2, and so on, incrementing TTL by one each round until a packet actually reaches the destination and gets a real reply instead of a Time Exceeded. It's genuinely just "how many routers does it take before this packet dies," turned into a diagnostic tool.

The layered troubleshooting order the book lays out is one I already use without thinking about it now: ping from a nearby router to rule out whether it's even an IP routing problem, extended ping to test the reverse route specifically, ping between LAN neighbors to confirm Layer 2 and ARP are working, then move on to Telnet or SSH between devices to check actual interface and routing table state by hand. And the detail I didn't know before this chapter — if a routing protocol is broken between two routers but the physical link and IP addressing on that link both still work, you can still Telnet or SSH hop through each router individually, because Telnet/SSH just needs the two ends of that one link to talk, not end-to-end routing.

## Tying it together — a troubleshooting scenario

Say I get a ticket: a branch office user can't reach a file server at HQ, but their coworker two desks over says everything's fine.

First move is ping, from the user's machine if I can get on a remote session, straight to the server's IP. It fails. So I RDP or SSH into the branch router (R1) and ping the same server IP from there. That works — so the WAN link, R1's routing, and the HQ side are all fine. The problem is specific to this one user's path from their PC to R1.

Next I ping the user's own default gateway from their machine. It fails too, and their coworker's ping to the same gateway succeeds. That rules out anything upstream of R1 entirely — this is a Layer 2 or local IP settings problem on this one workstation or its access port.

I check the user's IP settings with `ipconfig /all` and see a `169.254.x.x` address — classic APIPA, meaning DHCP never completed for this machine. So either the DHCP server ran out of leases in this subnet's pool, there's a bad cable or a shut access port between this PC and the switch, or — if this branch relies on a centralized DHCP server at HQ — the `ip helper-address` command is missing or pointing at the wrong IP on R1's interface for this VLAN. I check `show run interface` on R1 for the VLAN this user's port belongs to, confirm the helper address is there and correct, then check the switch port status with `show interfaces status` to rule out a physical or shutdown issue.

If the helper address and the port both look fine, the next step is confirming the DHCP server actually has an available range for this specific subnet — sometimes it really is just an exhausted pool, and the fix is boring: extend the scope. But I wouldn't have gotten there without walking the path in order — server, WAN router, gateway, local port, DHCP settings — the same order the book teaches with ping, extended ping, and interface status checks, because skipping a step means guessing instead of isolating.

This is also about the point in the book where CCNA stopped being an exam I was studying for and started being language I already needed at work. I'd been resetting passwords and provisioning mailboxes for a while before this, but I didn't have a real answer for *why* a branch office loses connectivity or what a helper-address is actually doing under the hood — I just knew "reboot it" worked more often than it should have. Now when a ticket says "user can't reach the shared drive," I've got an actual order of operations instead of a guess.
