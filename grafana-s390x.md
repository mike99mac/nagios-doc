
# Installing Grafana on zLinux

These steps are based on this Grafana document: 

```
https://grafana.com/docs/grafana/latest/setup-grafana/installation/redhat-rhel-fedora/
```

To install Grafana into a z/VM virtual machine, perform the following steps.

- Install AlmaLinux 9.4 onto a zLinux virtual machine.

```
head -2 /etc/os-release
NAME="AlmaLinux"
VERSION="9.4 (Seafoam Ocelot)"
```

- Install some co-requisite packages:

```
sudo dnf install git mlocate net-tools podman vim wget
```

- Get the Grafana GPG key:

```
wget -q -O gpg.key https://rpm.grafana.com/gpg.key
```

- Import the key:

```
sudo rpm --import gpg.key
```

- Create a new Yum repo:

```
cat /etc/yum.repos.d/grafana.repo
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

- Install Grafana:

```
time sudo dnf install grafana
...
Complete!

real    9m50.396s
...
```

- Reload systemd files:

```
sudo systemctl daemon-reload
```

- Set the Grafana service to start at boot time:

```
sudo systemctl enable grafana-server
```

- Open port 3000 in the firewall: 

```
sudo firewall-cmd --zone=public --permanent --add-port 3000/tcp
sudo firewall-cmd --reload
```

- Turn off SELinux:

```
cd /etc/selinux/
diff config config.orig
22c22
< SELINUX=disabled
---
> SELINUX=enforcing
```

- Reboot for the changes to take effect:

```
sudo reboot
```

When the server is back up, point a browser to port 3000. In this example it is ``http://172.17.16.55:3000/login``

You should see a Grafana login page.


## Install Node.js 

To install node.js, perform the following steps.

- Update the system. This could take some time.

```
time sudo dnf update
...
Complete!
real    19m42.972s
...
```

- List the versions of node.js available to install:

```
dnf module list nodejs
...
AlmaLinux 9 - AppStream
Name                 Stream               Profiles                                           Summary
nodejs               18                   common [d], development, minimal, s2i              Javascript runtime
nodejs               20                   common [d], development, minimal, s2i              Javascript runtime
```

- Install node.js version 20, the latest version avaiable at the time:

```
time sudo dnf module install nodejs:20
...
Complete!
real    2m14.029s
...
```

- Show the installed package:

``` 
dnf list nodejs
...
nodejs.s390x                               1:20.12.2-2.module_el9.4.0+100+71fc9528  
``` 

## Create a Grafana contaier

To create a container with s390x, perform the following steps:

- Create a ``Dockerfile``:

```
cat Dockerfile
```

```
# Use an appropriate base image for s390x
FROM docker.io/s390x/almalinux:9.4

# Set environment variables
ENV GRAFANA_VERSION 9.0.0
ENV ARCH s390x

# Install necessary packages
RUN dnf -y update && \
    dnf -y install wget tar gzip make go && \
    dnf clean all

# Download and untar Grafana
RUN wget https://github.com/grafana/grafana/archive/refs/tags/v11.1.0.tar.gz && \
    tar -zxvf v11.1.0.tar.gz && \
    rm v11.1.0.tar.gz

# Build Grafana
RUN cd grafana-11.1.0 && \
    make && \
    mv grafana-11.1.0 /usr/share/grafana && \
    ln -s /usr/share/grafana/bin/grafana-server /usr/sbin/grafana-server && \
    ln -s /usr/share/grafana/bin/grafana-cli /usr/sbin/grafana-cli

# Create Grafana user and directories
RUN useradd -r -s /sbin/nologin grafana && \
    mkdir -p /etc/grafana /var/lib/grafana /var/log/grafana && \
    chown -R grafana:grafana /etc/grafana /var/lib/grafana /var/log/grafana

# Expose Grafana port
EXPOSE 3000

# Define working directory
WORKDIR /usr/share/grafana

# Set the default command to run Grafana
CMD ["grafana-server", "--homepath=/usr/share/grafana", "--config=/etc/grafana/grafana.ini"]
```

