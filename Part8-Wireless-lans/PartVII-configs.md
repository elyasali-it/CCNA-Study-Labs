# Part VIII: WLAN Configuration Reference

This is a quick reference from a wireless network I actually built out during my studies, so I have real settings written down instead of just concepts. I'm including it to show that I don't just understand wireless networking in theory, I can sit down and configure one correctly.

## The network I built

I set up a wireless network for an "Engineering" department as a hands-on exercise, and here's what that actually involved.

I created the internal connection that ties the wireless network to the right part of the company network:

| What I set | What I entered |
|---|---|
| Name I gave it | Engineering |
| Internal network number (VLAN) | 100 |
| Its address on the network | 192.168.100.10 |
| Subnet mask | 255.255.255.0 |
| Gateway | 192.168.100.1 |
| Primary address server | 192.168.1.17 |
| Backup address server | 192.168.1.18 |

Then I created the actual wireless network people would see and connect to:

| What I set | What I entered |
|---|---|
| Network name (SSID) | engineering |
| Status | Turned on |
| Visible to devices searching for Wi-Fi | Yes |

And I locked it down with modern security instead of anything outdated:

| What I set | What I entered |
|---|---|
| Security type | WPA2 |
| Encryption | AES |
| Login method | Shared password |
| Older, weaker options (WPA, TKIP) | Turned off |

## Rules I have to follow when setting one of these up

I also know the limits the system enforces, which matters because it's easy to make a mistake if I don't know them going in:

| Field | How long it can be | What it accepts |
|---|---|---|
| Network profile name | 1 to 32 characters | Letters and numbers |
| Network name (SSID) | 1 to 32 characters | Letters, numbers, spaces, some symbols |
| VLAN number | 2 to 4094 | A number |
| Network ID | 1 to 512 | A number |
| Shared password | 8 to 63 characters, or exactly 64 if entered a different way | Text or a long code |

## Security options I understand, from weakest to strongest

Part of knowing wireless security is knowing what not to use. Here's how I'd rank the options I came across, and what each one actually is:

| Option | What it really is | Would I use it |
|---|---|---|
| No security | Open network, anyone can join | No, only for a public hotspot with no sensitive data |
| WEP | An old, broken security method | No, never |
| LEAP | An early Cisco fix that's also outdated now | No |
| EAP-FAST, PEAP | Middle-ground options using a shared secret or a server certificate | Only if the business already has that setup in place |
| EAP-TLS | The strongest option, but every device needs its own certificate installed | Yes, for high-security environments willing to manage certificates |
| WPA2 with a shared password | The standard, solid choice for most businesses | Yes, this is my default recommendation |
| WPA3 | The newest and most secure option available | Yes, whenever the equipment supports it |

## How I'd prioritize network traffic

I also learned how to tell a wireless network to treat different types of traffic differently, so a phone call doesn't lag just because someone's downloading a large file at the same time:

| Priority level | What it's for |
|---|---|
| Highest | Voice calls |
| High | Video |
| Normal | Regular web and file traffic (the default) |
| Lowest | Background updates and syncing |

## Limits I'd keep in mind

By default, the system doesn't cap how many devices can join a wireless network, but it does cap how many devices one single access point can serve at once, at 200 per radio. If a business needed tighter limits than that, for security or performance reasons, I know how to set those manually instead of relying on the defaults.
