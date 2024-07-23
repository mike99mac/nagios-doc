# Using ngingx and gunicorn with flask

This page is based on this: ``https://www.howtoforge.com/how-to-install-flask-with-nginx-and-gunicorn-on-rocky-linux/``

## Installing nginx gunicorn and flask
To install ngingx gunicorn and flask on AlmaLinux, perform the following steps:

- Login as root or switch user to it:

```
sudo su -
```
 
- Install the Extra Packages for Enterprise Linux tools:

```
dnf install epel-release
```

- Install co-requisite packages:

```
dnf install cifs-utils git gcc mlocate net-tools python3 python3-pip 
```

## Install ovos-tools
This repo has a number of helpful small tools that will be installed in ``/usr/local/sbin``. 
- Clone the ``ovos-tools`` repo:

```
git clone https://github.com/mike99mac/ovos-tools
```

- Install the tools 

```
ovos-tools/setup.sh
```

## Install start and enable nginx
- Install nginx:

```
dnf install nginx 
```

- Start nginx:

```
systemctl start nginx
```

- Set nginx to start at boot time

```
systemctl enable nginx
```

## Create a non-root user
A non-root user is recommended.

- Create a non-root group: 

```
groupadd -g 1009 mikemac
```

- Create a non-root user:

```
useradd -g 1009 -G wheel -u 1009 mikemac
```

- Change the home directory permission bits:

```
chmod 755 /home/mikemac
```

- Switch to that user:

```
su - mikemac
```

## Install flask
- Make a directory for flask:

```
sudo mkdir -p /srv/www/flask
```

- Set the owner and group to nginx:

```
sudo chown nginx:nginx /srv/www/flask
```

- Set the permission bits to 755:

```
sudo chmod 755 /srv/www/flask
```

## Create a python 3.10 environment
AlmaLinux 9.4 has Python 3.9 installed.  Python 3.10 is required for ``match`` statements.  

To install Python 3.10, perform the following steps.
python -V
- Update the system:

```
sudo dnf update
```

- Install co-requisite packages:
```
sudo dnf install wget yum-utils make gcc openssl-devel bzip2-devel libffi-devel zlib-devel
```

- Download the Python 3.10 tar file to your home directory:

```
cd; 
wget https://www.python.org/ftp/python/3.10.5/Python-3.10.5.tgz
```

- Untar the file:

```
tar xzf Python-3.10.5.tgz
```

- Change to the new directory:

```
cd Python-3.10.5
```

- Configure the package:

```
./configure --with-system-ffi --with-computed-gotos --enable-loadable-sqlite-extensions
```

- Build the package:

```
make -j ${nproc}
```

- Install the package:

```
sudo make altinstall
```

- Verify the installation

```
python3.10 -V
Python 3.10.5
```

## Create a virtual environment
Perform the following steps to create a Python 3.10 virtual environment.

- Change directory to ``/srv/``:

```
cd /srv/
```

- Create a Python 3.10 virtual environment:

```
sudo python3.10 -m venv venv
```

- Will this work?

## Change ownership and permissions
The ownership and permissions of the virtual environment must be modified so the user ``nginx`` can write to it. To do so, perform the following steps.

- This should work
```
sudo chgrp nginx /srv; 
sudo chmod g+w /srv; 
sudo chgrp -R nginx /srv/venv/; 
sudo chmod -R g+w /srv/venv/
```

- Activate the virtual environment:

```
. /srv/venv/bin/activate
```
  
- Upgrade pip:

```
sudo python3 -m pip install --upgrade pip
```

- Install flask and gunicorn:

```
pip3 install flask gunicorn
```

- Create the file ``/srv/www/myflask.py``:

```
# myflask.py
from flask import Flask, render_template  # importing the render_template function

app = Flask(__name__)
# route to index page
@app.route("/")
def hello():
    return render_template('index.html')
```

- Create the file ``/srv/www/flask/wsgi.py``: 

```
if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

- Create the file /srv/www/flask/templates/index.html:

```
<html>
    <body>
        <h1><center>Hello Flask - Rocky Linux!</center></h1>
    </body>
</html>

```

- Start flask:

```
python3 myflask.py

```
- Create the file ``/srv/www/flask/wsgi.py``:

```
# import myflask Flask application
from myflask import app

if __name__ == "__main__":
    app.run(debug=True)
```

- Open a browser and point it to **WHERE**?
 
