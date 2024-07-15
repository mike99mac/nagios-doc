# Installing Grafana

**NOTE:**

This is a document focused on Nagios, but we are moving away from Nagios, so this is just a place to drop some quick notes.


his document is based on this Grafana document:  ``https://grafana.com/docs/grafana/latest/setup-grafana/installation/redhat-rhel-fedora/``

To install Grafana, perform the following steps.

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

- Turn off the firewall (for now):

```
sudo systemctl disable firewalld
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

- Reboot

```
sudo reboot
```

When the server is back up, point a browser to port 3000. In this example it is ``http://172.17.16.55:3000/login``


sudo iptables -I INPUT -m state --state NEW -p tcp --dport 3000 -j ACCEPT

# Installing Apache and Nagios 

This document describes how to install Apache v2.4.52 and Nagios v4.5.1 on Ubuntu Desktop v22.04 running on an ARM architecture Raspberry Pi 4.

It addresses *Nagios Core* vs *Nagios XI* which is built on top of Nagios Core, and is more GUI-driven than command line driven.  The author still likes the command line, so Nagios XI will not be addressed in this document.  Perhaps it will be at a later date.


## Prepare the server 
To prepare for installations of Apache and Nagios, perform the following steps.

- Install a Linux on the platform of your choice.  This document was written using Ubuntu Desktop 22.04
- Open a terminal session.
- Upgrade the system:

```
sudo apt update
sudo apt upgrade -y
```

- Ubuntu Desktop does not come with an SSH server installed.  To install openSSH, git and vim, run the following command:

```
sudo apt install -y openssh-server git vim
```

The OpenSSH server should now be running, so you can work from an SSH session if you prefer.

## Install Apache
To install Apache and co-requisite packages, perform the following:
```
sudo apt install -y apache2 libssl-dev php php-cli gcc 
```

The directory ``/srv/`` is a better choice for Web server data than ``/var/`` per the *Linux Filesystem Hierarchy* standard. 

- Create a new directory ``/srv/www/html`` for Web data:
```
cd /srv
sudo mkdir -p www/html 
```

- Create a sample HTML file.

```
cd www/html
sudo vi index.html
```

- Add the following contents:

```
<html>
<head>
  <title> Nagios server </title>
</head>
<body>
  <p> Nagios will be running here soon!</p>
</body>
</html>
```

- Create a new Apache configuration file:

```
cd /etc/apache2/sites-available
sudo cp 000-default.conf nagios.conf
```

- Replace the contents of the file with these settings:

```
<VirtualHost *:80>
  ServerAdmin admin@example.com 
  DocumentRoot /srv/www/html
  ServerName model1500

  <Directory "/srv/www/html">
    Options Indexes FollowSymLinks
    AllowOverride all
    Require all granted
  </Directory>

  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- Enable the new web server with the ``a2ensite`` command and restart Apache:

```
sudo a2ensite nagios.conf
sudo systemctl reload apache2
```

- Test your web server by pointing a browser to it.  In this example the URL is ``http://model1500/``.
You should see rendering of the HTML file you just created.

## Install Nagios
To install Nagios, perform the following steps.

- Create the user ``nagios`` and set the password:

```
sudo useradd -m -s /bin/bash nagios
sudo passwd nagios
```

- Create new groups and add them to the ``nagios`` user:

```
sudo groupadd nagios nagcmd
sudo usermod -g nagios -G nagios,nagcmd,sudo nagios
```

- Add the group ``nagcmd`` to the user ``www-data``: 

```
sudo usermod -a -G nagcmd www-data
```

- Switch to the new ``nagios`` user: 

```
sudo su - nagios
```

- Run the ``id`` command and verify the groups:

```
id
uid=1001(nagios) gid=1001(nagios) groups=1001(nagios),1002(nagcmd)
```

## Build Nagios
To build Nagios, perform the following steps.

- Create the directory ``nagios`` in the home directory:
```
cd
mkdir nagios
cd nagios/
```

- Download the Nagios core and plugins code:

```
wget https://github.com/NagiosEnterprises/nagioscore/releases/download/nagios-4.5.1/nagios-4.5.1.tar.gz
wget https://github.com/nagios-plugins/nagios-plugins/releases/download/release-2.4.9/nagios-plugins-2.4.9.tar.gz
```

### Build the Nagios core
To build the Nagios core, perform the following steps.

- Untar the code and change to that directory:

```
tar xvf nagios-4.5.1.tar.gz
cd nagios-4.5.1
```

- Create the ``Makefile``. Set the architecture with the ``--build`` option. In this exampe ``aarch64`` is used because this was run and an ARM Raspberry Pi.

**NOTE:** the location of the ssl libraries specified by ``--with-ssl-lib`` will depend on architecture.

```
./configure --with-command-group=nagcmd --build=aarch64-unknown-linux-gnu --with-ssl-lib=/usr/lib/aarch64-linux-gnu
```

- Build Nagios core with ``make``:

```
make all
```

- Install the code with the following ``make`` commands:
```
sudo make install
sudo make install-init
sudo make install-config
sudo make install-commandmode
```

- Configure the web interface - replace the email address for the Web admin:

```
sudo vi /usr/local/nagios/etc/objects/contacts.cfg
...
define contact {

    contact_name            nagiosadmin             ; Short name of user
    use                     generic-contact         ; Inherit default values from generic-contact template (defined above)
    alias                   Nagios Admin            ; Full name of user
    email                   your-email@example.com
}
...
```

- Install the web server related code:

```
make install-webconf
```

**NOTE:** If it fails, check the value of the ``HTTPD_CONF`` variable. In this exampe it was pointing to ``/etc/httpd`` not ``/etc/apache2``:

```
# vi Makefile
...
# HTTPD_CONF=/etc/httpd/conf.d
HTTPD_CONF=/etc/apache2/conf-available
...
```

- Create a password file for the user ``nagiosadmin``. These will set the credentials needed to access the site:

```
htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
```

Remember the credentials!

### Build Nagios plugins
To build and install the Nagios plugins, perform the following steps.

- Untar the plugins code:

```
cd /home/pi/nagios/
tar xzf nagios-plugins-2.4.9.tar.gz
cd nagios-plugins-2.4.9
```

- Run ``configure`` to create a ``Makefile``: 

```
./configure --with-nagios-user=nagios --with-nagios-group=nagios
```

- Compile and install the plugins using the following ``make`` commands:

```
make
make install
```

If all went well, the Nagios plugins are now built and installed.

### Update Apache to add Nagios 
Now that Nagios and the plugins are installed, the Apache configuration file can be 
updated to point to the Nagios home page and the ``cgi-bin/`` directory.
which is actually the ``"/usr/local/nagios/sbin"`` directory.

- Edit the file:

```
cd /etc/apache2/sites-available
vi nagios.conf
```

- Replace it with the following content:

```
<VirtualHost *:80>
  ServerAdmin admin@example.com
  DocumentRoot /srv/www/html
  ServerName model1500

  <Directory "/srv/www/html">
    Options Indexes FollowSymLinks
    AllowOverride all
    Require all granted
  </Directory>

  ScriptAlias /nagios/cgi-bin "/usr/local/nagios/sbin"
  <Directory "/usr/local/nagios/sbin">
    Options ExecCGI
    AllowOverride None
    AuthName "Nagios Access"
    AuthType Basic
    AuthUserFile /usr/local/nagios/etc/htpasswd.users
    Require valid-user
</Directory>

  Alias /nagios "/usr/local/nagios/share"
  <Directory "/usr/local/nagios/share">
    Options None
    AllowOverride None
    AuthName "Nagios Access"
    AuthType Basic
    AuthUserFile /usr/local/nagios/etc/htpasswd.users
    Require valid-user
  </Directory>

  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

### Enable Nagios to start at boot time
To eable Nagios to start at boot time, perform the following steps.

- Create a systemd service file in ``/etc/systemd/system``: 

``` 
cd /etc/systemd/system
vi nagios.service
```

- Add the following content:

```
[Unit]
Description=Nagios
BindTo=network.target
[Install]
WantedBy=multi-user.target
[Service]
Type=simple
User=nagios
Group=nagios
ExecStart=/usr/local/nagios/bin/nagios /usr/local/nagios/etc/nagios.cfg
ExecStop=/usr/bin/kill -s TERM ${MAINPID}
ExecStopPost=/usr/bin/rm -f /usr/local/nagios/var/rw/nagios.cmd
ExecReload=/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
ExecReload=/usr/bin/kill -s HUP ${MAINPID}
```

- Reload the systemd unit files:
 
```
sudo systemctl daemon-reload
```

- Set the Nagios service to start at boot time and for the current session:

```
systemctl enable nagios
systemctl start nagios
```

## Test Nagios

- Check the status of Nagios.  It should be running:

```
systemctl status nagios
● nagios.service - Nagios
     Loaded: loaded (/etc/systemd/system/nagios.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2024-04-03 09:41:58 EDT; 8s ago
     ...
```

To test Nagios, point a browser to your new site.  In this example, the URL is ``http://model1500/nagios``.

You should be challenged for the credentials in the ``/usr/local/nagios/etc/htpasswd.users`` file created earlier.

**TODO:** get a screen shot

# Add Nagios agents to servers to be monitored
The Nagios help pages are here: https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/4/en/config.html

https://www.nagios.org/ncpa/#downloads

A good reference for building and installing ``ncpa`` is here: https://github.com/NagiosEnterprises/ncpa/blob/master/BUILDING.rst#building-ncpa

## Building the ncpa agent
To build and install ``ncpa``, perform the following steps.

- Clone the code from ``github``:

```
git clone https://github.com/NagiosEnterprises/ncpa
```

- Change to the ``ncpa/build`` directory and run ``build.sh`` to build the agent:
This step can take more than 20 minutes:

```
cd ncpa/build
./build.sh
```

- Show the resulting ``.deb`` file:

```
file *.deb
ncpa_3.0.2-1_arm64.deb: Debian binary package (format 2.0), with control.tar.zs, data compression zst
```

- Install the new package:

```
sudo dpkg -i ncpa_3.0.2-1_arm64.deb
```



