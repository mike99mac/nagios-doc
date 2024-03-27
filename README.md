# Installing Apache and Nagios 
This documents installing Apache v2.4.52 and Debian v4.5.1 on Ubuntu Desktop v22.04 on a Raspberry Pi 4.

## Prepare the server 
To prepare for installations of Apache and Nagios, perform the following steps

- Install Ubuntu Desktop 22.04 on the platform of your choice.  
- Open a terminal session.
- Refresh the system.

```
$ sudo apt update
...
$ sudo apt upgrade -y
...
```

- Ubuntu Desktop does not come with an SSH server installed.  To install it, run the following command.

```
$ sudo apt install -y openssh-server libssl-dev 
...
```

Now you can start an SSH session if you prefer.

## Install Apache
```
$ sudo apt install -y apache2
...
```

The directory ``/srv/`` is a better choice for Web server data than ``/var/`` per the *Linux Filesystem Hierarchy* standard. 
- Create new directories
```
$ cd /srv
$ sudo mkdir -p www/html www/cgi

- Create a sample HTML file.

```
$ sudo vi index.html
<html>
<head>
  <title> Nagios server </title>
</head>
<body>
  <p> Nagios will be running here soon!</p>
</body>
</html>

- Create a new Apache configuration file

```
$ cd /etc/apache2/sites-available
$ sudo cp 000-default.conf nagios.conf
```

- Replace the contents of the file with these settings:

```
$ sudo vi nagios.conf
<VirtualHost *:80>
  ServerAdmin mike99mac@gmail.com
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

- Enable the new web server

```
$ sudo a2ensite nagios.conf
...
$ sudo systemctl reload apache2
```

## Install Nagios
To install Nagios, perform the following steps.

- Create users and groups

```
$ sudo /usr/sbin/useradd -m -s /bin/bash nagios
$ sudo passwd nagios
...
$ sudo groupadd nagios
$ sudo usermod -G nagios nagios
$ sudo groupadd nagcmd
$ sudo usermod -a -G nagcmd nagios
$ sudo usermod -a -G nagcmd www-data
```

- Switch to the new ``nagios`` user and run the ``**id**`` command.

```
$ sudo su - nagios
$ id
uid=1001(nagios) gid=1001(nagios) groups=1001(nagios),1002(nagcmd)
```

## Build Nagios
To build Nagios, perform the following steps.

- Download the Nagios core and plugins code.

```
# cd
# mkdir nagios
# cd nagios/
# wget https://github.com/NagiosEnterprises/nagioscore/releases/download/nagios-4.5.1/nagios-4.5.1.tar.gz
# wget https://github.com/nagios-plugins/nagios-plugins/releases/download/release-2.4.9/nagios-plugins-2.4.9.tar.gz


## Build the Nagios core
To build the Nagios core, perform the following steps.

- Untar the code and change to that directory.

```
# tar xvf nagios-4.5.1.tar.gz
# cd nagios-4.5.1
```

Build the code

Create the Makefile
# ./configure --with-command-group=nagcmd --build=aarch64-unknown-linux-gnu
...
Creating sample config files in sample-config/ ...
*** Configuration summary for nagios 4.2.1 09-06-2016 ***:
 General Options:
 -------------------------
        Nagios executable:  nagios
        Nagios user/group:  nagios,nagios
       Command user/group:  nagios,nagcmd
             Event Broker:  yes
        Install ${prefix}:  /usr/local/nagios
    Install ${includedir}:  /usr/local/nagios/include/nagios
                Lock file:  ${prefix}/var/nagios.lock
   Check result directory:  ${prefix}/var/spool/checkresults
           Init directory:  /etc/init.d
  Apache conf.d directory:  /etc/httpd/conf.d
             Mail program:  /bin/mail
                  Host OS:  linux-gnu
          IOBroker Method:  epoll
                 HTML URL:  http://localhost/nagios/
                  CGI URL:  http://localhost/nagios/cgi-bin/

Build with make
# time make all
...
real    2m33.240s

Install the code
# make install
# make install-init
# make install-config
# make install-commandmode

Configure the web interface
Add an email address for the Web admin:
vi /usr/local/nagios/etc/objects/contacts.cfg

# make install-webconf
Failed - pointing to wrong apache /etc directory - had to change the Makefile:

# vi Makefile
# HTTPD_CONF=/etc/httpd/conf.d
HTTPD_CONF=/etc/apache2/conf-available

Create password file
# htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
New password:
Re-type new password:
Adding password for user nagiosadmin

Credentials: nagiosadmin/nagios

Compile the plugins
# cd /home/pi/nagios/
# tar xzf nagios-plugins-2.4.9.tar.gz
# cd nagios-plugins-2.4.9



Created a systemd service file
# cat nagios.service
[Unit]
Description=Nagios Core
Documentation=https://www.nagios.org/documentation
After=network.target local-fs.target
BindTo=network.target

[Install]
WantedBy=multi-user.target
[Service]
Type=forking
# Type=simple
User=nagios
Group=nagios
PIDFile=/run/nagios.pid
ExecStartPre=/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
ExecStart=/usr/local/nagios/bin/nagios -d /usr/local/nagios/etc/nagios.cfg
ExecStop=/usr/bin/kill -s TERM ${MAINPID}
ExecStopPost=/usr/bin/rm -f /usr/local/nagios/var/rw/nagios.cmd
ExecReload=/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
ExecReload=/usr/bin/kill -s HUP ${MAINPID}


