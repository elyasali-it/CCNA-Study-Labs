# Part IV: IPv4 Addressing

This part is where the book stops talking about Ethernet and starts talking about how IP actually splits a network into pieces. Chapters 1-10 were about wires, switches, and MAC addresses that just exist without anyone planning them. Subnetting is different. Someone has to sit down and design it, and then everyone else operating the network has to be able to read that design back out of an IP address and mask. That's really the whole arc of this part: Chapter 11 is the design side, Chapters 12-14 are the "here's an address, tell me about its network" side, and Chapter 15 comes back around to design with the full formal process.

## Chapter 11: Perspectives on IPv4 Subnetting

The book opens this chapter with a sandwich analogy - you buy one giant sandwich, it's too much for one person, so you cut it into pieces and hand them out. That's subnetting. You start with one classful network and chop it into smaller pieces so different parts of your internetwork can each have their own piece.

What actually stuck with me here is the four-question framework for planning:
1. Which hosts get grouped into the same subnet?
2. How many subnets do I need total?
3. How many host addresses does each subnet need?
4. Do I use one subnet size for everything, or different sizes?

The rule for question 1 is simple once you see it: anything on the same physical LAN, not separated by a router, has to be in the same subnet. The second a router sits between two devices, they're in different subnets, no exceptions. So when you're counting how many subnets you need, you're really counting locations that need one - every VLAN, every point-to-point WAN link, every Ethernet WAN connection between two routers. Each of those gets its own subnet.

The one-size-fits-all decision matters more than it sounds. If you use a single mask across the whole network, you have to size it for your biggest subnet, which means smaller subnets end up wasting a bunch of addresses they'll never use. The book's example: if one LAN needs 200 hosts but your branch offices only need 50, and you use one mask sized for 200 everywhere, those branch subnets are sitting on a pile of addresses they don't need. That's the tradeoff for keeping the math simple - everyone on the team only has to remember one mask.

Public vs private IP came up here too, and honestly this is a fact I hadn't really thought about the "why" of before: the internet ran out of room. By 2011, ARIN and the other regional registries had handed out the last of the public IPv4 blocks. RFC 1918 gives every organization a chunk of private address space they can reuse internally - one Class A network (10.0.0.0), sixteen Class B networks (172.16.0.0-172.31.0.0), and 256 Class C networks (192.168.0.0-192.168.255.0). NAT is what lets two completely different companies both use 10.0.0.0 internally and still both reach the internet without colliding. In the MSP world this is just... every client network I've ever touched. Every single one is sitting on some private range, and NAT at the edge is doing the translation work quietly in the background.

## Chapter 12: Analyzing Classful IPv4 Networks

This chapter is basically: here's an IP address, now tell me everything about the network it lives in - class, default mask, network ID, broadcast address, host range - without doing any subnetting math yet. Just classful facts.

The class ranges are things I mostly had memorized already from A+/Network+, but seeing them laid out with the exceptions was useful. Class A is 1-126, Class B is 128-191, Class C is 192-223. Notice 127 is missing - that whole range is reserved because of the loopback address (127.0.0.1), and 0.0.0.0 is reserved too. Those two gaps trip people up if they don't know to expect them.

The part that actually requires a little care is spotting network IDs and broadcast addresses that "look wrong" for their class. Class B network 128.0.0.0 has three zeros at the end, so it looks like it should be a Class A network ID - but the first octet is 128, and 128 falls in the Class B range, so it's Class B whether it looks that way or not. Same trap in reverse with something like 191.255.255.255 - three 255s at the end makes it look like a broadcast address for a Class A network, but it's actually the broadcast for the highest Class B network. The lesson is: the first octet always wins, don't let the zeros or 255s at the end fool you into guessing the wrong class.

The six-step process for deriving network ID and broadcast from any address is worth having memorized cold:
1. Determine class from the first octet
2. Mentally divide network and host octets based on that class
3. To get the network number, copy the address and zero out the host octets
4. Add 1 to the last octet of the network ID to get the first usable address
5. To get the broadcast, copy the address and set the host octets to 255
6. Subtract 1 from the broadcast to get the last usable address

CIDR blocks also got introduced here as the newer alternative to strict classful assignment - instead of handing a company an entire Class A, B, or C network, a registry can hand out any power-of-2 sized block. That's the difference between a company getting the whole 1.0.0.0 network versus getting something like 1.64.0.0-1.127.255.255, a chunk carved out of what used to be one Class A network. Same math skills apply either way, just a different starting boundary.

## Chapter 13: Analyzing Subnet Masks

This is where mask conversion gets nailed down properly - binary, decimal (dotted-decimal notation), and prefix (the /24 style) are three ways of writing the exact same thing, and you need to move between all three without hesitating.

The rule that makes binary masks legal is really simple: no interleaving. Once you hit a 0 bit, every bit after it has to also be 0. You can't have 1s appear again after a 0 shows up. So 11111111 11111111 11000000 00000000 is legal, but anything with 1s scattered after 0s isn't a valid mask at all.

For converting, I liked the shortcut the book pushes hardest: memorize that a subnet mask only ever has nine possible values in any given octet - 0, 128, 192, 224, 240, 248, 252, 254, 255 - corresponding to 0 through 8 binary 1s. Once that table is in your head, converting decimal to binary and back stops being math and starts being lookup. That's honestly the same instinct as memorizing powers of 2 for subnetting - front-load the memorization so the exam-day math is just recall, not calculation.

The other big concept here is classless vs classful addressing, and I want to be precise about the difference because it's easy to blur:
- **Classless addressing**: the address has two parts - prefix and host - with no consideration of class at all. Just look at where the mask's 1s stop.
- **Classful addressing**: the address has three parts - network, subnet, and host - because you're applying Class A/B/C rules on top of the prefix to split it further.

Same address, same mask, two different ways of describing its structure depending on whether you're layering class rules on top or not.

## Chapter 14: Analyzing Existing Subnets

Chapter 13 taught you what a mask looks like. This chapter is about what you do with a mask you already have - given an IP address and a mask, find the subnet ID, the broadcast address, and the range of usable addresses. The book calls this the resident subnet, meaning the specific subnet that address actually lives in.

Binary process, step by step:
1. Convert the mask to prefix format to know how many prefix bits (P) and host bits (32-P) you're working with
2. Convert the IP address to 32-bit binary
3. Copy the prefix bits exactly as they are
4. Set every host bit to 0 - that binary result is the subnet ID
5. Convert the whole 32-bit result back to decimal, 8 bits at a time

For the broadcast address it's the identical process except step 4 flips - you set every host bit to 1 instead of 0. Same walk, opposite ending.

The decimal shortcut is the one I actually want to get fast at, because doing full binary conversion every single time is slow. The logic:
- If a mask octet is 255, just copy the IP address's value straight over - it's already right
- If a mask octet is 0, write down 0 for the subnet ID (or 255 for the broadcast)
- Only the "interesting octet" - the one where the mask value isn't 0 or 255 - actually requires real work

That's the whole point of the "interesting octet" concept: two out of the four octets are almost always trivial copy-paste, one is usually all zeros or all 255s, and only one octet needs the magic number math. The magic number itself is just 256 minus the mask's decimal value in that interesting octet. Once you have it, you find the multiple of the magic number that's closest to the IP address's value in that octet without going over - that's your subnet ID's value there. For the broadcast, you take that subnet ID value, add the magic number, and subtract 1.

Worked example from the book that made it click: address 172.16.150.41, mask 255.255.192.0. The third octet is interesting because 192 isn't 0 or 255. Magic number is 256-192=64. Multiples of 64 are 0, 64, 128, 192 - and 128 is the closest one to 150 without going over. So the subnet ID is 172.16.128.0. Broadcast just runs the same math with 1s instead of 0s: 172.16.191.255.

## Chapter 15: Subnet Design

This chapter closes the loop and goes back to design, but now with a full formal process instead of the conceptual overview Chapter 11 gave. The question it answers: given a required number of subnets and a required number of hosts per subnet, which mask(s) actually work?

There are three possible outcomes when you go looking for a mask that meets requirements: no mask meets it, exactly one mask meets it, or several masks meet it - and each one matters for different reasons.

**No masks meet requirements** happens when the total bits needed (network + subnet + host) blow past 32. The book's example: Class B network needing 300 subnets and 280 hosts per subnet. That needs 9 subnet bits (2^9=512 ≥ 300) and 9 host bits (2^9-2=510 ≥ 280), plus the 16 network bits Class B already locks in. 16+9+9=34. There's no fitting that into a 32-bit mask - the design just isn't possible as stated, and you'd have to either accept fewer hosts/subnets or split into multiple classful networks.

**Exactly one mask meets requirements** happens when the minimum subnet bits and minimum host bits add up to exactly what's left after the network bits. No slack, one answer.

**Multiple masks meet requirements** is the more interesting case because now you have a choice, and the choice is a real tradeoff:
- Shortest prefix (fewest subnet bits) → maximizes hosts per subnet, minimizes subnets
- Longest prefix (most subnet bits) → maximizes subnets, minimizes hosts per subnet
- Anything in between → balances both, useful if you expect roughly proportional growth in both subnets and hosts

The three-step process to find the range: calculate the shortest prefix mask using the minimum S, calculate the longest prefix mask using the minimum H, and every prefix length between those two boundaries is a valid answer.

The formal six-step process for the whole thing:
1. Find N (network bits) from the class
2. Calculate minimum S so 2^S ≥ required subnets
3. Calculate minimum H so 2^H-2 ≥ required hosts/subnet
4. If N+S+H>32, no mask works
5. If N+S+H=32, exactly one mask works: P=N+S
6. If N+S+H<32, multiple masks work, and you pick where in that range you want to land

Once the mask is chosen, finding all the actual subnet IDs uses the exact same magic-number logic from Chapter 14, just run forward instead of backward. Start at the zero subnet (which is numerically identical to the classful network ID itself), then keep adding the magic number to the interesting octet until you hit 256 - that overflow point is where you stop, and the subnet right before it is called the broadcast subnet, the numerically highest subnet in the whole design.

One naming thing worth remembering: "zero subnet" and "broadcast subnet" both sound like they should be avoided based on the words alone, but they're both completely valid, usable subnets. Some older shops disabled the zero subnet with `no ip subnet-zero` because of confusion between "the whole classful network" and "just its first subnet," but by default both ends of the subnet ID range are fair game.

## How I'd troubleshoot this in the real world

Say a client calls in and says a new branch office can't reach anything past their local router - the PCs get IPs fine, but nothing routes out. First thing I'd want is the IP address and mask off one of the affected PCs, because that's Chapter 14 territory: run the decimal shortcut, find the resident subnet, and compare it against what the subnet plan document says that branch should be on. If the DHCP scope handed out an address that doesn't match the subnet the router's interface is actually configured for, that's the whole problem right there - the PC thinks it's in one subnet, the router thinks the LAN is a different one, and nothing will route until those two agree.

If the address and mask do line up correctly, next I'd pull up the original design (Chapter 15 thinking) and check whether this branch was ever actually accounted for in the subnet plan, or whether someone hand-typed a mask on the fly that doesn't match the mask used everywhere else in the network. A mismatched mask between two ends of the same link is a classic case of "it looks like it should work" but doesn't, because the subnet ID each device calculates for that link comes out different, and Chapter 11's core rule - hosts on the same link must be in the same subnet - gets silently broken. That's usually where I'd find it: not something exotic, just two devices disagreeing on where the subnet boundary actually is.
