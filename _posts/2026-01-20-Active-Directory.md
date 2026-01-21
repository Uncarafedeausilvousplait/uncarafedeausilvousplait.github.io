---
layout: post
title: How to Set Up an Active Directory Homelab
categories: Portfolio
---
When it comes to directory services, Active Directory is king. Doesn’t matter how you feel about it, doesn’t matter how big your system is, it’s the industry standard for a reason, and if you’ve ever used a windows computer in a corporate environment, you’ve almost certainly used one authenticated and authorized through Active Directory. I’ve worked with AD plenty of times, of course, but I hadn’t set one up from scratch before. Here’s how you do it.

First, you need a good virtualization program. The two best options I knew of for this purpose were VMWare and Oracle’s VirtualBox. I took one look at Broadcomm’s abysmal portal and decided VMWare could wait, resolving to try VirtualBox first. I wanted this to be completely free and local, but if you’re willing, you can try windows server on azure as well – just go to [this link](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025) for the microsoft server information and instead of downloading the iso, try it in azure instead. 

I installed VirtualBox, grabbed the server iso (why microsoft for some reason demands you input a question or comment I will never know), and then it was a simple matter of spinning up a VM using the iso. 

![Why is a comment mandatory anyway?](/assets/Active Directory Project screenshots/portfolio - why is a comment mandatory.PNG)

As for the virtual hardware, I’m on something of a budget with my PC, so I chose to do this with the 2022 server iso and a Windows 10 client, which can run on fewer resources. This let me use a minimum of 2 GB ram, 2 CPUs, and 50 GB of disk space. I had to remember to create a separate partition, which should’ve been obvious, but after that it was just a matter of waiting for the server to install. Once the server VM exists, head into its advanced settings, open the network tab, and make sure both Adapter 1 and Adapter 2 are enabled. Attach Network 1 to NAT, so it can talk to your wider network, and Adapter 2 to intnet so it can talk to the client VM we’ll make later.

Follow standard installation procedures – make a username and password, that sort of thing. Note that when you want to log in, you’ll need to use VirtualBox’s Input menu to inject Ctrl+Alt+Del since pressing those buttons in sequence will affect your host device, not the VM.

Before you dive into the glorious Active Directory server, start by renaming your two networks. Under Network Settings, then Change Adapter options, find the one with the IP address beginning 169 and rename that to indicate it’s the internal network. You don’t want to confuse the two later. Then go into Properties of the internal network and change the IP address to one of the private IP addresses in the 172 block, anything from 172.16.0.0 to 172.31.255.255. Put in 127.0.0.1 for the DNS server since the server functions as its own DNS.

From there, install the AD Domain Services, in Add Roles and Features, click past Before You Begin and under Installation pick Role-based or Feature Based, since we’re not configuring RDS at this time. Under Server Selection pick your server. Click Active Directory Domain Services, and accept the features required for it to function.

After promoting the server to a DC, give it a domain name, and any password you want.

The system is supposed to auto restart afterward, and that gave me an error about Option ROM requiring DDIM support. Restarting fixed it, but I looked it up anyway and everyone said they didn’t know why it happened but restarting fixes it, which I knew already.

We’ll need to create an Organization Unit (OU) and at least one object (the user) using the Users and Computer menu. You can name them whatever you want, just make sure the user is in the Admin group (if it exists, if not, create one). It’s a good idea to log out of your current profile and try the credentials of the user you created.

Now you need to get Remote Access running, so the client can access . Under Add Roles and Features, in the Server Roles menu, find Remote Access and follow the instructions.

Anyway, I installed Remote Access – allowing a client to access the internet through the DC. You’ll want to add Remote Access as a role under “Server Roles” and Routing under “Role Services.” Make sure you agree to the popup that wants to install the prerequisites.

![Setting up remote access](/assets/Active Directory Project screenshots/portfolio - installing remote access.PNG)

When it’s done it should prompt you that configuration is required. For that, head over to tools (top left), find your server under the list at Server Status, then in the drop down menu choose the Configure option. Choose the NAT option, since we’re connecting that way. The wizard should detect the connections already configured, so choose the external network and let the wizard finish the setup.

AD Servers need DHCP, which we haven’t set up yet. Go back to Add Roles and Features, find the DHCP Server under Server roles, and install. As before, if prerequisites are necessary, let the wizard install those as well.

![Installing DHCP](/assets/Active Directory Project screenshots/installing DHCP.PNG)

The system will direct you to configure things, clicking that link will bring up an offer to automatically set up DHCP admins and users. I chose not to do this, since this is mostly a proof of concept, but under normal circumstances I’d absolutely do this.

Either way, we still need to configure the actual server options themselves. Tools → DHCP will show us the IPv4 and IPv6 elements of our DHCP server. The clients I’m creating will only be using IPv4, but in a proper environment that’s not always a guarantee. Open up IPv4, then follow its instructions by choosing the New Scope option from the actions list. If you remember your networking fundamentals, we have access to the entire scope of private IP addresses that are only used internally. I won’t need ALL of them, there are hundreds of thousands of them, but I can set, say, 100 aside with no problem. I chose 172.16.0.150 as the start of the scope and 172.16.0.250 as the end point, so the DHCP Server that I’m setting up will assign new clients an address in that range. And just to make sure, I double checked if I had any static addresses on my network already using those addresses, and I’m clear on that front. I don’t need to exclude any addresses yet, but if a static address is later assigned for a separate service somewhere, having this option is useful. As for the duration, this model can be really whatever you want, but you’d of course default to best practices in a production environment. Make sure you take the option to set a default gateway, use the same address we used earlier – you don’t want the clients looking around more than they have to. I didn’t bother with WINS, just went on to activate the scope.

![The scope](/assets/Active Directory Project screenshots/newscopeexact.PNG)

As things stand I have one admin and one regular user. I could create many more, either manually or using an automated script [like this one](https://github.com/joshmadakor1/AD_PS), but for this proof of concept I made a couple manually. Now I need to set up another VM with standard windows, log in as a user, and poke around from the other side

![I had some fun with the new users, see if you can figure out the reason for the pun](assets/Active Directory Project screenshots/new users - i admit the theme.PNG)

Setting up the VM is very easy, just do what you did with the server iso but with stock windows. Again, I’m using Windows 10 because I’m not in a position to devote all the resources widows 11 needs to a VM, but the same principle applies. [Just make sure it’s Windows 10 Pro or similar or you won’t be able to join the Vm to the domain.](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/join-computer-to-domain?tabs=cmd&pivots=windows-client-10) Once you’ve followed the step and done that, go into “Active Directory User and Computers” under Tools in the server and you should be able to find the client under Computers.

Now that the computer is added to the system, go ahead and log in using one of the user accounts you made already, and once it works, hey presto your AD server is up and running.

Seeing it in a homelab environment might seem a little bit underwhelming, but that’s only because you went out of your way to thoroughly divorce it from the intended use case. The point is to have huge numbers of computers and accounts and to be able to control and oversee them from one location. By having them all by VMs, and only two VMs at that, it’s easy to lose track of why this is so important. But important it is, and having hands-on experience is vital.

As for improvements on this exercise, the speedrunner in me wants to try and do this against the clock – seeing how quickly I can set it up without mistakes. Being able to do that will guarantee a deep understanding, or at least strong memorization skills.