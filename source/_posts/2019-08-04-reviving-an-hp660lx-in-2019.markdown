---
layout: post
title: "Reviving an HP 660LX in 2019"
date: 2019-08-04 09:52
comments: true
categories: 
---

It started off as a joke...

<blockquote class="twitter-tweet" data-dnt="true" data-link-color="#E81C4F"><p lang="en" dir="ltr">Just setting up my burner machine for <a href="https://twitter.com/defcon?ref_src=twsrc%5Etfw">@defcon</a> <a href="https://t.co/Dz9pjCBTCo">pic.twitter.com/Dz9pjCBTCo</a></p>&mdash; Justin Collins (@presidentbeef) <a href="https://twitter.com/presidentbeef/status/1144761345847336960?ref_src=twsrc%5Etfw">June 29, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

I had spent some time _several years ago_ trying to get Linux running on this machine via the (defunct) JLime project,
so I had some of the pieces available to actually get this little "pocket computer" going again - mainly
compatible CompactFlash cards and an external card reader.
But I was mostly joking.

Then I starting thinking how funny it would be to actually sit in a talk and take notes at [DEF CON](https://defcon.org/) on it...

### Battery Power

The reason I was mostly joking is because the batteries in the 660LX were not working at all.
So, what was I going to do? Plug it into the wall? That's just sad, not funny.

I started looking around online for replacement batteries for this machine from 1998.
Despite visiting several rather shady websites, for some reason I was unable to find anyone selling twenty-year-old
laptop batteries.
Some sites claimed to offer a replacement, but from the pictures it was clear they would not work.
The only real possibilities I found were complete sets - a 660LX, manual, cables, etc.
I already had a 660LX, acquired for free, so I really didn't want to spend $100+ on another one!
Also, kind of ruins the joke to do so.

*(Side note: the 660LX has a button battery for backup power. Searching for "HP 660LX battery" will return sites trying to sell
you a little CR2032 battery.)*

Now, I will admit my mental model of a laptop battery was a block of chemical goop inside of some plastic wrap with some wires coming out of it.
After ungracefully disassembling the 660LX battery, I found inside it was just two smaller batteries?!

[![Open battery pack of HP 660LX](/images/blog/hp_660lx/hp660lx_battery_open.jpg)](/images/blog/hp_660lx/hp660lx_battery_open.jpg)

The batteries said "US18650S SONY ENERGYTEC" on them.

[![Sony 18650S Batteries](/images/blog/hp_660lx/hp_660lx_old_batteries.jpg)](/images/blog/hp_660lx/hp_660lx_old_batteries.jpg)

While I didn't find those exact batteries, some investigation showed the 18650 battery in general is extremely common.

There are two kinds of 18650 - one with "caps" (that little nub on the top) and one without.
It seems the ones with caps are safer, as they have an internal circuit to keep them from blowing up.
However, as you can see above, I needed the kind without caps. Presumably the little circuit board
regulates charging the batteries.

The Internet suggested sticking to "brand name" batteries, but weirdly Amazon does not carry any of those.
I took a chance on a pack of batteries which reviewers suggested looked like "genuine" Samsung batteries.

[![Samsung 18650 batteries](/images/blog/hp_660lx/samsung_18650-30Q_batteries.jpg)](/images/blog/hp_660lx/samsung_18650-30Q_batteries.jpg)

I carefully ripped the old batteries out. The leads from the batteries to the little circuit board were
actually soldered to the batteries, so I pried them off with a screwdriver. Probably not a great idea
unless they are really, truly dead.

With some effort, I shoved the new batteries back in the case and sandwiched the wires back in as well.
I didn't bother actually attaching/gluing/soldering anything.

I did, however, scare myself when I generated a terrifying little electric arc as I 
tried to use a screwdriver to squeeze everything back into the battery case.

[![HP 660LX batteries replaced](/images/blog/hp_660lx/hp660lx_batteries_replaced.jpg)](/images/blog/hp_660lx/hp660lx_batteries_replaced.jpg)

I may have damaged the case just a tiny bit when I gently pried it open,
contributing to it looking a slightly sketchy when I tried to close it back up.

[![HP 660LX battery put back together](/images/blog/hp_660lx/hp660lx_battery_together.jpg)](/images/blog/hp_660lx/hp660lx_battery_together.jpg)

But, who cares how it looks...does it work??

[![HP 660LX running on battery power](/images/blog/hp_660lx/hp660lx_running_on_new_batteries.jpg)](/images/blog/hp_660lx/hp660lx_running_on_new_batteries.jpg)

**YES!**

Hahahaha now I can walk around using my relic with no wires!

### Software Updates

Nowadays, a device that can't connect to anything does not seem like much fun.

Of course I wanted to hook this Windows CE 2.0 machine up to the Internet!

The HP 660LX has a "PC CARD" expansion slot where you can slap in an Ethernet or wireless card,
but of course it is ancient and you have to be careful about compatibility.

Enter [HPC: Factor](https://www.hpcfactor.com/)! This is a website/forum full of useful information.
For £10 you can get access to a ton of file downloads (software, drivers, updates) for a year.
Totally worth it.

One thing I learned quickly is that you need the service pack for Windows CE 2.0 and the Network Service Pack 
in order to have a chance at getting a wireless card to work.

At this point, I had been transferring files to the 660LX via a compact flash card (which, at 8GB, probably blew the little machine's _mind_).
However, most software for Windows CE requires installation via ActiveSync.

[![Kingston CompactFlash card](/images/blog/hp_660lx/kingston_cf_8gb_card.jpg)](/images/blog/hp_660lx/kingston_cf_8gb_card.jpg)

What is ActiveSync? Well, originally these "pocket computers" weren't meant to be tiny laptops.
They were more like little helpers you use while you are away from your main machine, then you sync up
files, calendars, email, etc. when you went back to your desk.

ActiveSync was the software used to sync between a pocket computer and your main machine. 

Now, for Windows CE 2.0, the recommended version of ActiveSync is 3.8. The very _newest_ operating system
supported by ActiveSync 3.8 is Windows XP.

By pure luck, I had an old Windows XP laptop and I was able to install ActiveSync!

**BUT**... you need a special serial cable to hook up the 660LX.
At first I poked around eBay, but no luck. Yet, in the back of my mind,
I was _pretty_ sure I still had that cable somewhere. I searched all around my office and
dug through my big box of (mostly useless) cables,
but still no luck.

Just when I gave up (of course) I found it!! Yay!!

**BUT**... turns out I don't have a serial port on my Windows XP laptop.

I thought about trying a Windows XP virtual machine on my main Linux box, but it doesn't have a serial port, either!
None of my machines had an infrared port, either.

After first buying the wrong cable on Amazon, I got an RS-232 to USB adapter and a tiny, tiny CD with drivers.
Luckily, the laptop has a CD drive, so I was able to actually install the proper drivers.

[![USB to serial cable](/images/blog/hp_660lx/usb_to_serial_cable.jpg)](/images/blog/hp_660lx/usb_to_serial_cable.jpg)

After an uncomfortable amount of configuration twiddling... they connected!!

[![ActiveSync is connected](/images/blog/hp_660lx/active_sync_connected.jpg)](/images/blog/hp_660lx/active_sync_connected.jpg)

I was then able to install Windows CE 2.0 SP1 and the CE Network Service Pack.

[![Installing Windows CE SP1](/images/blog/hp_660lx/install_ce_sp1.jpg)](/images/blog/hp_660lx/install_ce_sp1.jpg)

### Networking

One of my criteria for this project was to _not spend much money on a joke_.

After spending some time looking around, I bought a $20 wireless adapter off of eBay.
$20 was really right at the limit of my per-item budget.

In the meantime, though, I found out there was another way to access the Internet.

Turns out you can "share" the networking connection on the main machine with the 660LX
over the serial cable, via ActiveSync.

The only weird bit is that you need to run a proxy server on the main machine to route
the connection to the Internet.

In the modern world, that is not a problem. In the land of Windows XP, however, I was not sure I would be able to get something working.
I found [CCProxy](https://www.youngzsoft.net/ccproxy/windows-proxy-server.htm), which did work, despite its awful and confusing interface.

Configured the proxy for "The Internet" on the 660LX and...

[![Accessing Google via Pocket Explorer](/images/blog/hp_660lx/hp660lx_accessing_google.jpg)](/images/blog/hp_660lx/hp660lx_accessing_google.jpg)

Wow! The Internet!

Sadly... or not so sadly... the world has moved to HTTPS and to stronger protocols than what
lowly Pocket Explorer supports. Thus, most of the web is entirely inaccessible on the device.

[![Failure to access sites over HTTPS](/images/blog/hp_660lx/hp660lx_failure_to_access_github.jpg)](/images/blog/hp_660lx/hp660lx_failure_to_access_github.jpg)

As a result, when the eBay seller canceled my order for the wireless adapter, I figured "meh".
Even if you can get WiFi working (which would likely require connecting to a totally unsecured network),
there's not much of the web that one can even visit.

Yes, it would be possible to use an SSL stripper, etc., but I didn't want to go through the hassle of setting
that up on Windows XP.

### Wrapping Up 

[![The whole setup](/images/blog/hp_660lx/the_whole_setup.jpg)](/images/blog/hp_660lx/the_whole_setup.jpg)

This turned into more of a narrative than a how-to guide.
Maybe I'll do another write-up with the details.
In the meantime, I can try to answer questions about specifics.
