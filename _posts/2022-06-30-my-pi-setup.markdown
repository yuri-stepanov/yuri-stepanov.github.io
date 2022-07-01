---
layout: post
title: "My Pi setup"
date: 2022-06-30
categories: Raspberry-Pi
---

# My Raspberry Pi setup

1. [Motivation and assumptions](#motivation)
2. [Tailscale](#tailscale)
3. [SSH](#ssh)
4. [UFW](#ufw)
5. [Samba](#samba)
6. [Code-server](#code-server)
7. [IPad setup](#ipad-setup)
8. [Mosh](#mosh)
9. [Conclusion](#conclusion)

## 1. Motivation and assumptions {#motivation}

Some of you may have a Raspberry Pi laying around.
You've bought it on a whim and wanted to build something with it.
Me too!
Even though the internet is full of interesting [Pi projects](https://www.reddit.com/r/raspberry_pi/), I've never got around to finish one (automatic water supply for plants was my dream).
But Pi can be useful for some simple everyday tasks.
Here I will try to show you how I got around to use mine.
Also this is a reference for myself on how to reinstall everything.

I assume that Raspberry OS installed and the Pi is connected to your local network.
Before we setup SSH and UFW we need to have a monitor and a keyboard connected to the Pi.
Other steps can be done remotely.
You will also need a separate (main) computer with a terminal from which you are going to connect to your Pi.

## 2. Tailscale {#tailscale}

Tailscale can be described as managed WireGuard.
We will use it to create our own VPN.
With that you will be able to access our Pi from anywhere securely.
A guide on how to start with tailscale can be found [here](https://tailscale.com/kb/1017/install/).
And on how to install it on a Pi can be found [here](https://tailscale.com/download/linux/rpi).
By the end you should see your Pi in the [console](https://login.tailscale.com/admin/machines).
I use MagicDNS. You can enable it [here](https://login.tailscale.com/admin/dns).
I have changed my machine name to `pi`.
Hereafter I will use it as my Pi DNS name.

## 3. SSH {#ssh}

I want SSH service only to listen to connections made through our VPN.
For that we are going to change the interface on which SSH listens to for the incoming connections.
SSH config file usually located here `/etc/ssh/sshd_config`.
Open it in any editor and change the line with `ListenAddress` option.

```
ListenAddress <Your_PI_IP4_tailscale_address_here>
```

You can find Pi's address in the [console](https://login.tailscale.com/admin/machines).

During the startup tailscale service can start a bit later or for example something might be wrong with the network.
This will make SSH service fail. The easy way to fix that is by adding an option to restart SSH service automatically.  Open this file `/etc/systemd/system/multi-user.target.wants/ssh.service` and delete the following line `RestartPreventExitStatus`. Also change the following lines.

```
Restart=on-failure
RestartSec=5
```

Let's start SSH service and make sure it will be started during the startup.
On the Pi with a connected monitor and keyboard execute the following commands in the console.

```
sudo systemctl start ssh
sudo systemctl enable ssh
```

By default we can login using the username and password, that you have created during Pi setup.
It would be easier and safer to login using a key.
Let's create a new key.
On your main machine execute.

```
ssh-keygen
```

This command will generate a public / private key pair in the `~/.ssh` folder.
By default they will be given names `id_rsa` and `id_rsa.pub`.
Let's copy our key to the Pi.

```
ssh-copy-id -i ~/.ssh/id_rsa pi@pi
```

Here the second `pi` is the username.
With that we should be able to login remotely to our Pi.
Try to execute on your main machine.

```
ssh pi@pi
```

If all goes well you should see your prompt changed to Pi's.
Now we can disable login with the password by changing the following options in `/etc/ssh/sshd_config`.

```
PubkeyAuthentication yes
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no
```

Restart SSH service for changes to take effect.

```
sudo systemctl restart ssh
```

After that you should be able to login to your Pi using a key only.

## 4. UFW {#ufw}

Or Uncomplicated Firewall is an interface for `iptables`.
We will use it to lock down any ports that we are not using.
Be careful. Until we finish this stage you should be able to connect the monitor and keyboard.
It is very easy to get yourself locked out of the Pi.

First we make sure that by default we deny any incoming connection.

```
sudo ufw default deny incoming
```

And allow outgoing.

```
sudo ufw default allow outgoing
```

Then we open `ssh` port.

```
sudo ufw allow in on tailscale0 to any port 22
```

Here `tailscale0` is the name of the interface for which `ufw` will create a rule.
You can get all available interfaces on your system by running.

```
ip address
```

Let's make sure that the rule was created by running.

```
sudo ufw status
```

You should see something like this.

```
To                         Action      From
--                         ------      ----
22 on tailscale0           ALLOW       Anywhere
22 (v6) on tailscale0      ALLOW       Anywhere
```

Disable and reenable UFW to changes to take effect.

```
sudo ufw disable
sudo ufw enable
```

I will be using UFW throughout the rest of the chapters to open ports for relevant services.

## 5. Samba {#samba}

I have an external hard drive that I use as a fileshare.
We are going to turn it into a samba share that is available through tailscale.

Open `/etc/samba/smb.conf`

In the end add a section.

```
[share]
 comment = Hard drive
 path = /home/pi/share
 browsable = yes
 writeable = yes
 read only = no
 create mask = 0755
 force user = pi
```

`path` here is where you have mounted your external hard drive.

Enable and start the service.

```
sudo systemctl enable smbd
sudo systemctl start smbd
```

We also need to open port for samba.

```
sudo ufw allow in on tailscale0 to any port 445
```

Connect to share using your file manager.
For the MacOS open Finder, use the top menu `Go -> Connect to Server` and paste URL `smb://pi`.
For the user use `pi`.

> ### Sidenote
>
> I have tried to use `interfaces` and `bind interfaces only` option in `smb.conf` to force samba to use the tailscale interface but unsuccessfully.
> If you specify only tailscale interface and enable `bind interfaces only` you will get the following error:
>
> ```
>
> source3/smbd/server.c:1244(open_sockets_smbd)
> open_sockets_smbd: No sockets available to bind to.
>
> ```
>
> It might have something to do with the comment from the `smb.conf` file:
> `option cannot handle dynamic or non-broadcast interfaces correctly`

## 6. Code-server {#code-server}

I sometimes find that bringing my IPad is easier than my laptop.
To be able to code properly I will use code-server to run a vscode instance on the Pi and connect to it remotely.
Install code-server by running this command

```
curl -fsSL https://code-server.dev/install.sh | sh
```

Enable and start code-server

```
sudo systemctl enable code-server@pi
sudo systemctl start code-server@pi
```

Open config file `~/.config/code-server/config.yaml`

```
bind-addr: <Your_PI_IP4_tailscale_address_here>:8080
auth: password
password: <will be generated>
cert: false
```

Restart code-server service

```
sudo systemctl restart code-server@pi
```

Open 8080 port

```
sudo ufw allow in on tailscale0 to any port 8080
```

Now you can access the vscode instance on `http://pi:8080`.

## 7. IPad setup {#ipad-setup}

I am using Blink as a terminal.
Lately they have introduced a VSCode UI in their app.
Let's configure it.
First open the config by typing.

```
config
```

Create a host for Pi.
In the SSH config section put.

```
LocalForward 8080 <Your_PI_IP4_tailscale_address_here>:8080
```

First you need to connect to Pi. In the Blink app run

```
ssh pi@pi
```

Then create a new tab and run

```
code http://localhost:8080
```

Code editor should open. Use the password from the config file to login.

> ### Sidenote
>
> Unfortunately the step of connecting via SSH to Pi is required.
> If you just execute `code http://localhost:8080` nothing will happen.

## 8. Mosh {#mosh}

Mosh allows you to maintain your SSH connection over unstable networks.
Even if you switch from WiFi to cellular it won't drop the connection.
Install it on Pi.

```
sudo apt-get install mosh
```

By default mosh will try to use a range of udp ports starting from 60000.
I don't want to open so many ports so I usually use the command option to use only 60000.
Open this port.

```
sudo ufw allow in on tailscale0 to any port 60000 proto udp
```

On the client that you want to connect from, execute.

```
mosh -p 60000 pi@pi
```

## 9. Conclusion {#conclusion}

Every computer in my family is connected to samba via tailscale.
We can store and share files with each other.
I can connect to the vscode from my IPad and code from anywhere.

`It aint much, but it's honest work.`
