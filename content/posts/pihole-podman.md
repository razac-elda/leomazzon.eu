---
title: "Pi-hole container with Podman"
summary: "Podman mascot is a mole, so it make sense to use Pi-hole with it."
description: "Podman mascot is a mole, so it make sense to use Pi-hole with it."
date: 2022-05-15T20:25:15+02:00
draft: false
---

![logo](images/pihole_podman.png#center)

One piece of software that I always use on my local server is [Pi-hole](https://pi-hole.net/ "Pi-hole website"), it's just too good to not have it running. I had to reinstall Pi-hole so I was thinking of trying [Podman](https://podman.io/ "Podman website") instead of [Docker](https://www.docker.com/ "Docker website") because it's daemonless, rootless and it's nice to have alternatives.

In this guide I will create the container as root because Pi-hole needs to use privileged ports(<1024) which by default are protected from non-root users, you can [lower the unprivileged port range](https://stackoverflow.com/questions/413807/is-there-a-way-for-non-root-processes-to-bind-to-privileged-ports-on-linux) if you want.

I'm running **Ubuntu Server 22.04 LTS** on a Raspberry Pi 3b.

First things first, install **Podman** and pull **Pi-hole image**.

```console
sudo apt install podman
podman pull docker.io/pihole/pihole
```

Next we will need to edit **systemd-resolved** config because it uses port 53 for it's DNS stub resolver which will conflict with Pi-hole.

```console
sudo systemctl stop systemd-resolved
sudo nano /etc/systemd/resolved.conf #Sorry vim users
```

Add/uncomment the following line and change it's value to "no".

```text
DNSStubListener=no
```

Start back the service and check that it is running correctly.

```console
sudo systemctl start systemd-resolved
sudo systemctl status systemd-resolved
```

This is the main command to create and start the container, **remember to change** the *hostname*, *timezone* and *password*. To update the container simply delete it and create a new one from the latest image, all configurations will be saved within Podman's volumes "pihole" and "dnsmasq".

```console
sudo podman run -d \
    --name=pihole \
    --hostname=YOUR_HOSTNAME \
    -e TZ=YOUR_TIMEZONE \
    -e WEBPASSWORD=YOUR_PASSWORD \
    -e SERVERIP=127.0.0.1 \
    -v pihole:/etc/pihole \
    -v dnsmasq:/etc/dnsmasq.d \
    -p 53:53/tcp \
    -p 53:53/udp \
    -p 80:80 \
    --restart=always \
    pihole/pihole
```

Since Podman is deamonless we have to find a way to start the container after a reboot. Fortunately, we can do this with **systemd** and Podman will provide us the correct configuration file.

```console
sudo podman generate systemd --new --name pihole > pihole.service
sudo mv pihole.service /etc/systemd/system
```

To finish reload **systemd** and start **pihole.service**.

```console
sudo systemctl deamon-reload
sudo systemctl enable pihole.service
sudo systemctl start pihole.service
sudo systemctl status pihole.service
```

Now try to connect to the web interface at *device_ip/admin*. If everything works correctly you should see Pi-hole dashboard.
