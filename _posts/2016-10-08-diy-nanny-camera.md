---
title: DIY Nanny Camera
updated: 2016-10-08 09:00
---

My wife and I had a need for a couple of nanny cameras/video baby monitors around the house.  She wanted one so she could check in on the kid she nanny's during the day while he napped, and I wanted one so I check in on our three dogs while I was away from the house.  We both wanted something that wouldn't break the bank either. 

First I looked at getting a purpose built device like this [one](https://www.amazon.com/Motorola-MBP36S-Wireless-Monitor-3-5-Inch/dp/B00M2F0OYS?th=1) but didn't like how much it costs (I'd need to buy two) and how it wasn't an open platform (the device only works locally, so there'd be no way to access the video feed on a tablet, smartphone, or remotely). I could have gone with a [Nest Dropcam](https://nest.com/camera/meet-nest-cam/), but again the price is prohibitive (2x factor) and it relies on a third-party service to view the camera feed.  Nest products relay their data streams via their infrastructure... so if Nest folds or discontinues a product ([which has happened in the past](http://fortune.com/2016/04/06/nest-meager-response-google-revolv/)), your access to the data coming from that product is removed and you're left with a paperweight.

A friend of ours suggested that I pick up a pair of cheap [IP cameras](https://www.amazon.com/Foscam-FI8910W-Network-Camera-Two-Way/dp/B006ZP8UOW) and use an [iPhone App](https://itunes.apple.com/US/app/id511651356?mt=8) to connect to them. The cost and open nature of the interface had me sold... but there was a catch.

This sort of device is [notoriously](http://www.computerworld.com/article/2878741/hacker-hijacks-wireless-foscam-baby-monitor-talks-and-freaks-out-nanny.html) [insercure](https://krebsonsecurity.com/2016/10/who-makes-the-iot-things-under-attack/) in it's default state.  I needed to get it secured before I could trust in on my local network.

The immediate steps to do all of that were:

1. Update the device to the [latest firmware](http://foscam.us/firmware) insuring that I at least get all of the security updates that are known at that time.

2. Set a non-default username and password on the device:
...![screenshot](/assets/diy-baby-monitor-user-pass.png)

3. Set the device up to use a non-default port:
...![screenshot](/assets/diy-baby-monitor-network-port.png)

Once I had secured the devices themselves, I moved on to setting up a secure way to access the devices remotely.  My home router runs [OpenWRT](https://openwrt.org/), so I set out to do all of this via the CLI editing the configuration files as needed:

1. First, I setup a static DHCP assignment for each of the devices:

   ```
   config host                   
        option mac '48:02:2A:49:46:B9'
        option ip '172.16.0.26'
        option name 'camera-1'
   
   config host               
        option mac '48:02:2A:43:6C:1A'
        option ip '172.16.0.27' 
        option name 'camera-2'
   ```

2. Next, I setup port forwarding for each of the devices to the non-standard port I set earlier:

   ```
   config redirect                   
        option target 'DNAT'
        option src 'wan'
        option dest 'lan'   
        option proto 'tcp udp'
        option src_dport 'REDACTED' 
        option dest_ip '172.16.0.26'
        option dest_port 'REDACTED'   
        option name 'Camera 1'

   config redirect           
        option target 'DNAT'        
        option src 'wan'     
        option dest 'lan'   
        option proto 'tcp udp'
        option src_dport 'REDACTED' 
        option dest_ip '172.16.0.27'
        option dest_port 'REDACTED'
        option name 'Camera 2' 
   ```

3. Finally, I setup a subdomain hostname override in the local DNS settings. This allowed the monitoring app, when connected to my WiFi, to connect to the cameras via the LAN, and not via the internet (saving a ton of data usage on my cell phone plan):

   _(Note: I've used a generic domain name here in my example.)_

   ```
   config dnsmasq
        option some-private-domain.com
   ```

   ```
   config domain                  
        option name 'camera-1'
        option ip '172.16.0.26'
                          
   config domain                         
        option name 'camera-2'  
        option ip '172.16.0.27'
   ```

I setup the same domain with AWS Route53 setting a wildcard A record using public IP of my router so that the same subdomains would resolve properly on the public internet as well.  This allowed me to access the cameras remotely.

After all of that I was then able to add each of the IP cameras to the monitoring app, entering the FQDN instead of an IP address, along with the username and password I set.

What I ended up with was a baby monitor that used off the shelf components (allowing me to upgrade, replace, or configure each component seperately), was accessible remotely and locally, and didn't rely on any third party services... and cost me me 50% less than any of the other options.
