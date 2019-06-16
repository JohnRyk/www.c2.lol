# Spoofing Wifi with a Pineapple

Originally Published: Februrary 12th, 2015

My place of employment was gracious enough to get me a [Wifi Pineapple](https://www.wifipineapple.com/) to play around with. It's a pretty slick device that allows you to easily perform attacks on wireless networks. There's no real secret sauce to it, it's just a custom bit of hardware running a version of OpenWRT and several Linux utilities. I think it's important to not just treat this as a little hacking black box, but to understand how it works which is what I've tried to document here.

The main components that make the Wifi Pineapple so effective are Karma and PineAP. PineAP is a recent addition to the platform with the upgrade to the 2.0 firmware about 6 months ago.

### Karma <a id="karma"></a>

Karma is a set of patches for [Hostapd](https://en.wikipedia.org/wiki/Hostapd) that allows the wireless access point service to respond to probes for any SSID requested. Most wireless devices \(phones, laptops, etc\) will send out probes for networks that they've connected to in the past. For example, if you connect to "HomeWifi" at your house with your phone, as you walk around your phone sends out probes asking "Hey, is HomeWifi around?". Karma takes advantage of this by replying to these probes with "Yup! That's me." In turn your device connects to this access point thinking that it's your home network.

### PineAP <a id="pineap"></a>

Eventually, some wifi implementations started to be a little more thorough before associating with a WAP. These implementations now wait for an AP to broadcast a list of it's SSIDs through a beacon before connecting. PineAP is a suite of tools to get around this as well as allow the platform to be used for targeted attacks. These tools include:

* **Beacon Response** - PineAP can create beacons and either broadcast them or target them to a specific client.
* **Harvester** - Listens for probe requests for SSIDs and logs them.
* **Dogma** - Takes a list of SSIDs from Harvester and generates beacons for either a target or broadcast.

### Things I've learned <a id="things-i-ve-learned"></a>

I still have a lot of things to learn about the tech behind this box. I haven't even touched on [aircrack](http://www.aircrack-ng.org/) \(and aireplay\) yet. I'd like to play around with deauth and WPS attacks. Also I believe what I've written here is accurate but I could still learn more about how the various tools work together. In working with box, I did learn that if you want to spoof a SSID on one radio while you're connected to that SSID on the other, it's best to connect to the SSID first then start PineAP and Karma.

From a personal standpoint, while I didn't really trust open wifi points before playing around with this, it really opened my eyes to just how easy it is to take over an access point. I feel better about my existing paranoia to use a VPN or SSH tunnel whenever I'm on a network that I don't own.

