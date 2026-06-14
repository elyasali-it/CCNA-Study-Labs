# CCNA Study Labs

> Interactive training tools built chapter by chapter for the Cisco CCNA certification exam.

**Author:** Elyasali — CompTIA A+ | Network+ | Security+ | CCNA in Progress

---

## Why I Built This

Most people study for certifications by watching videos and reading notes. I learn by **doing** and **repeating**.

So I built my own training tools. These are real HTML applications I wrote from scratch to drill Cisco CLI commands and networking concepts until they became automatic — the same way a musician practices scales, not just reads sheet music.

I already hold CompTIA A+, Network+, and Security+. The CCNA is the next step — and this repo is where I do the work.

---

## What's Inside

### File 1 — `ccna-ch4-ch5-study-lab.html`

A complete study application with 5 tools in one:

- **Quiz** — 20 questions covering CLI modes, memory types, frame forwarding, MAC tables, and access methods. Tracks score and explains every answer in plain English.
- **CLI Simulator** — A working Cisco IOS simulator in the browser. Type real commands and get real output.
- **Flashcards** — 24 cards covering every key concept from Chapters 4 and 5. Tap to flip. Score tracked live.
- **Reference Sheet** — Memory types table, frame forwarding visual, MAC address table, STP diagram, CLI mode ladder, and access methods comparison.
- **Lab Tracker** — Checkbox list of all 11 Chapter 4 labs, Chapter 5 switching concepts, and the Branch Switch Project.

### File 2 — `ccna-drill-trainer.html`

A muscle memory drill trainer. No clicking, no multiple choice — you type every command yourself, every time.

| Drill | Topic | Reps per Command |
|---|---|---|
| 1 | Enter & Exit Modes | 3 |
| 2 | Set a Hostname | 3–4 |
| 3 | Save Configuration | 5 |
| 4 | Port Descriptions — 3 interfaces | 4 |
| 5 | Full Workflow — cold switch to saved config | 3–5 |
| 6 | MAC Table Commands | 4–5 |
| 7 | Frame Forwarding Recall | 4–5 |
| 8 | Branch Switch Project — full real job scenario | 2–5 |

---

## Concepts Covered

### Chapter 4 — CLI Fundamentals

| Concept | Details |
|---|---|
| CLI Mode Navigation | User EXEC → Privileged EXEC → Global Config → Interface Config |
| Configuration Save | RAM (running-config) vs NVRAM (startup-config) |
| Port Documentation | Interface descriptions across multiple ports |
| Memory Types | RAM · NVRAM · Flash · ROM |
| Show Commands | running-config · startup-config · version · mac address-table |
| Access Methods | Console · Telnet (insecure) · SSH (encrypted) |

### Chapter 5 — Ethernet LAN Switching

| Concept | Details |
|---|---|
| Switch Operation | Layer 2, MAC addresses only |
| Frame Forwarding | Known Unicast = Forward · Unknown Unicast = Flood · Broadcast = Flood |
| MAC Learning | Learn from SOURCE MAC · Forward using DESTINATION MAC |
| MAC Aging | Default 300 seconds |
| Spanning Tree Protocol | Prevents Layer 2 loops and broadcast storms |

---

## Real-World Connection

- **Port descriptions** — Every rack installation requires documenting what is plugged into each port
- **Save configuration** — Skipping copy running-config startup-config means config is lost on reboot
- **MAC address table** — First command when a user cannot reach the server
- **STP** — Understanding why a network crashes after someone adds a second cable between two switches

---

## How to Open These Files

1. Download either `.html` file
2. Open in Chrome, Firefox, or Edge
3. No internet required — no install, no login

---

## What's Coming Next

- [ ] Chapter 6 — VLANs and Trunking
- [ ] Chapter 7 — IP Addressing and Subnetting
- [ ] Chapter 8 — Static Routing
- [ ] PowerShell automation scripts

---

*Built by Elyasali · Columbus, OH · Network Engineer | CompTIA A+ | Network+ | Security+ | CCNA in Progress*
