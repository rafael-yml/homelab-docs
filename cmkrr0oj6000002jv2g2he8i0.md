---
title: "Making a Beefy Private Cloud - Infrastructure & Networking"
datePublished: Sat Jan 24 2026 03:27:05 GMT+0000 (Coordinated Universal Time)
cuid: cmkrr0oj6000002jv2g2he8i0
slug: making-a-beefy-private-cloud-infrastructure-and-networking
tags: linux, server, debian, freebsd, vpn, homelab, proxmox, opnsense, tailscale

---

# Why?

For many years now, dare I say 'since I was a little kid', I've been wanting to make my own server, with my own things, my own services, and with all the freedom that entails. Growing up in a South America I wasn't really able to afford or ask for computer parts, my family just didn’t have those kinds of priorities. What they did eventually give me was a PS2 with some games, and a simple Celeron computer.

This was pretty much all I needed to start going crazy with technology, apparently my PS2 came 'modded' to run any game ‘burned‘ in any DVD-ROM, I didn’t really know what this meant at the time and only got to understand it a few years later, but it allowed my mom to buy a number of games from different sellers for pennies. I am sure my mom was, and probably still is, not aware of what modchips are (and I am interested to know if this is just a common occurrence in South America at large). Needless to say, I started using our home dial-up internet connection to research a lot about my favorite games, tricks, secrets, and then I stumbled upon the world of 'Action-Replay', and 'Code Breaker' (homework for you to look it up). I learned how to burn DVDs, I burned my own copy of Code Breaker, and started understanding what 'Memory' and 'Values' were, eventually making my own codes which required me to learn Hexadecimal.

This was critical for me becoming who I am today. Since way back then I wanted my own server, I fantasized with hosting my own MMOs to give my friends infinite money and destroy all those dungeons that we struggled with after grinding for hours. But this was just not meant to be for the time and the place. Life got in the way. I grew up, the economy completely collapsed in my country, my mother lost her job, I dropped out of uni, eventually got a scholarship to fly and study on the other side of the world and had to start from scratch, studying and working constantly, until finally graduating from uni, getting a 'proper' job and marrying my wife. There was basically no space (or money) for a server during all this time.

But late last year, when I had some extra money laying around, I saw the news about RAM prices going up, and the night that Micron decided to axe their consumer brand Crucial, I knew that I had to purchase something for self-hosting or it could be too costly later on. The night I knew about Crucial, I hopped online and bought a couple of mini-PCs with 16GB RAM each (DDR4), one with an older Skylake i5, and one with a more modern Ryzen with an i-GPU. I already had plans to start getting pieces to run a server throughout 2026 but I had to get it done asap. The mini-PCs arrived and I basically procrastinated for a couple of months because I 'didn’t have a pfSense router yet'. I was waiting for my next salary to buy a cheap box to route my traffic using pfSense, but once my salary came, the same box was going for twice the price already. It was a moment of despair for sure and just made me procrastinate even more.

My Ryzen box has 2 RJ45 NICs so I thought that there could be a possibility to virtualize the router OS and pass through my NICs to the OS and have it manage my LAN. This was basically what encouraged me to try, I also had an old Archer C20 v4 laying around with a super outdated OpenWRT firmware. So I started researching. Originally I was planning on having Arch Linux as the base system and have different VMs or Docker containers running my services, but then I saw Proxmox popping up in many of my searches so I decided to go for that. Okay enough blabbering, time to get into the Hardware, Infrastructure and Networking that made my final server possible! I am unsure if you can follow my article and followups as guides to set up your own, but I'm sure it can at least serve as inspiration. It is very possible for almost anyone!

# What?

## Hardware

### Mini-PC:

* AMD Ryzen 5 7430U (6 Cores / 12 Threads + i-GPU).
    
* 16GB DDR4 (SODIMM).
    
* 512GB SSD (M.2).
    

### Networking:

* The two NICs (Realtek RTL8125 1Gbps) included with the mini-PC.
    
* My rental’s Huawei Fiber Modem as the gateway to the internet (ISP).
    
* An old Archer C20 v4 I had laying around as my Wi-Fi Access Point (AP).
    

## Infrastructure

* Fiber Modem wired to the first RJ45 port in the mini-PC (WAN).
    
* Proxmox installation on the Ryzen mini-PC to manage all the LXCs and VMs.
    
* Virtual Network Layer inside Proxmox with Linux Bridges (vmbr0 - WAN & vmbr1 - LAN).
    
* Virtualized OPNsense that acts as the gateway between the two Linux Bridges and routes traffic.
    
* Archer C20 v4 wired to the second RJ45 port in the mini-PC (LAN), all other devices connect here.
    
* Tailscale as a global bridge, makes my server and services accessible from anywhere in the world.
    
* (Optional) Server as exit node in Tailscale to route traffic from anywhere else.
    

# How?

## Cero - The Wall

Before anything, I have a big problem, I am renting. And sharing the house/internet with other tenants. It is a really good deal and everything is included in the monthly rent, the one caveat is that I don’t have access to the ISP modem connected to the wall (via Fiber). This is not really an issue for setting up a LAN and having a server and all that, the only issue is accessing my server from elsewhere in the world, which is kind of the whole point of this. Other than not having a dedicated x86 router, this was one of the biggest hurdles. No port forwarding, no NAT configuration, no options. Spoiler: Tailscale, I will get into that later. Bur for now just know that my mini-PC server is directly connected to the modem with an Ethernet cable.

## Uno - The Hypervisor

So why Proxmox? My original intention was to run Arch or maybe a Debian OS on my main computer and use docker for the rest. I had seen some Proxmox content and recommendations before but it really was not in my original plans because I originally had planned to have a dedicated x86 router box running pfSense attached to the internet, but when that plan went out the window I looked into what could work with what I had. Now, my Ryzen mini-PC was a pretty good deal, and usually that is for a reason, nothing is free in this life and in the listing they didn’t specify what type of NIC the mini-PC had, so I knew they were most likely Realtek, which if you have spent any time reading pfSense and OPNsense posts online with setups and configurations, you would know that Realtek is not very well supported without going thru hoops and usually Intel NICs are recommended. The solution as you might expect is Proxmox, OPNsense is based on FreeBSD so driver support is not as extensive as you would with Linux, Proxmox is Linux-based so it has no problem dealing with the Realtek NICs and by using Linux Bridges, Proxmox can pass clean, virtual NICs to OPNsense which bypasses all the problems mentioned before. Yay!

I took a monitor outside, connected the mini-PC to it, plugged the modem to the first NIC, and installed Proxmox, I didn’t plug my AP to the second NIC yet, so that I could know which port was to become the WAN port through the Proxmox installation (in my case it was enp2s0). This NIC would have no IP address and as such we don’t need to configure it during installation. The other NIC (in my case it was enp1s0) will be our Management Interface (even though there is no cable connected to it yet) and become our LAN. So we have to set it up. I configured it thusly:

* Hostname: pve.home
    
* IP Address: 192.168.1.5 (Make sure there is no conflict with this IP range with your modem’s LAN).
    
* Netmask: 255.255.255.0
    
* Gateway: 192.168.1.1 (Make sure this is no conflict with this IP range with your modem’s LAN).
    
* DNS Server: 1.1.1.1 (You can choose from any of the major vendors like Cloudflare, Google, Quad9, etc. as long as we can resolve addresses for the time being).
    

Then, we let it install and once it's done we'll be greeted with a text guiding us to a web browser to configure the server by connecting to the IP we set up (192.168.1.5 in my case). Well then, how do we connect to the server if there is no cable connected to the other port? Time to plug a cable. Of course I plugged my Archer C20 v4 because my other computer is in another room and I don't have a cable long enough to reach into my computer so I guess it’s time for-

## Dos - The Listener

It’s Archer time. My old Archer C20 v4 is a TP-Link unit, a few years back I flashed it with OpenWRT to get access to extra plugins and features but ended up not using for a while and it was collecting dust, the firmware was quite outdated and when I plugged it into my newly set up server… Nothing much happened. Honestly I can’t remember how I had set it up last time, so I decided to flash a new firmware and go for a factory reset. To do this I had to move my Archer to the room with my computer to plug it directly and upload the new firmware and to set it up. I enabled the 2.4Ghz and 5Ghz radios to connect my devices. Then in Network &gt; Interfaces, edited my LAN interface as such:

* IPv4 address: changed to 192.168.1.2 (we are planning our OPNsense to be our gateway at .1).
    
* IPv4 gateway: 192.168.1.1
    
* IPv4 broadcast: 192.168.1.255
    
* DNS server: 192.168.1.1
    

Then we have to disable DHCP by going to the DHCP tab and checking the 'Ignore interface' checkbox. I also disabled IPv6 and configured my Firewall to accept input, output and forward. Save & Apply to let the changes take effect. Then I took the Archer back outside to connect one of its LAN ports it directly to my mini-PC on our previously configured LAN port. The Archer is now effectively a dumb AP that only passes traffic from the LAN cable to the wireless radios and doesn’t do any routing or filtering. Keep in mind that to access our server now, since we do not have any 'Router' anymore (we will be creating it next), we do not have any DHCP to tell our connected devices which IP address they should use, so for the time being, we have to manually set our IP address to something like 192.168.1.10 so our devices can talk to our server. We are now done with our AP and hopefully won’t have to touch it again!

![Mini-PC, Archer AP on a Desk with all the cables connected.](https://cdn.hashnode.com/res/hashnode/image/upload/v1769218376965/8fed8434-c21d-4909-9889-5067bf468e48.jpeg align="center")

## Tres - The Director

We have all the pieces ready, we just need to put them together. The key to all this is OPNsense or any other router OS that can help you achieve the same thing. It will take care of your traffic, the routing, the filtering, and other networking requirement you may need. We need an ISO to install OPNsense on our Proxmox, I make sure to get the ISO first from OPNsense, connect the device where I have it stored to my Archer Wi-Fi SSID and set my IP address manually to 192.168.1.10. Then once I can log in to Proxmox (192.168.1.5:8006) I go to local storage &gt; ISO Images &gt; Upload. Then Create VM (ID 100):

* System: Qemu Agent.
    
* Network: Bridge: vmbr1 (LAN) → will become vtnet0 for OPNsense to see.
    
* After creating the VM, Hardware Tab: Add &gt; Network Device &gt; Bridge: vmbr0 → will become vtnet1
    

Now we can start our new OPNsense VM. Once it boots we have to configure the interfaces. In my case it they were set up incorrectly automatically. So through the console, I reassigned the interfaces by:

* Typing 1, hitting Enter (this is to assign interfaces).
    
* LAGGs/VLANs, we type n. (we do not need these).
    
* WAN Interface: we type vtnet0 and Enter (this is vmbr1 which is our enp2s0 NIC).
    
* LAN Interface: we type vtnet1 and Enter (this is vmbr0 which is our enp1s0 NIC).
    
* Optional Interfaces: Enter to skip.
    
* Then we type y and Enter to proceed.
    

Now we can actually take the opportunity to make sure we set up our LAN properly before attempting to log in, to make sure our Archer AP can see OPNsense.

* We type 2 and hit Enter (Set interface IP address).
    
* Select the LAN interface from the list (2 in my case).
    
* Configure IPv4 address via DHCP?: type n.
    
* IP: 192.168.1.1
    
* Subnet: 24
    
* Gateway: None.
    
* DHCP Server: type y.
    
* Start Range: 192.168.1.100
    
* End Range: 192.168.1.200
    

Now we can check if we can log in! Now our OPNsense is our router and DHCP server, so we don’t need to manually set up our IPs anymore. If everything is working fine, our setup should look something similar to this (I added Immich, Pi-hole, Vaultwarden as examples of other services, and we will be setting them up later anyway):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1769222080933/5674b3a6-30a3-433f-95ba-66c37d98eb31.png align="center")

## Cuatro - The Diplomat

Awesome, our server is running and connected. Now the most interesting roadblock (which funnily enough is the easiest to solve *with some caveats*): Getting our server online and accessible from anywhere in the world 🌐! To do this there are many ways and for a home server the best thing you can do is set up a VPN, that way we are not completely exposed (by opening ports for example) to the net which is extremely risky, you can go about this by using OpenVPN or WireGuard to create a tunnel to your server. The problem, if you recall (or if you just skipped to this section) is that I do not have access to the modem connected to my ISP and have to way of opening a port for OpenVPN, which is why I will be using Tailscale. Tailscale is really useful for situations exactly like mine, they use NAT Traversal through UDP Hole Punching, which allows devices to talk to each other.

The first caveat is that you need an account with them (And they claim that is free forever) and limited to 3 users and 100 devices which is plenty for my use-case. Yes, if their server goes down we won’t be able to use our Tailnet (Tailscale Network) but they do cache the connections so it might still work even if their servers are down until the lease expires. I do not like depending on a corporate third party to access my services and files, but given my situation this is probably the most sensible option. It is possible to 'self-host' a service like Tailscale by using Headscale for example, but I think as a starting point, Tailscale is a good idea to get things running and to learn about servers which is what this journey is all about. Eventually, once I move to a new place it is likely that I will switch to WireGuard or OpenVPN if I have access to the modem.

The second caveat is that you need their Tailscale software running in any of the devices that you might want to access your server or services with, the good thing of course is that they have software compatible with pretty much any device, and it gets set up as a VPN with a simple toggle. For example, if we want to access our Immich photos, we would have to turn on the VPN on our phone first, and then open our Immich app to sync with our server.

Okay with that out of the way, let’s set it up! We will do it inside OPNsense, System &gt; Firmware &gt; Plugins. Search for os-tailscale (we have to enable community plugins), and install (it might prompt you to update your OPNsense firmware first). Once installed we need to get our Auth Key from your Tailscale Admin Console (you have to log into Tailscale from a browser) copy it and paste it on our OPNsense Tailscale dashboard. Make sure to check the Enable checkbox.

Finally, to make sure our devices see our server, we have to enable Subnet Routing in OPNsense Tailscale settings, you need to enable Advertised Routes and type 192.168.1.0/24 and click Apply. On the Tailscale Admin Console (on a browser) we need to approve this, find the OPNsense machine, click the three dots, select Edit route settings and Approve the 192.168.1.0/24 route. Boom! We are DONE! We have a server accessible from anywhere in the world! With Tailscale, the diagram would look something like this:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1769225084139/321f1b09-5bb9-4e20-8f9c-501bcc92d690.png align="center")

# What’s Next?

Next we have to set up services to actually use in our new beefy server.

* Immich to manage and view your personal photos and videos with ease.
    
* Pi-hole to protect our network.
    
* Vaultwarden to store passwords.
    

More to be added (like backups, maybe a media server?), but this is my scope for the time being.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text"><strong><em>This Article was created without the use of AI.</em></strong></div>
</div>

<div data-node-type="callout">
<div data-node-type="callout-emoji">📚</div>
<div data-node-type="callout-text"><em>Much of my encouragement and technical inspiration for this project comes from Louis Rossmann’s ‘</em><a target="_self" rel="noopener noreferrer nofollow" href="https://wiki.futo.org/index.php/Introduction_to_a_Self_Managed_Life:_a_13_hour_%26_28_minute_presentation_by_FUTO_software" style="pointer-events: none"><em>FUTO’s Guide to a Self Managed Life</em></a><em>’ - Also available on </em><a target="_self" rel="noopener noreferrer nofollow" href="https://www.youtube.com/watch?v=Gj6HqWCdk3s" style="pointer-events: none"><em>YouTube</em></a><em>. I do of course have many more sources of inspiration that would be impossible for me to list out, though I might go through those at a later time, but his guide was definitely what encouraged me to proceed.</em></div>
</div>