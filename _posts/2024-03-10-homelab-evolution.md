---
title: Homelab Evolution
author: metheis
date: 2024-03-10 22:00:00 -0700
categories: [Servers, Networking]
tags: [docker, kvm, qemu]
pin: false
---

Back during my highschool years, I spent a lot of time on projects & buildings things, and that is where I constructed my first [homelab](https://linuxhandbook.com/homelab/). For me, it was the answer to an old Netgear NAS that my parents had in the basement (but was quite slow). Pretty much all of my family members used it, storing everything from school essays, to photos, computer games, music, etc. It was both a neat undertaking to convert it into a small tower server, and daunting to make sure I didn't make a mistake and lose anyone's data. In 2019, I significantly overhauled it, moving to SATA SSDs for storage (from HDDs), a small Optane NVMe boot drive, a server grade motherboard with IPMI support, and more RAM (of course).

When it comes to the OS and services running, the original iteration was simply Samba installed directly on Ubuntu to provide the same CIFS functionality of a NAS. I quickly started to explore other additions, from management tools like Webmin, to adding a Minecraft server, and eventually to Plex. It were these additions that provoked hardware upgrades.

## Application Deployment Maturation

![Container & VMs Diagram](/assets/img/2024-03-10-homelab-evolution/containers-and-vms-together-1.jpg)
_Source: Docker

The original setup for applications consisted of compiled binaries running directly on a bare metal OS with all requisite dependencies installed. I've run into my fair share of dependency mismatch scenarios, namely breakages after an update or installation of another program that uses a different version of the same package. A couple of particularly frustrating pickles ended in reinstalling the entire boot drive as the easiest method to get things back in order.

Around that time, I discovered Docker & containerization, of which one of the key offerings is to solve this exact problem: each docker image includes all requisite packages with itself, and touches none in the main operating system or of other containers. 

Accompanying the major 2019 hardware upgrade, I started with a fresh OS install (transitioning to Intel's Clear Linux, which I discuss in a previous post), and then exclusively employed Docker to run services from the get-go. 

I transitioned the file server access and management to Nextcloud, which was more useful as my brothers and I were now out of the house and could access that anywhere. If you browse the [example configurations](https://github.com/nextcloud/docker/blob/master/.examples/docker-compose/with-nginx-proxy/mariadb/fpm/docker-compose.yml) for Nextcloud Docker, it actually utilizes Docker Compose and multiple containers together to create one service. Beyond the hello-world container, this was my first real deployment of a Docker service. It required a significant amount of research to understand how persistent file storage works, its networking configuration, etc. But the powerful nature of its composition meant I could hook the needed files up to Plex and still maintain the principles of containerization (Plex also running in its own separate container).

As an ancient Greek once said, ["The only constant in life is change."](https://arapahoelibraries.org/blogs/post/the-only-constant-in-life-is-change-heraclitus/) And that is certainly true of the homelab (as is somewhat the point). Not long after I had Nextcloud and Plex working, I decided that I would like to deploy a Minecraft server to this new system, but was interested in a bit more security. Docker containers share the same kernel as the host operating system, and given the history of vulnerabilities found in Minecraft server (looking at you [log4j](https://www.pcmag.com/opinions/critical-exploit-for-apache-log4j2-could-be-far-reaching-proves-real-in)), I did not want to leave this attack surface open. This meant adding a virtualization layer to completely contain the application.

There are virtualized containers, most notably Kata Containers, which I had tried out in the first go around with Plex. However, for reasons beyond what I debugged to find out, the Plex Media Server image refused to function properly in that environment. So I was not too inspired by the promise of Kata.

Instead, I decided to deploy a full virtual machine to run the Minecraft server inside of it. This is a well established means to achieve the extended security I was looking for, and with modern Linux, you don't even need a dedicated virtual machine monitor running on the hardware. Linux has the Kernel Virtualization Module that can be activated to offer near bare metal hypervisor performance while still offering the standard suit of operating system resources to other programs. Moreover, they allow for granular control of how many resources the services running in the VM can use, which helps reserve capacity for services that are more "important" than others.

Once inside the virtual machine, I was back to the consideration of how I wanted to deploy the service (in this case, the Minecraft server) via either direct install or Docker. I only intended to use this VM for this specific application, so I could install it directly onto the guest OS (in my case Ubuntu), and I would have less to worry about with dependency overlap or mis-cross-configuration compared to a multi-service machine. That would also correspond to slightly better performance as it could run without the Docker engine taking any resources. This was the initial approach I chose, and I got the server up and running! However, when updating the server, the matrix of dependencies remained (especially with Java). I figured that the overhead to containerize an application is miniscule enough for modern systems that the benefits of portability and customized configuration outweigh it. Therefore, I ended up back on the other path: the Minecraft server inside of a Docker container, inside of the virtual machine, on the physical server 🙃. In short, my setup was starting to look more like that of the diagram above.

## It Get's Better

What do we have now? Well, it's a little bit of a mess. Two of the services are deployed via Docker, and another in Docker inside of a VM. Why not make it consistent? (And by that making it even more complicated in the end).

I decided to also move Nextcloud and Plex to VMs (choosing Rocky Linux as the guest OS). Now, for some reason beyond my current logic, I decided it would be easier to deploy Nextcloud from a Snap package instead of through Docker. Firstly, why not try something else, right? Secondly, I had run into some minor issues with installing things like PHP extensions through the original Docker setup. So a curated all in one package sounded better! Snap also containerizes the application, so it maintains the benefit from the dependency standpoint, and it also has the plus of the install being a single command instead of an entire configuration file. It was indeed easier to get Nextcloud up and running, but I found myself unable to configure beyond the original Snap's vision, for example, to make a significant change to the proxy configuration. This meant that I eventually disliked this approach for the long run. Back to Docker I went!

Plex followed similarly (without the whole Snap part :), and now I needed to access the proper media files stored in Nextcloud. No longer quite so easy to just share volumes or mounts given they were running inside two separate machines, I added NFS to the mix to create a network share. From there, it was more or less similar to the original Docker setup, now with VM security. Overall, I ended with a uniform system: a handful of services spun up inside of VM's utilizing containers for easy deployability, and the manageability of this design has been the best all around to date!

By now, you're probably thinking, "Mark, you really need a precise diagram of what's going on here." And honestly, I should probably make one to keep track of everything myself! I'll put one together for the current state of things and post it to a follow up. Since 2022, I've added [Cloudflare Access](https://www.cloudflare.com/zero-trust/products/access/) to both more easily and securely manage the VM's on the server, Netdata health monitoring, and added more VM's! I also want to share how that was put together in a more tutorial style post, so stay tuned!


