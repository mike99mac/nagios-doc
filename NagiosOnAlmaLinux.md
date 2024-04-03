# Installing Apache and Nagios 
This document describes how to install Apache v2.4.52 and Nagios v4.5.1 on AlmaLinux v9.3 running on an x86_64 architecture virtual server. 


## Prepare the server 
To prepare for installations of Apache and Nagios, perform the following steps.

- Update the system:
```
yum check-update
yum update
```

## Install Apache
To install Apache and co-requisite packages, perform the following:

- Login as root:

```
id
uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

- Install Apache and associated packages:

```
yum install gcc glibc glibc-common wget unzip httpd httpd-tools php gd gd-devel openssh-devel perl postfix
```

- Start the ``httpd`` service now and at set it to start at boot time:

```
systemctl status httpd
systemctl enable httpd
```

- Allow ``http`` and ``https`` traffic through the firewall:

```
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-service=https --permanent
```

- Create a sample HTML file.

```
cd /var/www/html
vi index.html
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

- Test your web server by pointing a browser to it.  In this example the URL is ``http://mmac01.devlab.sinenomine.net/index.html``.
You should see rendering of the HTML file you just created.  This means that Apache is running and serving pages.

## Prepare Nagios
To prepare to install Nagios, perform the following steps.

- Create the user ``nagios`` and set the password:

```
useradd -m -s /bin/bash nagios
passwd nagios
```

- Create new groups and add them to the ``nagios`` user:

```
groupadd nagios 
groupadd nagcmd
usermod -g nagios -G nagios,nagcmd nagios
```

- Add the group ``nagcmd`` to the user ``apache``: 

```
sudo usermod -a -G nagcmd apache 
```

- Show the updated user ids:

```
id nagios
uid=1000(nagios) gid=1000(nagios) groups=1000(nagios),1001(nagcmd)
id apache
uid=48(apache) gid=48(apache) groups=48(apache),1001(nagcmd)
```

## Build Nagios
To build Nagios, perform the following steps.

- Create the directory ``nagios`` in the nagios user's home directory:
```
cd /home/nagios
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

- Create the ``Makefile``. Set the architecture with the ``--build`` option. In this exampe ``x86_64`` is used because this was run on an 64-bit Intel server.

```
./configure --with-command-group=nagcmd --build=x86_64-unknown-linux-gnu 
```

- Build Nagios core with ``make all``:

```
make all
```

### Install Nagios
To install Nagios, perform the following steps.

- Install the code with the following ``make`` commands:

```
sudo make install
```

- Install the systemd ``nagios.service`` file and enable it to start at boot time:

```
make install-daemoninit
systemctl enable nagios
```

- Install Nagios command mode:

```
make install-commandmode
```

- Install sample configuration files:

```
make install-config
```

- Configure the web interface - replace the email address for the Web admin:

```
vi /usr/local/nagios/etc/objects/contacts.cfg
...
define contact {

    contact_name            nagiosadmin             ; Short name of user
    use                     generic-contact         ; Inherit default values from generic-contact template (defined above)
    alias                   Nagios Admin            ; Full name of user
    email                   mmacisaac@sinenomine.net
}
...
```

- Install the web server related code:

```
make install-webconf
```

- Create a password file for the user ``nagiosadmin``. These will set the credentials needed to access the site:

```
htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
New password:
Re-type new password:
Adding password for user nagiosadmin
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



