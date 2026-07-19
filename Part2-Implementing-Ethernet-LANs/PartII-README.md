# Part II: Implementing Ethernet LANs

**Covers:** Chapter 4 – Using the Command-Line Interface · Chapter 5 – Analyzing Ethernet LAN Switching · Chapter 6 – Configuring Basic Switch Management · Chapter 7 – Configuring and Verifying Switch Interfaces

Part I was all theory. This part is where I actually got my hands on a switch — logging into the CLI, watching it learn MAC addresses, locking it down with passwords and SSH, and messing with interface settings. This is where things stopped being abstract for me.

## Getting into a switch and controlling it

A Cisco switch runs an OS called IOS. You talk to it through the CLI, and you get there one of three ways: console cable, Telnet, or SSH. Console is the physical connection — you plug directly into the switch, usually with USB now instead of the old rollover cable. Telnet and SSH both go over the network, but Telnet sends everything as plain text, including your password. SSH encrypts it. I don't use Telnet outside a lab anymore, there's just no reason to.

Once you're logged in, you're in one of two modes. User mode is a `>` prompt and it's basically look-but-don't-touch. Enable mode (also called privileged mode) is a `#` prompt, and that's where you can actually change things. You get there by typing `enable` and entering the enable password.

From enable mode, `configure terminal` drops you into global config, and from there you branch into more specific modes depending on what you're touching — interface config, line config for the console or vty ports, VLAN config. The prompt tells you where you are, so `(config-if)#` means you're inside a specific interface right now.

One thing that tripped me up early on: there are two config files. The running-config is what's active right now, sitting in RAM. The startup-config is in NVRAM and it's what loads when the switch boots. If you make changes and don't run `copy running-config startup-config`, and the switch loses power, everything you did is gone. I've made that mistake exactly once.

Also worth knowing: `?` by itself lists everything you can type from where you're standing. `command ?` shows you what comes next for that specific command. Tab completes what you're typing. I use these constantly instead of trying to memorize every command cold.

## What a switch is actually doing

A switch has one job: look at the destination MAC address on a frame and figure out where to send it. Everything else exists to support that.

Here's the way I picture it. Imagine a new receptionist who doesn't know anyone in the building yet. A call comes in for someone she doesn't recognize, so she pages the whole building — that's flooding. But every time someone walks by and mentions their name, she jots it down. After a while she knows exactly which office to send each call to without paging anyone.

That's learning and forwarding. The switch looks at the source MAC on every incoming frame and remembers which port it came in on. If it already knows where the destination MAC lives (a known unicast), it sends the frame straight to that one port. If it doesn't know the destination, or the frame is a broadcast, it floods it out every port except the one it arrived on.

Entries don't stick around forever either — there's an aging timer, 300 seconds by default, and if the switch doesn't hear from that MAC again in that window, the entry gets dropped. And if you've got more than one switch on the network, each one is building its own MAC table completely on its own, based only on what passes through its own ports.

The command I actually use to check all this is `show mac address-table`. When someone tells me a device can't reach something, checking whether that device's MAC even shows up where I expect is usually one of my first moves.

## Locking a switch down and making it reachable

A brand new switch has no security on it at all. Anyone with a console cable has full control. This chapter is basically about closing that gap.

The simplest option is shared passwords — one password for console, one for Telnet/SSH access (that's the vty password), and a separate enable password to get into privileged mode. No usernames involved, everyone shares the same password.

A step up from that is local usernames. Instead of one shared password, you configure individual username/password pairs directly on the switch and turn on `login local`, so it checks logins against that list. Beyond that there's AAA — an external server that holds all the usernames and passwords centrally, so you're not managing credentials on every single device separately. That's what real production networks lean on.

To turn on SSH you need three things set up first: a hostname, a domain name, and a generated key pair using `crypto key generate rsa`. That key is what makes the encryption work. I always follow that with `ip ssh version 2`, since version 1 has known weaknesses.

A Layer 2 switch doesn't technically need an IP address to do its main job of switching frames, but you need one anyway so you can actually manage it remotely. That address goes on a virtual interface, usually VLAN 1 — think of it as the switch giving itself a virtual NIC. You also have to set a default gateway on the switch, same as you would on any regular PC, or it won't be able to reach anything outside its own subnet.

There's also a handful of small quality-of-life commands I add in a lab that aren't really about security — things like `logging synchronous` so log messages don't interrupt whatever you're typing, `exec-timeout 0 0` so the session doesn't time out while you're working, and `no ip domain-lookup` so a mistyped command doesn't hang for a minute trying to resolve as a hostname.

## Speed, duplex, and reading a port

Every port has to agree with whatever's plugged into it on speed and duplex. Autonegotiation handles that automatically most of the time. The problems start when it's disabled on one end but not the other.

Picture two people meeting and quickly checking which language they both speak before talking. Both sides declare what they support, then use the fastest speed and best duplex they have in common. That's autonegotiation working normally.

Now picture one person just starts talking without checking first. The other person has to guess the language based on context. Sometimes that guess is wrong. That's basically what happens when one side has autonegotiation off — the other side can actually sense the speed by reading the electrical signal, but it can't detect duplex that way, so it falls back to a default. Half duplex if the speed is 10 or 100 Mbps, full duplex otherwise. That guess is where duplex mismatches come from.

A duplex mismatch is sneaky because the link still comes up fine — status shows connected, everything looks normal — but performance is bad, with errors and retransmissions that don't look like an obvious outage.

Auto-MDIX is a separate thing worth knowing: it lets a port automatically sense and fix a cable that's the wrong type (straight-through where a crossover was technically needed). It's on by default on Cisco switches, which is one more reason I leave speed and duplex on auto unless I have a real reason not to.

For managing interfaces day to day: `description` to note what's plugged in, `shutdown` / `no shutdown` to disable or enable a port, `interface range` to configure a batch of ports at once instead of one by one, and putting `no` in front of any subcommand to revert it back to default.

`show interfaces status` gives you the quick one-line-per-port summary — connected, notconnect, disabled. `show interfaces` gives you way more detail, including error counters. The one I pay closest attention to is late collisions, because that specific counter is one of the strongest signs of a duplex mismatch.

## Working through a real scenario

Say a user tells me their connection to a nearby server feels slow and drops occasionally, but everything looks "up."

First I'd SSH into the switch and check `show mac address-table` to confirm the user's device is actually showing up on the port I think it's on. If it's not there, the problem's probably not even at this switch.

Next, `show interfaces status` on that port. If it says connected, I don't stop there, because a mismatch hides behind a status that looks fine.

Then I'd pull `show interfaces` on that specific port and look at the collision counters. Late collisions climbing is my strongest clue that this is a duplex mismatch and not just a bad cable.

From there I'd check whether speed or duplex got manually set on one end and left on auto on the other — that's the classic setup for this exact problem. If I find it, the fix is either putting both ends back on autonegotiation, or manually matching both ends to the same speed and duplex. Either way, I'd add a description on that interface noting what's connected, so whoever looks at this port next doesn't have to work it out from scratch.
