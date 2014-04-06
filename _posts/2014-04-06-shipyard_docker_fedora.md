---
layout: post
title: Shipyard and Docker on Fedora 20
---

[Shipyard](https://github.com/shipyard/shipyard) is a management interface for [Docker](http://www.docker.io/).
The [QuickStart instructions](https://github.com/shipyard/shipyard/wiki/QuickStart) require some minor modifications for Fedora 20.


## Docker configuration

    $ yum install docker-io
    $ systemctl start docker

`docker` requires admin privileges, and can either be run using `sudo`, or you can add yourself to the `docker` system group then logout and back in

    $ sudo usermod -a -G docker <username>

Docker listens on a local Unix socket by default, but Shipyard requires a TCP connection to Docker.
Fedora uses [systemd](https://fedoraproject.org/wiki/Systemd) instead of SysV init.d scripts.
The Docker systemd configuration can be overridden by copying the system file into `/etc/systemd/system/` (files in here will automatically override the provided files) and modifying the `ExecStart` line

    $ cp /usr/lib/systemd/system/docker.service /etc/systemd/system/

Open `/etc/systemd/system/docker.service` in a text editor, change

    ExecStart=/usr/bin/docker -d

to

    ExecStart=/usr/bin/docker -d -H tcp://127.0.0.1:4243 -H unix:///var/run/docker.sock

and restart the Docker service

    $ sudo systemctl restart docker


## Shipyard installation

Next install Shipyard using the `shipyard/deploy` image, which automatically installs the Shipyard components

    $ docker run -i -t -v /var/run/docker.sock:/docker.sock shipyard/deploy setup

At this point Shipyard should be running: [http://localhost:8000/](http://localhost:8000/)

Next install, register and run the [Shipyard Agent](https://github.com/shipyard/shipyard-agent) following the [instructions](https://github.com/shipyard/shipyard-agent/blob/master/readme.md).
This is required even if Shipyard is running on the same host.

If you download a [binary release](https://github.com/shipyard/shipyard-agent/releases) (and make it executable `chmod +x shipyard-agent`) you may see the following error when starting it

    $ ./shipyard-agent
    shipyard-agent: error while loading shared libraries: libdevmapper.so.1.02.1: cannot open shared object file: No such file or directory

There are a few solutions described [here](https://github.com/shipyard/shipyard/issues/134).
One way around this which doesn't involve messing with your system libraries

    $ mkdir ~/shipyard-agent
    $ cd ~/shipyard-agent
    $ ln -s /usr/lib64/libdevmapper.so.1.02 libdevmapper.so.1.02.1

    $ LD_LIBRARY_PATH=~/shipyard-agent shipyard-agent

In Shipyard add this host (click on `Hosts`), do not use `localhost` or `127.0.0.1` because this will refer to the container running Shipyard, instead of the host.


## Host network configuration

Shipyard requires network access to the host, but the host network created for Docker may be firewalled.
If you can login to Shipyard and see a list of containers and images (for instance all the Shipyard containers should be listed) but clicking on a container ID results in

    Server Error (500)

this suggests Shipyard is unable to connect to the agent.

Check the name of the network interface by running `ifconfig`, it will probably be called `docker0`.
Fedora 20 uses [FirewallD](https://fedoraproject.org/wiki/FirewallD), you can check which zone the network interface is in

    $ firewall-cmd --get-zone-of-interface=docker
    no zone

and add it to the trusted zone if necessary

    $ firewall-cmd --zone=trusted --add-interface=docker0

