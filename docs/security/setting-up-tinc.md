---
author:
  name: Linode Community
  email: docs@linode.com
description: 'This guide will show you how to set up a TINC VPN on your Linode.'
keywords: 'VPN, tinc, Ubuntu, CentOS, security'
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
published: 'Tuesday, September 19th, 2017'
modified: Wednesday, September 20th, 2017
modified_by:
  name: Linode
title: 'Setting Up tinc'
contributor:
  name: Damaso Sanoja
external_resources:
 - '[tinc Project](https://www.tinc-vpn.org/)'
---

*This is a Linode Community guide. [Write for us](/docs/contribute) and earn $300 per published guide.*
----

Virtual Private Networks (VPN) are an essential part of any serious security-wise network deployment. There are many open source VPN options but one of them shines above the others: *tinc*. While all VPNs behave as a secure tunnel between two points, tinc stands out because of its "Peer to Peer" design. This allows tinc users a great deal of flexibility, especially when planning a mesh-type network.

This guide will describe how to configure tinc VPN in three different use-case scenarios, from a simpler two-server connection to a more complex mesh private-network.

## Before You Begin

To complete all of the different configurations in this guide, you will need multiple Linodes. For each Linode, complete the following steps:

1.  Complete the [Getting Started](/docs/getting-started) guide.

2.  Follow the [Securing Your Server](/docs/security/securing-your-server/) guide to create a standard user account, harden SSH access and remove unnecessary network services.

    This guide will use `sudo` wherever possible. Please ensure you have access to privileged user rights.

3.  Update your packages:

		sudo apt update && sudo apt upgrade

{: .note}
>
>In order to focus on Tinc configuration, three assumptions are made:
>1. There are no active firewalls on any server.
>2. Each server is connected directly to the Internet (no router or proxy is involved).
>3. Each server is running the same version of Tinc.

## Basic Setup: Two Node Tinc VPN

The first use-case for tinc is a very simple setup involving only two instances, an Application Server *(appserver)* and a Database Server *(dbserver)*. Each instance will run on a separate Linode:

![Two Node VPN](/docs/assets/tinc-two-node.png)

### Gathering network information

Before getting started, it's a good idea to create a "cheat-sheet" with network information, including the public IPv4 addressses for both servers, desired VPN address for each instance, VPN network name designations, and a tinc-daemon name for each node. The VPN address can be any arbitrary private network IPv4 address, the only rule to follow (if you want to avoid extra routing work) is that they must have the **same network prefix**, just like a typical LAN. VPN and daemon names must be unique and cannot contain any spaces or special symbols. For the current use-case the following information will be used for Tinc configuration:

![Two node VPN cheat-sheet](/docs/assets/tinc-2-node-cheat-sheet.jpg)

{:.note}
> Throughout this guide, replace the IP address for each server with the public IP address of the corresponding Linode.

### Installing Tinc VPN on Ubuntu 16.04

Ubuntu 16.04 will be used for all of the Linodes in this guide. Other distros can be used, but the steps to install the latest version of tinc will vary. At the time of this writing the latest stable version of tinc is 1.0.31. Ubuntu's repositories use an older version, so you will have to build from source:

1. Install the necessary dependencies for building tinc:

        sudo apt install build-essential automake libssl-dev liblzo2-dev libbz2-dev zlib1g-dev

2. Get the latest version of tinc from developer's site:

        wget http://tinc-vpn.org/packages/tinc-1.0.31.tar.gz

3. Extract the archive in a temporary folder:

        tar -xf tinc-1.0.31.tar.gz

4. Compile tinc in your system:

        cd tinc-1.0.31
        ./configure --prefix=
        make
        sudo make install

### Creating VPN working directory

You can implement as many tinc networks as you need as long as you create a unique working directory for each one (gaming VPN, backups VPN, etc). The name of the folder must match the designated name for your VPN–in this case *linodeVPN*. You also need to use the same structure across the servers, which means you need to create the working directory on both:

        sudo mkdir -p /etc/tinc/linodeVPN/hosts

### Creating configuration files

You will require to set up a "main" configuration file `tinc.conf` on each server. The default location for this file is the root of the working directory.

1.  Let's start with the Application Server file:

{: .file}
/etc/tinc/linodeVPN/tinc.conf
:   ~~~ conf
	Name = appserver
	Device = /dev/net/tun
	AddressFamily = ipv4
	~~~

* `Name` - This is daemon-specific name within the VPN. It should be unique.
* `Device` - Determines the virtual network to use, tinc will automatically detect what kind of device is.
* `AddressFamily` - Tell tinc which type of address to use.

2.  Now you need to generate the Database Server configuration file:

{: .file}
/etc/tinc/linodeVPN/tinc.conf
:   ~~~ conf
	Name = dbserver
	Device = /dev/net/tun
	AddressFamily = ipv4
	ConnectTo = appserver
	~~~

`ConnectTo` - This value points to the tinc daemon you want to connect. When is not present (like the previous file) tinc enter in listening mode, waiting for connections.

{: .note}
>
>You can customize tinc behavior with many other parameters available for the configuration file, for more information visit the [tinc documentation](https://www.tinc-vpn.org/documentation/tinc.conf.5).

3.  Create the **hosts** configuration files. Because tinc is build using a Peer to Peer model each node needs to know about the others. The basic host file for the Application Server is:

{: .file}
/etc/tinc/linodeVPN/hosts/appserver
:   ~~~ conf
	Address = 11.11.11.11
	Subnet = 192.168.100.209
	~~~

4.  In similar way create a host file for Database Server:

{: .file}
/etc/tinc/linodeVPN/hosts/dbserver
:   ~~~ conf
	Address = 22.22.22.22
	Subnet = 192.168.100.130
	~~~

5.  Add the public key of each node. Tinc can create the key pair using the following command:

        sudo tincd -n linodeVPN -K 4096

    For this example we are using 4096-bit encryption, you can choose a higher level if necessary. Save both keys `rsa_key.pub` and `rsa_key.priv` in the root of your working directory `/etc/tinc/linodeVPN`.

6.  Next, append the `rsa_key.pub` at the end of each host file. From the Application Server:

            sudo bash -c "cat /etc/tinc/linodeVPN/rsa_key.pub >> /etc/tinc/linodeVPN/hosts/appserver"

    From the Database Server run a similar command:

            sudo bash -c "cat /etc/tinc/linodeVPN/rsa_key.pub >> /etc/tinc/linodeVPN/hosts/dbserver"

### Creating VPN Control Scripts

Control scripts are the responsible for setting up the virtual interfaces on each server. They are needed on both servers.

1.  From the Application Server, create the following file to enable the tinc interface:

    {: .file}
    /etc/tinc/linodeVPN/tinc-up
    :   ~~~ conf
    	#!/bin/sh
    	ip link set $INTERFACE up
    	ip addr add 192.168.100.209 dev $INTERFACE
    	ip route add 192.168.100.0/24 dev $INTERFACE
    	~~~

    Create a script to disable the interface:

    {: .file}
    /etc/tinc/linodeVPN/tinc-down
    :   ~~~ conf
        #!/bin/sh
        ip route del 192.168.100.0/24 dev $INTERFACE
        ip addr del 192.168.100.209 dev $INTERFACE
        ip link set $INTERFACE down
    	~~~

2.  Create a similar set of scripts from the Database Server:

    {: .file}
    /etc/tinc/linodeVPN/tinc-up
    :   ~~~ conf
        #!/bin/sh
        ip link set $INTERFACE up
        ip addr add 192.168.100.130 dev $INTERFACE
        ip route add 192.168.100.0/24 dev $INTERFACE
    	~~~

        And another script to shutdown it:

    {: .file}
    /etc/tinc/linodeVPN/tinc-down
    :   ~~~ conf
        #!/bin/sh
        ip route del 192.168.100.0/24 dev $INTERFACE
        ip addr del 192.168.100.209 dev $INTERFACE
        ip link set $INTERFACE down
    	~~~

After creating the control scripts you will need to change the permissions accordingly (on both servers):

        sudo chmod -v +x /etc/tinc/linodeVPN/tinc-{up,down}

### Create tinc Unit File

In order to run tinc as a service on startup, you will need to set up the proper Unit file for each server. Edit the following file from both the Application and Database servers.

{:.file}
/etc/systemd/system/tinc.service
:   ~~~ conf
	[Unit]
	Description=Tinc net linodeVPN
	After=network.target

	[Service]
	Type=simple
	WorkingDirectory=/etc/tinc/linodeVPN
	ExecStart=/sbin/tincd -n linodeVPN -D -d3
	ExecReload=/sbin/tincd -n linodeVPN -kHUP
	TimeoutStopSec=5
	Restart=always
	RestartSec=60

	[Install]
	WantedBy=multi-user.target
    ~~~

{: .note}
>
>For tinc Unit a debug level of "3" was choosen in the `tincd` command. This will log all requests from other daemons and include some authentication interchange between them. For more information about debug levels please read the [tincd documentation](https://www.tinc-vpn.org/documentation/tincd.8).

### Host files interchange

Up to this point, you have all configuration files created on each server. Because of tinc's P2P nature, the last thing you need to do is to interchange host files between nodes. There are many ways to accomplish that, for the purpose of this guide we will use `scp`.

From `appserver`

        scp /etc/tinc/linodeVPN/hosts/appserver <user>@<dbserver>:/tmp/
        ssh -t <user>@<dbserver> sudo mv -v /tmp/appserver /etc/tinc/linodeVPN/hosts/

From `dbserver`

        scp /etc/tinc/linodeVPN/hosts/dbserver <user>@<appserver>:/tmp/
        ssh -t <user>@<appserver> sudo mv -v /tmp/dbserver /etc/tinc/linodeVPN/hosts/

### tinc Testing

It is time for testing your newly configured VPN. From Application Server start the `tincd` daemon:

        sudo tincd -n linodeVPN -D -d3

And now from Database Server do the same:

        sudo tincd -n linodeVPN -D -d3

If everything has been configured successfully you should see an output similar to this:

![VPN output](/docs/assets/appserver-dbserver-output.jpg)

You can open another terminal (i.e. from `appaserver`) and check active interfaces:

        sudo ip a

You should have at least three interfaces, one of them your `linodeVPN`.

Another quick test you can perform is to `ping` Database Server from Application Server using its VPN address:

        ping 192.168.100.130

If you encounter errors during your testing you can always add this flag to your daemon command:

        sudo tincd -n --logfile[=FILE] linodeVPN -D -d5

### Finishing your tinc configuration

Once you have your tinc VPN ready you can enable the service on startup:

        sudo systemctl enable tinc.service

Now you can start, stop, restart and check the service using common `systemd` syntax.

        sudo systemctl status tinc.service

## Deploying a Three Node tinc mesh

For the purpose of this guide we will use the following use-case scenario:

![Three Node VPN](/docs/assets/tinc-three-node.png)

In the interest of showcasing tinc's easy expandability, we will assume that **LINODE 1** and **LINODE 2** are the same Application Server and Database Server from the previous section.

The "Cheat-Sheet" for this topology is:

![Three node cheat-sheet](/docs/assets/tinc-3-node-cheat-sheet.jpg)

1.  Configure the new webserver (Linode 3). Follow the previously explained steps to install tinc VPN and create its working directory.

2.  The main configuration file for this instance would be:

    {: .file}
    /etc/tinc/linodeVPN/tinc.conf
    :   ~~~ conf
    	Name = webserver
    	Device = /dev/net/tun
    	AddressFamily = ipv4
    	ConnectTo = appserver
    	ConnectTo = dbserver
    	~~~

3.  Edit the host file:

    {: .file}
    /etc/tinc/linodeVPN/hosts/webserver
    :   ~~~ conf
    	Address = 33.33.33.33
    	Subnet = 192.168.100.140
    	~~~

4.  For the encryption keys, this time increase the security level:

            sudo tincd -n linodeVPN -K 8192

5.  Once again, you need to append the public key to the host file:

            sudo bash -c "cat /etc/tinc/linodeVPN/rsa_key.pub >> /etc/tinc/linodeVPN/hosts/webserver"

6.  Add control scripts:

    {: .file}
    /etc/tinc/linodeVPN/tinc-up
    :   ~~~ conf
    	#!/bin/sh
    	ip link set $INTERFACE up
    	ip addr add 192.168.100.140 dev $INTERFACE
    	ip route add 192.168.100.0/24 dev $INTERFACE
    	~~~


    {: .file}
    /etc/tinc/linodeVPN/tinc-down
    :   ~~~ conf
        #!/bin/sh
        ip route del 192.168.100.0/24 dev $INTERFACE
        ip addr del 192.168.100.140 dev $INTERFACE
        ip link set $INTERFACE down
    	~~~

7.  Remember to change permissions:

            sudo chmod -v +x /etc/tinc/linodeVPN/tinc-{up,down}

8.  The Unit file on this server will be identical to the previous servers:

    {: .file}
    /etc/systemd/system/tinc.service
    :   ~~~ conf
    	[Unit]
    	Description=Tinc net linodeVPN
    	After=network.target

    	[Service]
    	Type=simple
    	WorkingDirectory=/etc/tinc/linodeVPN
    	ExecStart=/sbin/tincd -n linodeVPN -D -d3
    	ExecReload=/sbin/tincd -n linodeVPN -kHUP
    	TimeoutStopSec=5
    	Restart=always
    	RestartSec=60

    	[Install]
    	WantedBy=multi-user.target
        ~~~

9.  Just like the previous chapter, the last step is to interchange host files between all nodes. So you will need to copy Application and Database servers host files to Webserver and Webserver's host file to both of them.

    From `webserver`:

        scp /etc/tinc/linodeVPN/hosts/webserver <user>@<appserver>:/tmp/
        ssh -t <user>@<appserver> sudo mv -v /tmp/webserver /etc/tinc/linodeVPN/hosts/

        scp /etc/tinc/linodeVPN/hosts/webserver <user>@<dbserver>:/tmp/
        ssh -t <user>@<dbserver> sudo mv -v /tmp/webserver /etc/tinc/linodeVPN/hosts/

    From `appserver`:

        scp /etc/tinc/linodeVPN/hosts/appserver <user>@<webserver>:/tmp/
        ssh -t <user>@<webserver> sudo mv -v /tmp/appserver /etc/tinc/linodeVPN/hosts/

    From `dbserver`:

        scp /etc/tinc/linodeVPN/hosts/dbserver <user>@<webserver>:/tmp/
        ssh -t <user>@<webserver> sudo mv -v /tmp/dbserver /etc/tinc/linodeVPN/hosts/


10.  Finally, modify the `tinc.conf` file on `appserver` and `dbserver`. The new configuration files need to tell tinc to look for other nodes:

{: .file}
/etc/tinc/linodeVPN/tinc.conf
:   ~~~ conf
	Name = appserver
	Device = /dev/net/tun
	AddressFamily = ipv4
	ConnectTo = dbserver
	ConnectTo = webserver
	~~~


{: .file}
/etc/tinc/linodeVPN/tinc.conf
:   ~~~ conf
	Name = dbserver
	Device = /dev/net/tun
	AddressFamily = ipv4
	ConnectTo = appserver
	ConnectTo = webserver
	~~~

11.  Now you can start the tinc service on the webserver:

        sudo systemctl enable tinc.service
        sudo systemctl start tinc.service


As you can see, adding a new node to a tinc VPN is very easy. Once you get a hold of configuration files the rest is trivial.

## Using tinc for Centralized Cloud Interconnection

In the last use-case, you will configure a VPN with four LINODE servers. Each one is in a different geographic location.

![Centralized Cloud VPN](/docs/assets/tinc-four-node.png)

The Cheat-Sheet in this case is:

![Centralized VPN cheat-sheet](/docs/assets/tinc-4-node-cheat-sheet.jpg)

After the previous two sections, you should be familiar with the configuration procedure:

1. Install tinc VPN on each node.
2. Create VPN working directory on each node.
3. Create `tinc.conf` and host files on every node.
4. Create VPN control scripts for all tinc instances.
5. Create Unit files.
6. Interchange host files across all nodes.
7. Enable tinc Unit(s)

The configuration file `tinc.conf` on Linodes 1, 2 and 3 should point only to Linode HQ (`ConnectTo`). The configuration file on Linode HQ should define connections with all nodes using the same directive. The corresponding configuration files are:

Linode1:

{: .file}
/etc/tinc/linodeVPN/tinc.conf
:   ~~~ conf
	Name = linode1
	Device = /dev/net/tun
	AddressFamily = ipv4
	ConnectTo = linodeHQ
	~~~

Linode2:

{: .file}
/etc/tinc/linodeVPN/tinc.conf
:   ~~~ conf
	Name = linode2
	Device = /dev/net/tun
	AddressFamily = ipv4
	ConnectTo = linodeHQ
	~~~

Linode3:

{: .file}
/etc/tinc/linodeVPN/tinc.conf
:   ~~~ conf
	Name = linode3
	Device = /dev/net/tun
	AddressFamily = ipv4
	ConnectTo = linodeHQ
	~~~

And finally the central server:

{: .file}
/etc/tinc/linodeVPN/tinc.conf
:   ~~~ conf
	Name = linodeHQ
	Device = /dev/net/tun
	AddressFamily = ipv4
	ConnectTo = linode1
	ConnectTo = linode2
	ConnectTo = linode3
	~~~