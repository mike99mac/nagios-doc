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

- Ubuntu Desktop does not come with an SSH server installed.  To install openSSH, run the following command:

```
sudo apt install -y openssh-server
```

The OpenSSH server should now be running, so you can work from an SSH session if you prefer.

## Install Apache
To install Apache and co-requisite packages, perform the following:
```
sudo apt install -y apache2 libssl-dev php php-cli gcc glibc glibc-common gd gd-devel net-snmp openssl-devel 
```

**NOTE:** A number of these packages do not exist in Ubuntu, but they are left in to test with RHEL.

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
sudo vi nagios.conf
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

- Enable the new web server:

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

- Build with ``make``:

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
```

- Install the Apache code:

```
make install-webconf
```

**NOTE:** If it fails, it may be due to the wrong value of the ``HTTPD_CONF`` variable: 

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

Compile and install the plugins using the following ``make`` commands:

```
make
make install
```

If all went well, the Nagios plugins are now built and installed.

### Update Apache to add Nagios 
Now that Nagios and the plugins are installed, the Apache configuration file can be updated to point to the Nagios home page and the ``cgi-bin/`` directory.

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

- Create a systemd service file in ``/etc/systemd/system`` with the following content:

``` 
cd /etc/systemd/system
vi nagios.service
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

## Test Nagios
To test Nagios, point a browser to your new site.  In this example, the URL is ``http://model1500/nagios``.

You should be challenged for the credentials in the ``/usr/local/nagios/etc/htpasswd.users`` file created earlier.

**TODO:** get a screen shot

# Add Nagios agents to servers to be monitored
The Nagios help pages are here: https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/4/en/config.html

https://www.nagios.org/ncpa/#downloads

Good reference:

https://github.com/NagiosEnterprises/ncpa/blob/master/BUILDING.rst#building-ncpa
