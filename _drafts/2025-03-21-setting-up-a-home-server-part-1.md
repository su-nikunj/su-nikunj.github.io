---
layout: post
title: Setting up a home server - Hardware and OS
date: 2025-03-22 08:00 +0200
categories: [Home Server, Getting Started]
tags: [home server, setup]
---

This blog post is the first out of many in my journey to setup my home server. The first plan of action is to choose a hardware and software for my home server.

### Motivation
So first question would be, why do I want a home server? And the simple answer would be, why not? I love the concept of DIY, so why not? But the main motivation was the constant annoyances with all the streaming services raising their prices, reducing their overall quality of service, etc.

So for this home server, I want to setup a media server, along with a few other services like calibre for my ebooks, a network wide ad-blocker, etc.

In the future, I would like to setup my own cloud and image library and get away from Google's ecosystem.

### Hardware Selection
Time to shop for the PC that would become my home server. After watching a lot of YouTube videos, I was interested in a NAS system. But those prebuilt ones were super expensive with such a weak CPU. Then I thought of building it by myself. I like the idea of building my own PC, but then I found Ace Magician's AM06 Pro Mini PC.

![Acemagic AM06 Pro](https://acemagic.com/cdn/shop/files/AM06_-1.jpg?v=1718699221){: width="300" height="300"}

This Mini PC has a Ryzen 5 5625U CPU. But wait, don't you need an Intel CPU for media servers? Well, yes. Intel CPUs have this technology called QuickSync used for hardware encoding and decoding. But the mini PCs with Intel chips were all Celerons or equivalent. I couldn't find any mini PCs with a powerful intel chip. Plus, this Ryzen chip is a laptop chip, so it has onboard graphics for hardware encoding/decoding and it also consumes less power while having 6 cores and 12 threads.

So I decided to go with this mini PC which came with 16 GB of RAM and 512 NVME SSD. I also bought a 4 TB SATA SSD that goes inside it. It comes pre-installed with Windows, but I will be wiping it off soon and tucking this little thing in the corner, out of sight, where it will run in silence 24x7.

> The problem with this mini PC is that it doesn't have space for additional drives and the internal space is only big enough for one 2.5" SATA drive. So if you are going to be storing any critical data on it, there won't be any redundancy. Be sure to have a backup solution (Well you should be doing that even with a redundancy in place).
{: .prompt-warning}

I won't be going for backups because I don't plan on setting it up for anything critical (at least for now).

### Operating System
Now onto the main topic of this post, what operating system should I install on this. Usually, we have three categories to choose from:

* A Hypervisor OS
* A NAS OS
* Just a regular base OS

#### Hypervisor OS
Hypervisors are used for running other Operating Systems on top of them in Virtual Machines or Containers.The most popular choice is [Proxmox](https://proxmox.com/en/). It provides a Web interface for creating and controlling Virtual Machines or LXCs easily. What are LXCs? They're Linux Containers that shares kernel with the host and are much more lightweight that Virtual Machines, but you can only run Linux on them, as the name suggests.

So does Proxmox fit in my use case? Yeah I can run separate containers for my applications. But that will be bringing issues of hardware passthrough to all the containers and VMs. Also I don't really have a use case of running multiple virtual machines or LXCs. I would rather prefer all apps run in containers that share the same hardware resources. Therefore, I decided maybe I should give the other options a shot and see if they fit my needs better.

#### NAS OS
These Operating Systems are designed to be run on a Network Attached Storage (NAS) system. Their primary purpose is to serve files over the network, but they have evolved into supporting other functions as well such as having Docker support for installing apps, ability to create virtual machines, etc. The most popular options are [TrueNAS Scale](https://www.truenas.com/truenas-scale/), [UnRAID](https://unraid.net/) and [OpenMediaVault](https://www.openmediavault.org/).

UnRAID was a paid option, so I'll leave it for now, but it is quite a popular option. TrueNAS scale provides a lot of other benefits such as easy RAID functionality, ZFS storage pools, docker apps, and much more. This seems like a good fit. All apps share the same resources as the OS so no headaches there. But that got me thinking, do I really need all the functionalities provided by TrueNAS? I only want to run a Samba share on my SSD and run a few docker apps, so couldn't I do it on a much more lightweight Operating System? Plus TrueNAS doesn't seem to allow using your boot drive for any storage which seems like a complete waste. 

OpenMediaVault is lightweight, but how about going even more lightweight and doing it yourself?

#### Regular Base OS
How about just installing a regular Linux distribution and setting up a Samba share by myself. There are easy ways to do it through Web GUI as well, no need to linger around the terminal too much.

Now before choosing a Linux distribution, I should mention that you also have the option of installing a Windows server, if you wanna do it for some reason, I'm not judging.

So for Linux distributions, should I go with the popular Ubuntu, or one of the RedHat clones such as AlmaLinux and RockyLinux. While these are great for businesses, what I need is a simple Linux Distribution without much pre-installed stuff such as good ol' Debian Linux.

### Next steps
In the next post, I'll be setting up my Debian installation and installing some useful applications.