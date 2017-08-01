# Udacity Linux Server Configuration Project
The Udacity Linux Server Configuration project steps are presented  below.
I took this approach because the Item Catalog project
in the third module is tightly linked with the Linux Server Configuration project
in module five.  Finally, I am only hosting one app on the server for now, but my plan
this project is on ec2 instance

Notes for reviewer:
* Public IP: `http://54.190.10.86/`
* SSH PORT: `2200`
* Catalog App URL using DNS domain: http://ec2-54-190-10-86.us-west-2.compute.amazonaws.com/
* Grader Passphrase for SSH login: 2680259
* 
### Updating server software, securing the server with UFW firewall and setting the timezone
Here we update the server software and configure the UFW firewall to make sure our server is secure.
1. Update all currently installed packages:
  - Find updates: `sudo apt-get update`
  - Install updates: `sudo apt-get upgrade` Hit Y for yes and give yourself a break while it installs.
2. Configure the Uncomplicated Firewall (UFW) to only allow  incoming connections for SSH (port 2200),
HTTP (port 80), and NTP (port 123):
  - Check UFW status to make sure its inactive: `sudo ufw status`
  - Deny all incoming by default: `sudo ufw default deny incoming`
  - Allow outgoing by default: `sudo ufw default allow outgoing`
  - Allow SSH: `sudo ufw allow ssh`
  - Allow SSH on port 2200: `sudo ufw allow 2200/tcp`
  - Allow HTTP on port 80: `sudo ufw allow 80/tcp`
  - Allow NTP on port 123: `sudo ufw allow 123/udp`
  - Turn on firewall: `sudo ufw enable`
3. Make sure the timezone is set to UTC:
  - Ceck that the timezone is set to UTC: `sudo dpkg-reconfigure tzdata`
  - It should default to UTC.

### Creating and securing the grader account with SSH and sudo access
There are several ways that this could have been done.  Here I tried to
follow the class instructions as closely as possible.
1. Create a new user named grader:
  - Enter: `sudo adduser grader`
  - Give grader a password of: `2680259`
  - Install finger: `apt-get install finger`
  - Check grader account with finger: `finger grader`
2. Give the grader the permission to sudo:
  - Create a sudoers.d file for grader: `sudo nano /etc/suoders.d/grader`
  - Inside the file add: `grader ALL=(ALL) NOPASSWD:ALL`
  - Save file: (nano: `ctrl+x`, `Y`, Enter)
  - This gives grader the same level of admin access as the Ubuntu account which is created
    when the cloud instance of Ubuntu is built.
3. Create SSH keys and copy the public key to the server manually:
  - On your local machine generate SSH key pair with: `ssh-keygen`
  - I added a passphrase for security: `2680259`
  - Save the public and private SSH keys in in the default SSH directory: `C:\Users\Mohamed\.ssh`  (in my case.)
  - My public and private key files are: `mo.pub` and `mo`
  - Login into grader account using password set during user creation: `ssh grader@54.190.10.86 -p 2200`
  - Make .ssh directory: `mkdir .ssh`
  - Make file to store public key: `touch .ssh/authorized_keys`
  - On your local machine read and copy contents of the public key: `cat ~/.ssh/mo.pub` then
    copy the key to  the clipboard.
  - Login to the grader account and paste contents of clipboard into the authorized_keys file:
    `nano /home/grader/.ssh/authorized_keys` then paste contents(ctr+v).
  - Save file: (nano: `ctrl+x`, `Y`, Enter)
  - Set permissions for the directory and file: `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
  - Change `PasswordAuthentication` from `yes` back to `no` in the sshd_config file:  `nano /etc/ssh/sshd_config`
  - Save file(nano: `ctrl+x`, `Y`, Enter)
  - Login with the corresponding private key: `ssh grader@54.190.10.86 -p 2200 -i ~/.ssh/mo`

### Setting up and configuring the Apache HTTP server and the Python wsgi interface
Here we install the Apache web server and make sure the default Apache web page can be
accessed on port 80.  The Apache and wsgi configurations are then added using these steps:
1. Install Apache web server: `sudo apt-get install apache2`
  - Check port 80 with the public IP address given above: `54.190.10.86`.
2. Install mod_wsgi: `sudo apt-get install libapache2-mod-wsgi`
  - Configure Apache to handle requests using the WSGI module: `sudo nano /etc/apache2/sites-enabled/000-default.conf`
  - Add: `WSGIScriptAlias / /var/www/html/myapp.wsgi` before `</VirtualHost>` closing line
  - Save file: (nano: `ctrl+x`, `Y`, Enter)
  - Restart Apache: `sudo apache2ctl restart`
3. Install python-dev package: `sudo apt-get install python-dev`
4. Verify wsgi is enabled: `sudo a2enmod wsgi`

### Setting up PostgreSQL server
Next the PostgreSQL is installed and configured.  This is required for the port of
the catalog database from SQLite to PostgreSQL.
1. The following steps get the catalog database up and running:
  - `sudo apt-get install libpq-dev python-dev`
  - `sudo apt-get install postgresql postgresql-contrib`
  - `sudo su - postgres`
  - `psql`
  - `CREATE USER catalog WITH PASSWORD 'password';`
  - `ALTER USER catalog CREATEDB;`
  - `CREATE DATABASE catalog WITH OWNER catalog;`
  - `\c catalog`
  - `REVOKE ALL ON SCHEMA public FROM public;`
  - `GRANT ALL ON SCHEMA public TO catalog;`
  - `\q`
  - `exit`
2. After you have installed the catalog app below, make sure that you:
  - Change create engine line in your `__init__.py` and `models.py` to:
    `engine = create_engine('postgresql://catalog:password@localhost/catalog')`
  - Make sure no remote connections to the database are allowed: `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`
    Should produce output that looks like this:
```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

###  Installing the Catalog app code and preparing it for deployment
Here we clone the catalog app from GitHub, install it on the Lightsail Ubuntu
instance and configure it for production.
1. Install git using: `sudo apt-get install git`
  - `cd /var/www`
  - `sudo mkdir catalog`
  - Change owner of the newly created catalog folder `sudo chown -R grader:grader catalog`
  - `cd /catalog`
2. Clone your project from github `git clone https://github.com/MoKhaled3003/MoRestaurants/`
3. Create a catalog.wsgi file, then place it in the /var/www/catalog directory.
  - Run this: `sudo nano /var/www/catalog/catalog.wsgi`
  - Paste the code below and save the file (nano: `ctrl+x`, `Y`, Enter):
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'lees_secret_key'
```
4. Go to the /var/www/catalog/catalog directory and rename the project.py file:
  - `cd /var/www/catalog/catalog`
  - `mv project.py __init__.py`
5. Install application dependencies and APIs from requirements.txt:
  - `sudo pip install -r /var/www/catalog/catalog/requirements.txt`
  - It will install the following modules if they are not already there:
  ```
  bleach==1.4.2
  dicttoxml==1.6.6
  dominate==2.1.16
  Flask==0.10.1
  Flask-Bootstrap==3.3.5.7
  Flask-SQLAlchemy==2.1
  Flask-WTF==0.12
  html5lib==0.9999999
  httplib2==0.9.2
  itsdangerous==0.24
  Jinja2==2.8
  MarkupSafe==0.23
  oauth2client==1.5.2
  psycopg2==2.6.1
  pyasn1==0.1.9
  pyasn1-modules==0.0.8
  rsa==3.2.3
  six==1.10.0
  SQLAlchemy==1.0.9
  visitor==0.1.2
  Werkzeug==0.11.2
  wheel==0.24.0
  WTForms==2.0.2
  ```

6. Make sure the Apache server is configured so that that the Catalog app is accessible on port 80:
  - Run this: `sudo nano /etc/apache2/sites-available/catalog.conf`
  - Paste the code below and save the file (nano: `ctrl+x`, `Y`, Enter):
  ```
  <VirtualHost *:80>
  ServerName 54.190.10.86
  ServerAlias ec2-54-190-10-86.us-west-2.compute.amazonaws.com
  ServerAdmin ljfarre@att.net
  WSGIScriptAlias / /var/www/catalog/catalog.wsgi
  <Directory /var/www/catalog/catalog/>
  Order allow,deny
  Allow from all
  </Directory>
  Alias /static /var/www/catalog/catalog/static
  <Directory /var/www/catalog/catalog/static/>
  Order allow,deny
  Allow from all
  </Directory>
  ErrorLog ${APACHE_LOG_DIR}/error.log
  LogLevel warn
  CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>
  ```
7. As stated above, tune the Python code to work with PostgreSQL instead of SQLite:
  - Change create engine line in your `__init__.py` and `models.py` to:
  `engine = create_engine('postgresql://catalog:password@localhost/catalog')`
  -Run to set up database initially: `python /var/www/catalog/catalog/models.py`

8. Make sure that Oauth2 is configured to work with the new URIs:
  - Go to the project on the Developer Console: https://console.developers.google.com/project
  - Navigate to APIs & auth > Credentials > Edit Settings
  - Add host name and public IP-address to your Authorized JavaScript origins and your
    host name + oauth2callback to Authorized redirect URIs, e.g.  http://ec2-54-190-10-86.us-west-2.compute.amazonaws.com/
    and `54.190.10.86`
  - Very Important:  Edit `__init__.py` and make sure that the path to the client_secrets.json
    file is fully elaborated, e.g., change all instances to /var/www/catalog/catalog/google_client_secrets.json.

### Running the catalog application
Here we reboot the server and run the application:
  -  command line reboot your server to make sure all dependencies
    are running properly and that server memory and cache are cleared.
  - Then restart apache: `sudo service apache2 restart`
  - The following URL should now take you to Lee's Classic Car Catalog: 
  -  http://ec2-54-190-10-86.us-west-2.compute.amazonaws.com/

### References
This README file was created from class notes, the project rubric requirements, several good examples in
GitHub and the links below:
* [Changing the timezone](https://help.ubuntu.com/community/UbuntuTime#Using_the_Command_Line_.28terminal.29)
* [Configuring Postgresql](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
* [Configuring a flask application with Apache2 and mod-wsgi](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
* [Monitoring apache2 with mod_status](http://www.tecmint.com/monitor-apache-web-server-load-and-page-statistics/)
