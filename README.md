# Full Stack ND P5: Linux Server Configuration
This is the 5th project of the FullStack Web Developer Nanodegree at Udacity.

The goal for this project was to take a baseline installation of a Linux distribution on a virtual machine and prepare it to host web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.

This server will host the P3 Item Catalog application.

#### IP Address:
The server is running at 52.37.52.10.

#### Live Demo URL:
You can visit the server at http://ec2-52-37-52-10.us-west-2.compute.amazonaws.com/.

#### Content:
* `README.md`: This file.

#### Resources used:
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps

## Steps done in order to setup the server:

* Creating user _grader_ and give _sudo_ permission:
```
adduser grader
adduser grader sudo
```

* Update all currently installed packages:
```
sudo apt-get update
sudo apt-get upgrade
```

* Generate ssh public-private key pair:
At local machine:
```
ssh-keygen
```
At remote server:
```
su - grader
mkdir .ssh
nano .ssh/authorized_keys
```
copy the content of public key into it
and change permissions:
```
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
service ssh restart
```
Logout and the login using:
```
ssh -i ~/.ssh/id_rsa grader@52.37.52.10 -p 2200
```

* Change the SSH port from 22 to 2200 and disable root login:
```
sudo nano /etc/ssh/sshd_config
```
Set: `Port 2200` and `PermitRootLogin no`
```
sudo service ssh restart
```

* Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123):
```
sudo ufw status
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw status
sudo ufw allow 2200/tcp
sudo ufw allow ntp
sudo ufw allow www
sudo ufw enable
```

* Configure the local timezone to UTC:
```
sudo dpkg-reconfigure tzdata
```

* Install and configure Apache to serve a Python mod_wsgi application:
```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi
sudo nano /etc/apache2/sites-enabled/000-default.conf
```
add the following line at the end of the `<VirtualHost *:80>` block, right before the closing `</VirtualHost>` line:
```
WSGIScriptAlias / /var/www/html/myapp.wsgi
```
Restart the apache service:
```
sudo apache2ctl restart
```

* Install and configure PostgreSQL:
```
sudo apt-get install postgresql
```
Check for remote connections:
```
sudo nano /etc/postgresql/9.3/main/pg_hba.conf
```

* Create a new user named catalog that has limited permissions to your catalog application database:
```
sudo su - postgres
CREATE DATABASE catalog;
CREATE USER catalog;
ALTER ROLE catalog WITH PASSWORD 'logcata';
GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
\q
exit
```

* Install git, clone and setup your Catalog App project:
```
sudo apt-get install git
cd /var/www
sudo mkdir itemCatalog
cd itemCatalog
sudo git clone https://github.com/marionuevo/fsnd-p3-itemCatalog.git
sudo mv fsnd-p3-itemCatalog/ itemCatalog/
cd itemCatalog
sudo apt-get -y install python-pip
sudo pip install virtualenv
sudo virtualenv venv
source venv/bin/activate
sudo pip install -r requirements.txt
sudo nano database_setup.py
```
Set: `engine = create_engine('postgresql://catalog:logcata@localhost/')`
```
sudo mv project.py __init__.py
sudo nano __init__.py
```
Set / update the following lines:
```
engine = create_engine('postgresql://catalog:logcata@localhost/')
```
```
APP_PATH = os.path.dirname(__file__)
```
```
open(APP_PATH + '/client_secrets.json', 'r').read())['web']['client_id']
```
```
oauth_flow = flow_from_clientsecrets(APP_PATH + '/client_secrets.json', scope='')
```
Install the following packages:
```
sudo pip install flask-seasurf
sudo apt-get -qqy install postgresql python-psycopg2
```

* Test for everything is working fine:
```
sudo python __init__.py
```
Exit from the virtual environment:
```
deactivate
```

* Enable the itemCatalog site at apache:
```
sudo nano /etc/apache2/sites-available/itemCatalog.conf
```
Set this file as:
```
<VirtualHost *:80>
    ServerName ec2-52-37-52-10.us-west-2.compute.amazonaws.com
    ServerAdmin admin@mywebsite.com
    WSGIScriptAlias / /var/www/itemCatalog/itemCatalog.wsgi
    <Directory /var/www/itemCatalog/itemCatalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/itemCatalog/itemCatalog/static
    <Directory /var/www/itemCatalog/itemCatalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
```
sudo a2ensite itemCatalog
service apache2 reload
cd /var/www/
sudo nano itemCatalog.wsgi
```
Set this files as:
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/itemCatalog/")
from itemCatalog import app as application
application.secret_key = 'super_secret_key'
```
