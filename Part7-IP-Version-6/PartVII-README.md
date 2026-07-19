# Part VII: IP Version 6

This part is where I moved from treating IPv6 as "the addressing scheme with the long hex numbers" to actually understanding why it exists and how a router handles it differently from IPv4. Chapters 25 through 29 cover the fundamentals, the addressing and subnetting math, how routers implement it, how hosts get their settings dynamically, and finally how routing works once addresses are in place.

## Why IPv6 exists in the first place

IPv4 gives the world about 4.3 billion addresses. That sounded enormous in the 1980s and became a real problem once the internet started connecting every phone, camera, and thermostat on the planet. NAT and CIDR stretched IPv4's life by years, but they were patches, not a fix. IPv6 is the actual fix: a 128-bit address space instead of 32-bit, which works out to roughly 340 undecillion addresses. I don't think in undecillions day to day, but the practical takeaway is that address exhaustion stops being a design constraint.

What surprised me is how much of IPv6 is a straight port of IPv4 concepts with new packaging. Subnetting still exists. Routing tables still work on longest-prefix match. Routers still forward based on destination address against a routing table. The header even got simpler in some ways even though it holds bigger addresses, because a handful of fields that used to require router processing (like fragmentation) got pushed elsewhere in IPv6.

The pieces that did change: ARP is gone and NDP replaced it, ICMP became ICMPv6, and OSPFv2 became OSPFv3 to handle the new address format. When I see "IPv6" I now think of it as IPv4's core ideas with a bigger address field and a couple of protocol swaps, not as an unrelated system.

## Reading and writing the addresses

An IPv6 address is 32 hex digits split into eight groups of four, separated by colons. Two abbreviation rules make these livable:

Inside any single group, drop leading zeros, so `0100` becomes `100` and a group of all zeros becomes a single `0`.

Across the whole address, one run of consecutive all-zero groups can collapse to `::`, and only one run gets that treatment even if there are two separate runs, you use it on the longest one and spell the rest out.

The mistake I had to train myself out of was trimming trailing zeros within a group instead of leading ones. `FE80` can't shorten to `FE8`, but it can turn into just `FE80` staying as-is if there's nothing to drop from the front. It's a small rule but it's exactly the kind of thing that trips people up on the exam and in real command output.

Prefix length works the same way it does in IPv4, just described as bits instead of a dotted mask. To find a subnet prefix from an address and a prefix length that's a multiple of 4, I copy the hex digits that fall inside the prefix and zero out the rest. A /64 on `2001:DB8:1111:1::1` gives a subnet prefix of `2001:DB8:1111:1::/64`, which is the overwhelming default you'll see on every LAN subnet in practice, since nearly every IPv6 RFC steers you toward /64 for anything with hosts on it.

## Global unicast and the three-part structure

A global unicast address is IPv6's version of a public IPv4 address, and it's built in three pieces: a global routing prefix assigned by an ISP or RIR, a subnet ID that the local network engineer carves out, and a 64-bit interface ID that identifies the host. The math clicked for me once I stopped comparing it to classful IPv4 and instead thought of it as: however many bits the provider gave me for the prefix, I get to use everything up to the 64-bit boundary for subnetting, and the last 64 bits belong to host addressing.

If a company gets a /48 from its ISP, that leaves 16 bits for the subnet field, which is 65,536 possible subnets, more than most organizations will ever need even at that supposedly small allocation. That's a different mindset than IPv4 subnetting where you're constantly negotiating between subnet count and host count. With a /64 host boundary basically locked in, IPv6 subnetting becomes "how many subnet bits do I have" rather than "how do I split scarce host bits."

Unique local addresses are the private-IPv4 equivalent, always starting with `FD` and meant to be randomly generated rather than made up, specifically so that two companies merging their networks don't end up with the same private range colliding the way so many `10.0.0.0` networks do today. I've seen the temptation to just type something memorable like `FD00:1:1::/48` for a lab, and that's fine for a lab, but it defeats the entire purpose of the ULA design in a production network.

## What a router does when you configure an interface

Configuring `ipv6 address <address>/<prefix-length>` on an interface does four things at once: it assigns the interface a routable address, turns on IPv6 forwarding for that interface, calculates the local subnet from the address and prefix, and adds a connected route for that subnet to the routing table. That's the same bundle of side effects IPv4 configuration triggers, and once I saw it laid out that way, IPv6 interface config stopped feeling like a separate skill from IPv4 interface config.

There are two ways to hand the router the host portion of the address. You can type the full 128 bits yourself, or you can give the router just the /64 prefix and the `eui-64` keyword and let it build the interface ID from the MAC address. EUI-64 splits the 6-byte MAC in half, drops `FFFE` in the middle, and flips the seventh bit of the first byte. That bit flip is the part people forget exists, and it's the difference between an interface ID that looks right and one that's actually correct. I keep a binary/hex conversion chart handy for exactly this reason, because eyeballing a bit flip in hex is asking for a mistake.

Router-level IPv6 forwarding is off by default even though IPv4 forwarding is on by default, which is backwards from what I expected the first time I configured a lab router and couldn't figure out why nothing was routing. `ipv6 unicast-routing` is a one-time global command and it's easy to forget on a fresh device.

## Link-local, multicast, and the addresses I didn't know existed

Every IPv6-enabled interface auto-generates a link-local address starting with `FE80::`, whether or not it has a routable GUA or ULA. LLAs never leave the local link, routers won't forward them, and that's by design, not a limitation. What I didn't expect is how much real work LLAs do: they're the source and destination for NDP, and they're commonly used as the next-hop address for both routing protocols and static routes on point-to-point WAN links, because a WAN link between two routers doesn't need a globally routable subnet at all if the only thing crossing it is router-to-router traffic.

IPv6 replaced broadcast entirely with multicast. There's no IPv6 equivalent of `255.255.255.255`. Instead you get well-known addresses like `FF02::1` for all IPv6 nodes on the link and `FF02::2` for all routers, plus protocol-specific ones like `FF02::5` and `FF02::6` for OSPFv3. The solicited-node multicast address is the one that took the longest to click: every unicast address gets a matching solicited-node multicast address built from its last 24 bits, and that's what NDP uses instead of a broadcast ARP request, so only the host that actually owns the target address has to process the message instead of every host on the subnet.

Anycast was the genuinely new concept for me. Multiple routers configure the identical address, and the network just routes traffic to whichever instance is topologically closest. It's built for services that exist redundantly across several routers, and the elegant part is that from the sending host's point of view it's just a normal-looking unicast destination.

## How hosts actually learn their settings without a human touching them

This is where NDP earns its keep beyond neighbor discovery. A host needs four things to function on IPv6: an address, a prefix length, a default router, and a DNS server. IPv6 gives you two philosophically different ways to deliver all four.

Stateful DHCPv6 is the direct descendant of DHCPv4: a server leases an address and tracks state about which client has which lease, using Solicit, Advertise, Request, and Reply messages instead of DORA.

SLAAC flips the model. The host learns the subnet prefix from a router's Router Advertisement, then builds its own interface ID either through EUI-64 or through a randomly generated value, runs Duplicate Address Detection to make sure nobody else on the link already claimed that address, and only then starts using it. Nobody hands the host an address, the host makes one for itself and just checks that it's safe to use.

DAD itself is a clean piece of design once you see the whole exchange: a host about to use an address sends an NS message asking, essentially, "is anyone here already using this?" Silence means the address is safe. A reply means duplicate, stop. It's the same NS/NA mechanism used for ordinary neighbor discovery, just aimed at your own address instead of someone else's.

In practice most modern hosts run SLAAC for the address itself and lean on stateless DHCPv6 or RA-based DNS configuration just to pick up the DNS server list, since SLAAC alone doesn't carry DNS information. That combination shows up constantly in real Windows and macOS output, where you'll see one permanent address plus a rotating temporary address, since RFC 8981 has hosts cycle temporary addresses every so often specifically to make them harder to track across sessions.

## Getting packets from one subnet to another

Once addressing works, routing is close to a copy-paste of IPv4 logic with new syntax. Routers build connected and local routes automatically from any working interface's unicast address configuration, the same as IPv4, with the one difference being that local routes use a /128 prefix (exactly one address) instead of IPv4's /32.

Static network routes use the `ipv6 route` command, and there are three legal ways to tell the router where to send matching traffic: name the outgoing interface, name the next-hop GUA or ULA, or name the next-hop link-local address plus the outgoing interface. That third option surprised me until I understood why it's required: a link-local address by itself isn't unique across the whole router, so IOS needs the outgoing interface to know which link's LLA table to check. You cannot configure an LLA as the sole next-hop with no interface, IOS will reject it outright.

Ethernet interfaces have a quirk worth remembering: `ipv6 route` with only an outgoing interface and no next-hop address will be accepted by IOS on an Ethernet link, but it won't actually forward packets correctly, because Ethernet is a multi-access medium and the router has no way to know which neighbor on that segment should receive the frame. Serial point-to-point links don't have that problem since there's only one possible next hop. That's a distinction I didn't appreciate until I saw it laid out as a troubleshooting checklist item, since it's exactly the kind of route that looks completely fine in the configuration and in `show ipv6 route`, yet quietly fails in the field.

Default routes use `::/0` the same way IPv4 uses `0.0.0.0/0`, and floating statics work identically to their IPv4 counterparts too: configure a higher administrative distance than the primary route source so the static route only becomes active if the primary (usually a routing protocol) route disappears. A cellular backup interface sitting there unused until the primary WAN link fails is the classic use case, and it's one I can picture directly against MSP client environments running dual-carrier failover.

## Troubleshooting scenario: the static route that looks right but doesn't work

Say a client calls in because two sites aren't talking over IPv6, even though both static routes are sitting in the configuration and both show up in `show ipv6 route`. Before touching anything, I'd walk the same checklist I'd use for IPv4 with IPv6-specific twists layered in.

First question: does the route even show up in the routing table, or does it show up in the config but not the RIB? A route configured with an outgoing interface that's currently down won't make it into the table at all, since IOS pulls connected and local routes only from interfaces that are actually up/up, and a static route referencing that interface follows the same rule. If the route is missing entirely, that's usually the answer right there.

Second: if it is in the table, is the prefix and prefix length actually correct? A typo in the fourth hex quartet is an easy way to end up with a route that's syntactically fine and semantically pointing at the wrong subnet entirely, and IOS has no way to catch that for you since it only validates syntax, not intent.

Third: if the route uses a next-hop LLA, is there an outgoing interface listed alongside it? A next-hop LLA alone will be rejected outright, but a next-hop LLA that's simply the wrong router's LLA, or that points at the wrong local interface, will be silently accepted and just won't forward correctly, since there's no route table for LLAs to catch the mismatch.

Fourth: if the route uses a next-hop GUA or ULA instead, does the local router actually have a working route to reach that next-hop address in the first place? IOS does an iterative lookup here, first matching the packet to the static route, then matching the next-hop address in that static route to some other route (usually a connected route) to figure out the real outgoing interface. If that second lookup fails, the static route effectively goes nowhere even though it's sitting in the table looking legitimate.

And last, if it's an Ethernet WAN link specifically, is the route relying on outgoing-interface-only forwarding, which technically gets accepted by IOS but never actually works on a multi-access segment. Swapping to a next-hop address, or LLA plus interface, is the fix.

Nine times out of ten on a real network the culprit is the LLA-plus-wrong-interface case or the interface-down case, both of which look completely fine on a config review and only reveal themselves once you actually check interface status and trace the next-hop lookup by hand.
