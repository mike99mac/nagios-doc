# Using ngingx and gunicorn with flask

This page is based on this: ``https://www.howtoforge.com/how-to-install-flask-with-nginx-and-gunicorn-on-rocky-linux/``

## Installing ngingx gunicorn and flask
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
dnf install python3-pip python3-devel gcc
```

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

- Make a directory for flask:

```
mkdir -p /srv/www/flask
```

- Set the owner and group to nginx:

```
sudo chown nginx:nginx /srv/www/flask
```

```
sudo chmod 755 /srv/www/flask
```

- Create a virtual environment in ``/srv/venv``:

```
cd /srv/
```

```
python3 -m venv myenv
```
  
-Upgrade pip:

```
/srv/www/flask/venv/bin/python3 -m pip install --upgrade pip
```

- Install flask and gunicorn:

```
   16  pip3 install flask gunicorn
```

- Create the file myflask.py:

```
$ cat myflask.py
# myflask.py
from flask import Flask, render_template  # importing the render_template function

app = Flask(__name__)
# route to index page
@app.route("/")
def hello():
    return render_template('index.html')

- Create the myflask.py

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

- Create the file templates/index.html:

```
$ cat templates/index.html
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
- Open a browser

- Create the file wsgi.py:

```
$ cat wsgi.py
# import myflask Flask application
from myflask import app

if __name__ == "__main__":
    app.run(debug=True)
```