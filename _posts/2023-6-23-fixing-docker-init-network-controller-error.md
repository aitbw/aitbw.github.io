---
layout: post
subtitle: Troubleshooting a Docker network error the right way
tags: [docker, troubleshooting]
readtime: true
last-updated: '2023-6-23'
---

I'm currently doing a deep dive into Docker, and while installing it on my Linux distro ([EndeavourOS](https://endeavouros.com/)), I came across the following error after trying to execute `docker run hello-world`:

```
$ systemctl status docker
docker.service: Start request repeated too quickly.
docker.service: Failed with result 'exit-code'.
Failed to start Docker Application Container Engine.
```

This was a surprise, as I've installed Docker multiple times before, and this was the first time I came across this error. After a little bit of research, I found a hint:

```
$ journalctl -xeu docker.service
failed to start daemon: Error initializing network controller: Error creating default "bridge" network: Failed to program NAT chain: INVALID_ZONE: docker
```

So, something was off with my network controller. Some more research led me to believe my VPN ([Mullvad](https://mullvad.net/en)) was the culprit, and sure, I could have probably uninstalled it to see if that would fix my issue, but I was determined to troubleshoot and fix this properly.

Some more digging led me to check out my firewall, and I found out what was going on:

```
$ systemctl status firewalld
ERROR: RUNNING_BUT_FAILED: Changing permanent configuration is not allowed while firewalled is in FAILED state. The permanent configuration must be fixed and then firewalld restarted. Try `firewall-offline-cmd --check-config`
# Omitted for brevity
ERROR: INVALID_ZONE: docker
```

Something was wrong with my Docker installation, so I did what any other person would've done in my place: run the command stated above.

```
# firewall-offline-cmd --check-config
ERROR: Failed to load user configuration. Falling back to full stock configuration.
Configuration error: INVALID_ZONE: not a valid zone file: no element found: line 1, column 0
```

We got it! There's a bad file out there, but where is it? After some back and forth, and with the help of the [firewalld](https://wiki.archlinux.org/title/firewalld) entry on the Arch Wiki, I found out the misbehaving file:

`$ cat /etc/firewalld/zones/docker.xml`

The file was empty, and that was the reason of this error. I `rm`'d it out of existence, and checked again my firewall's config:

`# firewall-offline-cmd --check-config`

No errors! Now, we can restart the process:

`# systemctl restart firewalld`

But surely we're still missing something, right? Yep.

The file we deleted is a configuration file for Docker containers to be able to connect to the Internet, so we need that.

How do we do it? By typing in the following:

```
# firewall-cmd --permanent --new-zone=docker
# firewall-cmd --permanent --zone=docker --change-interface=docker0
```

Every command stated above should output `success` after you execute it. If that's not the case, you'll need to troubleshoot it.

To persist the changes, just run the following:

```
# firewall-cmd --reload
# systemctl restart firewalld
```

Now, to get Docker finally up and running:

```
# systemctl restart docker
$ systemctl status docker
```

And just to really make sure Docker works and can connect to the Internet:

`$ docker run hello-world`

![Docker ran successfully](/2023-6-23-docker-run-success.png)

Now we can get back to learning Docker, thanks for reading! 🐋👋

## Update

The original version of this post defined a rich rule via `firewall-cmd`, which prevented Docker containers to connect to the Internet. This step has been removed to avoid confusion. It also adds additional steps explaining how to persist configuration changes in `firewalld`.
