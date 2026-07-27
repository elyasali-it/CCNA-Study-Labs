# Part VIII: Wireless LANs

This is what I learned working through the wireless networking material in my CCNA studies. I'm writing it in plain language because I want anyone reading this, technical or not, to understand what I actually know how to do.

## How wireless networks work

When I plug a computer into a wall jack with a cable, that cable is the whole network. There's no guessing about who's allowed to talk or when. Wireless doesn't work that way, because the signal is just floating in the air where any device nearby can pick it up. So the first thing I had to understand is how a wireless network keeps itself organized without a cable holding everything in place.

Every wireless network starts with an access point, which is the device that creates a coverage area I can connect to. I know that access point as an SSID, which is just the network name I see when I search for Wi-Fi on my phone or laptop. Behind the scenes, that access point is also broadcasting its identity constantly, about ten times a second, so devices nearby know it's there and can connect to it.

Before I can actually use a wireless network, my device and the access point go through a short back-and-forth where my device basically says "let me join" and the access point says "okay, you're in." Once I'm connected, I don't talk directly to other devices around me even if they're in the same room and on the same network. Everything I send goes through the access point first. That's by design, so the network stays organized and secure.

I also learned that one access point isn't stuck standing alone. It has a wired connection back into the rest of the network, and that connection can carry more than one wireless network at the same time. So a business can have a "Staff" Wi-Fi and a "Guest" Wi-Fi running off the same access point, and I know how to keep those two separated properly so guest traffic never touches the internal company network.

For bigger spaces, one access point isn't enough, so I learned how multiple access points work together to cover a whole building using the same network name everywhere. As I walk around with my phone, it automatically switches from one access point to the next without me noticing, and I understand how that switch actually happens behind the scenes.

I also know a few special setups beyond the basic "access point and clients" model:

- Extending a wireless signal further using a device that repeats it, and I understand the tradeoff that comes with doing that.
- Giving a wired-only device (like older medical equipment) a way onto a wireless network.
- Connecting two buildings together wirelessly instead of running cable between them, which is useful when a business has multiple locations close by.
- Linking several access points together in a chain so I can cover a large outdoor or hard-to-wire area without running a cable to every single one.

Lastly, I learned how Wi-Fi actually splits up the airwaves into channels, and why picking the right channels matters so nearby networks don't interfere with each other. I also know the different generations of Wi-Fi (what most people just call Wi-Fi 5, Wi-Fi 6, and so on) and what each one improved.

## How Cisco manages wireless networks at scale

Once I understood how one access point works, the next question I had to answer was: how does a business manage dozens or hundreds of them at once? I learned there are a few different ways Cisco handles this, and I know when each one makes sense.

The oldest way treats every access point as its own standalone device that I'd log into and configure one at a time. I understand why that gets painful fast once a business has more than a handful of them, because any change has to be made everywhere individually.

A simpler version of that same idea moves the management into the cloud instead. I know how Cisco Meraki works this way: every access point checks in with a cloud dashboard on its own, so I can push settings and see what's happening across the whole network from one screen instead of touching each device by hand.

The setup I focused on most is how larger businesses actually run wireless today, where the access points themselves do less work and a central device called a wireless LAN controller does the heavy lifting. The access point just handles the radio signal, and the controller handles everything else: who's allowed to connect, how devices roam between access points, and all the security settings. I understand how the access point and controller stay in constant contact with each other over a secure tunnel, and why that setup makes it possible to manage a huge number of access points from one place.

I also learned that a wireless controller can live in different spots depending on how big the company is. Some businesses run one central controller for their whole campus. Others put the controller in the cloud. Smaller branch offices sometimes don't need a dedicated controller box at all, because a single access point can handle that job locally. I know how to match the setup to the size of the business.

One thing I think is genuinely useful knowledge: I understand what happens to a branch office's Wi-Fi if the connection back to company headquarters goes down. I know how to set things up so the local Wi-Fi keeps working for people in the building even during an outage, instead of the whole wireless network going dark just because the connection to the main office dropped.

## How I secure a wireless network

Because a wireless signal travels through the air, anyone within range can potentially listen in, so I spent real time learning how to lock a wireless network down properly. I think about wireless security as answering four questions: is this device who it says it is, is this access point the real one and not a fake, is the data private while it's traveling, and can I tell if someone tampered with it.

I know that older security methods like WEP are outdated and shouldn't be used anymore, because they can be broken. I understand why, and I know what replaced it.

I'm comfortable with how modern authentication works, where a device proves who it is against a central server before it's allowed onto the network, rather than everyone just typing the same password. I know several different ways this gets done, and I understand the tradeoffs between them, like how some methods need a certificate installed on every single device while others don't.

For actually protecting the data itself once someone's connected, I know the difference between the older, weaker encryption methods and the current standard, which pairs strong encryption with a way to verify nothing was changed in transit. I understand why a network is considered secure or not based on which of these methods it's using.

Finally, I know the three security certifications that most people have heard of: WPA, WPA2, and WPA3. I understand that each one comes in a "personal" version, which uses one shared password for everyone, and an "enterprise" version, which authenticates each person individually against a server. My rule of thumb, and something I'd apply on any network I set up, is to always use the newest version everything in the building can support, and to never fall back to the outdated stuff just because it's easier.

## How I actually build a wireless network

This is where everything above turns into real work I can do. I know how to physically connect an access point to a network switch, and I understand the difference in how that connection needs to be configured depending on whether the access point is working on its own or reporting to a central controller.

I'm comfortable logging into a wireless controller through a web browser to configure it, and I've worked with both of the two Cisco platforms currently in use, which look different on screen but work on the same underlying logic. I know my way around either one.

I understand what a controller's various network ports and connections are for, including how businesses combine multiple connections into one for reliability, so that if one connection fails, the network keeps running.

I know how to actually build a wireless network from scratch on a controller: setting the network name, deciding which part of the internal network it connects to, and choosing the right security settings. I've walked through this process on both Cisco platforms, and I know how to verify the network I built is actually working correctly once it's done, rather than just assuming it is.

I also know a practical limit that matters in the real world: a business shouldn't just keep adding wireless networks endlessly on the same equipment. Every additional wireless network adds overhead that slows things down for everyone, so I know to keep the number of networks on a single access point reasonably small and only add what's actually needed.

## How I'd troubleshoot a wireless problem

Here's how I'd think through a real situation: say I get a call that a branch office's Wi-Fi keeps dropping people, and a couple of guest devices can't get online at all.

First, I'd figure out whether this location's access point depends on a controller somewhere else, and whether the connection to that controller is even up. If that connection to headquarters is down, I'd know that explains devices getting kicked off, because the access point relies on that central controller for a lot of what keeps a connection stable.

Next, I'd check whether the guest network is actually pointed at the right part of the internal network. A guest device that can see the Wi-Fi name but can't get online is usually a sign that it's not properly connected through to wherever it's supposed to get an internet address from.

If those check out, I'd look at the security settings next, since a device that connects but then immediately drops is often a sign of a password mismatch or a security setting that doesn't match what the devices are expecting. I'd confirm the settings on the controller match what people are actually typing in, and I'd check whether the access point itself is even in the right mode to be serving clients in the first place, since it's possible for one to accidentally be set to a mode where it isn't actively serving anyone at all.
