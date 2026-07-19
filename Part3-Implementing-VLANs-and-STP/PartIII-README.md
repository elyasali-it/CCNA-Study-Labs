# Part III: Implementing VLANs and STP

**Covers:** Chapter 8 – Implementing Ethernet Virtual LANs · Chapter 9 – Spanning Tree Protocol Concepts · Chapter 10 – RSTP and EtherChannel Configuration

Part I taught me how data moves. Part II taught me how a single switch decides where to forward a frame. Part III is where things start to look like a real building — multiple switches, multiple VLANs, redundant links between closets, and all the mechanisms that keep that redundancy from turning into a network-wide meltdown. This is the part of the CCNA where "why does the network fall over when I add a second cable between switches" finally gets answered.

---

## 1. VLANs: Splitting One Switch Into Many Broadcast Domains

**The concept:** A LAN and a broadcast domain are basically the same thing from a switch's point of view. Out of the box, every port on a switch belongs to the same broadcast domain — a broadcast sent by one device gets copied out every other port. A VLAN (virtual LAN) is how a switch splits its own ports into separate broadcast domains without needing a separate physical switch for every group of users.

**Real-world analogy:** Picture an office building with one open floor. Without VLANs, it's like nobody put up walls — a shout in the accounting corner is heard by everyone in sales, engineering, and the break room. VLANs are the drywall. Accounting is walled off from sales, sales is walled off from engineering, and a shout in one room stays in that room. The building (the switch) didn't change — just how it's divided.

**Why bother splitting things up:** fewer devices per broadcast domain means less wasted CPU processing broadcasts, smaller failure domains when something goes wrong, tighter security policies per group, and the ability to group people by department instead of by which port they happen to plug into.

**The process, on a single switch:**
1. Create the VLAN with `vlan vlan-id` in global config mode.
2. Optionally name it with `name vlan-name`.
3. Assign each port to that VLAN with `switchport access vlan id-number` in interface mode.
4. Optionally lock the port out of trunking with `switchport mode access`.

**Why it works:** the switch tags every port internally with a VLAN membership, and its forwarding logic simply refuses to send a frame out a port that isn't in the same VLAN as the port it arrived on. Two VLANs on one switch behave exactly like two separate physical switches — the hardware just enforces the wall in software instead of with a second box.

---

## 2. VLAN Trunking: Carrying Multiple VLANs Over One Cable

**The concept:** The moment a VLAN needs to exist on more than one switch, I need a way to send traffic from several VLANs across a single physical link between switches. Without trunking, I'd need one dedicated cable per VLAN between every pair of switches — that doesn't scale past a handful of VLANs.

**The process:** a trunk link uses 802.1Q tagging. Before a switch sends a frame out a trunk port, it inserts a 4-byte 802.1Q header into the Ethernet frame that lists a VLAN ID. The receiving switch reads that tag, knows which VLAN the frame belongs to, strips the tag back off, and forwards the original frame out the correct access ports.

**Why it works:** the tag is the only thing that changes. The switches agree ahead of time (via configuration, or via negotiation with DTP) that a link is a trunk, and once that's settled, one physical cable can safely carry frames for every VLAN configured on both switches, each one clearly labeled as it crosses.

**One wrinkle worth remembering — the native VLAN.** 802.1Q reserves one VLAN per trunk (VLAN 1 by default) that does *not* get tagged. If the native VLAN configured on one end doesn't match the other end, frames can quietly jump from one VLAN to another — a bug called VLAN hopping that's easy to create by accident and annoying to track down if I don't know to check for it.

**Configuration, in practice:**
- `switchport trunk encapsulation dot1q` — pick the tagging method (most modern switches only support 802.1Q anyway).
- `switchport mode trunk` — force the port to always trunk, or use `dynamic desirable`/`dynamic auto` to let DTP negotiate.
- `switchport trunk native vlan vlan-id` — set the native VLAN, and make sure it matches on both ends.
- `switchport trunk allowed vlan vlan-list` — optionally restrict which VLANs are actually allowed across, instead of allowing all 4094 by default.

The Cisco best practice I'm keeping in mind: disable negotiation on ports that should always trunk, using `switchport nonegotiate`, since leaving DTP running on every link is one more thing that can silently misconfigure itself.

---

## 3. Voice VLANs: One Port, Two Devices, Two VLANs

**The concept:** IP phones complicate the tidy access-port-vs-trunk-port picture. A phone typically sits between the wall jack and a user's PC, with a tiny embedded switch inside it — one cable comes from the wiring closet, and the phone passes a second connection through to the PC. Cisco's answer is to let a single switch port act like an access port for the PC's data and like a mini-trunk for the phone's voice traffic, tagging only the voice frames with 802.1Q.

**The process:** `switchport access vlan id-number` sets the data VLAN like normal, and `switchport voice vlan id-number` adds the second VLAN specifically for the phone. The switch then expects untagged frames from the PC (data VLAN) and 802.1Q-tagged frames from the phone (voice VLAN), both arriving on the same physical port.

**Why it works:** it lets IT run a single cable to a desk instead of two, while still keeping voice and data traffic in separate broadcast domains — useful for QoS prioritization and for not mixing an IP phone's traffic in with general user data.

---

## 4. Why Redundant Links Break Everything Without STP

**The concept:** Once a LAN has more than one switch, engineers naturally want a backup path — a second cable between two switches (or three switches in a triangle) so that if one link dies, the network keeps working. The problem: switches don't know about "backup" links. By default they forward every frame out every other port in that VLAN. With a loop in the topology, a single broadcast frame gets copied around the loop forever.

**Real-world analogy:** it's the office intercom system with a wiring mistake where the output of speaker A feeds back into speaker B's microphone, and B's output feeds back into A's microphone. One person coughs near a mic, and that cough echoes through the building nonstop until somebody physically pulls a cable.

**What actually breaks, specifically:**
- **Broadcast storms** — the looping frame gets duplicated over and over, eventually consuming all the bandwidth on every link in the loop.
- **MAC table instability** — because the same source MAC keeps arriving from different directions as the frame loops, switches keep flipping their MAC table entry for that address back and forth, meaning they can't reliably deliver traffic *to* that device anymore either.
- **Multiple frame delivery** — the destination device receives many copies of what was supposed to be a single frame, which can confuse or crash applications that aren't expecting duplicates.

**Why it works (or rather, why STP prevents it):** Spanning Tree Protocol's whole job is to look at a topology with physical loops and calculate a *logical* tree with no loops, by putting certain switch ports into a blocking state. The physical redundancy stays in place as a backup, but only one active path exists at any moment for any given VLAN, so a broadcast frame has exactly one way to travel and nowhere to loop back to.

---

## 5. How STP Picks What to Block: Root Election, Root Ports, Designated Ports

**The concept:** STP doesn't block ports randomly. It runs a three-part election process, and once that election settles, every port in the LAN ends up with a role, and every role maps to either a forwarding or a blocking state.

**Step one — elect a root switch.** Every switch has a Bridge ID (BID): a priority value plus its own MAC address. Switches send Hello BPDUs advertising their BID as the root. Whichever switch has the numerically lowest BID wins — lower priority wins first, and if priorities tie, the lower MAC address breaks the tie. Once elected, every working port on the root switch becomes a Designated Port in the forwarding state — the root switch never blocks anything.

**Step two — each nonroot switch picks its root port.** This is the single port on that switch with the least-cost path back to the root, based on adding up the per-interface STP costs along the way. That port becomes the Root Port and goes into forwarding state.

**Step three — elect a designated port on every LAN segment.** For each individual link, whichever connected switch has the lowest total root cost becomes the designated switch for that segment, and its port becomes the Designated Port (forwarding). Every other port that isn't a root port or a designated port goes into blocking.

**Why it works:** by definition, a spanning tree has exactly one path from every switch back to the root. Root ports point "up" toward the root, designated ports point "down" away from it, and anything left over is a redundant path that gets blocked — kept alive electrically, ready to take over, but not actively forwarding frames.

**A troubleshooting habit worth building here:** if I ever see a port stuck in a blocking state and I'm not sure why, the fastest sanity check is `show spanning-tree vlan X` — it tells me the root ID, the local bridge ID, and the role/state of every interface, all in one shot.

---

## 6. STP vs. RSTP: Same Election, Much Faster Convergence

**The concept:** Original 802.1D STP works, but it's slow — the classic 50-second convergence time comes from a switch waiting through Blocking, then Listening (15 sec), then Learning (15 sec) before it starts forwarding again, on top of possibly waiting a full MaxAge timer (20 sec by default) just to notice a link failed in the first place. RSTP (802.1w) keeps the same election math — same root, same root ports, same designated ports — but adds mechanisms to skip almost all of that waiting.

**Why the wait existed at all:** STP's Listening and Learning states exist so a newly-forwarding port doesn't immediately start looping traffic before the rest of the network has caught up and MAC tables have cleared out stale entries. RSTP solves the same problem differently — by having switches directly exchange proposal/agreement messages with their neighbors and by flushing MAC tables proactively — so it doesn't need to sit and wait on a timer to be safe.

**The concrete RSTP additions:**
- **Alternate port** — a backup for the root port. If the root port fails, an alternate port can take over almost immediately, without re-running the whole election from scratch.
- **Backup port** — a backup for a designated port, relevant mainly in hub-based (shared) segments, which I'll basically never see in a modern network but which is still testable material.
- **Discarding state** — RSTP merges STP's Disabled, Blocking, and Listening states into one "discarding" state, since functionally they all mean the same thing: don't forward, don't learn.
- **Edge ports** — a port RSTP knows connects to an endpoint (not another switch) transitions to forwarding immediately, no waiting at all. This is the RSTP-native version of what Cisco's PortFast has been doing all along.

**Why it works:** RSTP replaces "wait and hope nothing changed" with "actively check and coordinate with my neighbor before changing state." That's the entire reason a topology change that used to take 50 seconds under STP typically converges in under a couple of seconds under RSTP — sometimes closer to one second in a lab.

---

## 7. Locking Down STP: PortFast, BPDU Guard, Root Guard, Loop Guard

These four features exist because STP's default behavior, while safe, isn't always the *best* behavior for every port — and a couple of them exist specifically to stop someone from accidentally (or maliciously) breaking the carefully-elected topology.

**PortFast** — tells STP to skip straight to forwarding on a port, no listening/learning delay, because the port connects to an endpoint device (a PC, a phone) that will never create a loop. The risk: if someone unplugs that PC and plugs in a switch instead, PortFast happily forwards immediately and can create a loop before STP has a chance to react.

**BPDU Guard** — the safety net for that exact PortFast risk. If a PortFast port ever receives a BPDU (a sign that something that thinks it's a switch just got connected), BPDU Guard immediately puts the port into an err-disabled state, shutting it down rather than risking a loop. In my experience this is the pairing I'll use almost everywhere PortFast shows up — PortFast alone without BPDU Guard is a half-finished config.

**BPDU Filter** — a related but different tool. It can be used two ways: paired with PortFast, where it silently stops sending/listening for BPDUs on that port after a short window (softer than BPDU Guard, which just shuts the port down); or as a standalone interface command that flatly disables STP on a port. That second usage is genuinely dangerous — Cisco's own documentation is blunt about it — because one misapplied command on a redundant link can take down the whole spanning tree.

**Root Guard** — protects the *root election*, not an individual endpoint. I enable it on ports facing switches that should never become root — usually access-layer uplinks. If that port ever receives a superior BPDU (one claiming a better, lower BID than the current root), Root Guard blocks the port in a special "root inconsistent" state rather than letting a misconfigured or rogue switch steal the root role. It recovers automatically once the superior BPDUs stop.

**Loop Guard** — protects against a more subtle failure: a unidirectional link, where the physical connection stays up on both ends but one direction of traffic silently stops working (common with a bad fiber strand). Without Loop Guard, the switch that stops receiving Hellos on that link can conclude the link is gone and start forwarding on it again, creating a loop even though the cable technically never "failed." Loop Guard keeps that port from becoming a designated port until the Hellos actually resume.

**Why it works, as a set:** each of these targets a different way a human or a rogue device could accidentally undo the topology STP worked out. PortFast + BPDU Guard covers user-facing ports. Root Guard covers "who's allowed to be the root." Loop Guard covers the failure mode where the link doesn't cleanly go down. Put together on the right ports, they turn STP from "it'll figure itself out eventually" into "it's actively defended."

---

## 8. RSTP Configuration: PVST+, Rapid PVST+, and the Bridge Priority Math

**The concept:** Cisco switches default to running RSTP, and they'll build a working spanning tree with zero configuration. But real campus designs usually want *some* input — specifically, control over which switch becomes root, since an unplanned root election can put the root switch somewhere inconvenient, like an access-layer switch instead of a distribution switch built to handle the traffic.

**Per-VLAN spanning trees.** The original 802.1D standard defines a single spanning tree for an entire LAN. That's a problem once VLANs exist, because it means all VLAN traffic is forced to load-balance identically across redundant links — there's no way to send VLAN 10 over one uplink and VLAN 20 over the other. Cisco's proprietary answer was PVST+ (Per-VLAN Spanning Tree, STP-based) and later Rapid PVST+ (the RSTP-based version, configured with `spanning-tree mode rapid-pvst`). Both build one independent spanning tree instance *per VLAN*, so I can deliberately make one switch root for VLAN 10 and a different switch root for VLAN 20, spreading traffic across both uplinks instead of leaving one sitting idle in blocking state.

**The bridge ID and the system ID extension.** To support one tree per VLAN, the original 2-byte priority field in the BID got split: 4 bits stay as the actual configurable priority (in multiples of 4096, from 0 to 61440), and the remaining 12 bits became the "system ID extension," which holds the VLAN ID. That's why `spanning-tree vlan x priority` only accepts values like 4096, 8192, 12288 — the low 12 bits always have to end in binary zero to leave room for the VLAN number.

**Two ways to set the priority:**
- Directly: `spanning-tree vlan x priority value`.
- Relatively, and much more common in practice: `spanning-tree vlan x root primary` (sets this switch's priority to 4096 less than the current root, or 24576 if that's already the lowest) and `spanning-tree vlan x root secondary` (sets it to 28672, a safe second-best). These commands calculate the number for me instead of making me do the math by hand.

**Verifying it.** `show spanning-tree vlan x` is the command I'll lean on constantly — it shows the Root ID section (about the root switch), the Bridge ID section (about the local switch), and confirms whether "This bridge is the root." The interface table at the bottom shows role (Root/Desg/Altn), state (FWD/BLK), and cost per port, which is exactly what I need to answer "why is this port blocking" without guessing.

---

## 9. EtherChannel: Making STP Ignore Redundant Links Entirely

**The concept:** STP blocking a redundant link works, but it wastes bandwidth — that backup cable sits idle until something fails. EtherChannel takes a different approach: instead of treating multiple parallel links between two switches as separate STP-visible ports (where STP would block all but one), it bundles up to eight physical links into one logical interface. STP sees a single link and doesn't need to block anything, so all the bundled links can be active and load-balancing traffic simultaneously.

**Manual configuration** — the simplest form, and the one I want to understand first before touching the dynamic protocols:
```
interface range g1/0/21-22
 channel-group 1 mode on
```
The `on` keyword forces the port straight into the channel with no negotiation at all. Both switches need matching `channel-group` commands (the number itself can differ between switches — it's a locally significant label), and IOS automatically creates a matching `Port-channel` interface once the physical ports are configured.

**Dynamic configuration — LACP and PAgP.** Rather than blindly forcing a bundle, the dynamic protocols actually negotiate and verify that both ends agree on the settings before adding a link. Cisco's proprietary PAgP uses `desirable` (actively initiates) or `auto` (passively waits) keywords; the IEEE standard LACP uses `active` (initiates) or `passive` (waits). The one hard rule to remember: never pair `on` with a negotiating keyword on the other end — `on` doesn't speak either protocol, so it will simply refuse to form the channel with an `active`/`desirable` neighbor.

**Consistency checking.** Before a switch adds a physical port to an existing channel, it compares the new port's settings against the ports already in the channel — speed, duplex, whether it's access or trunk, allowed VLAN list, and native VLAN all have to match. If they don't, the port stays configured as part of the channel on paper but doesn't actually get used, usually landing in some kind of suspended or down state instead of joining. This is exactly the kind of subtle failure I want to know how to spot fast: a mismatched native VLAN on just one of two bundled ports is enough to knock that single link out of the channel while the other keeps working, silently cutting my bandwidth in half.

**Load distribution.** Once a channel is up, the switch needs to decide, frame by frame, which physical link inside the bundle actually carries a given frame. It does this by hashing on fields like source/destination MAC, IP, or port number (configurable with `port-channel load-balance`), which guarantees that all the packets in a single flow — like one file transfer — stay on the same physical link (avoiding out-of-order delivery) while different flows get spread across different links for overall balance.

**Verification commands I'll actually use:** `show etherchannel summary` for a quick bundled/not-bundled status per port, `show interfaces portchannel 1` for bandwidth and member interfaces, and `show etherchannel port-channel` for the deepest detail, including per-port load-balancing state.

**Why the whole thing works:** EtherChannel doesn't trick STP — it genuinely removes the redundancy problem at its source by presenting what used to be multiple STP-visible links as a single logical one. STP still runs, still elects roots, still blocks true redundant *paths* if they exist elsewhere in the topology — it just never sees the bundled links as multiple paths in the first place, so there's nothing there to block.

---

## Bringing It Together: A Realistic Troubleshooting Scenario

Say I've got two access switches, SW1 and SW2, connected by two Gigabit links bundled into an EtherChannel, both trunking VLANs 10 (data) and 11 (voice), with SW1 configured as the root for both VLANs via Rapid PVST+. A user on VLAN 10 reports intermittent slowness, and phones on VLAN 11 occasionally drop calls.

My first move isn't to touch STP at all — it's `show etherchannel summary` on both switches, because a bundle silently running on one link instead of two would explain exactly this kind of intermittent, load-dependent symptom without anything looking obviously "down." If I see only one port marked `(P)` bundled instead of two, I already know where to look next: `show interfaces portchannel 1` combined with checking speed/duplex/VLAN settings on the two physical members, since a native VLAN or trunking mismatch on just one of the two links is enough to knock it out of the channel while leaving the port administratively up and easy to miss on a quick glance.

If the EtherChannel checks out clean, my next stop is `show spanning-tree vlan 10` and `show spanning-tree vlan 11` on both switches — separately, since Rapid PVST+ means these two VLANs can have completely different topologies and different root elections. I'm looking for whether SW1 is still listed as root for both VLANs, and whether the port roles look like what I'd expect. An unexpected root change usually means somebody plugged in an unmanaged switch with a lower default priority somewhere in the topology — which is exactly the scenario Root Guard exists to prevent, and if it's not already enabled on the access-facing ports, that's a gap I'd flag for follow-up regardless of what's actually causing today's symptom.

That's the instinct I'm trying to build out of this Part: don't just memorize what EtherChannel, STP, and Root Guard each do in isolation — know which `show` command answers which question, and know the order to check them in so I'm not guessing my way through a live outage.
