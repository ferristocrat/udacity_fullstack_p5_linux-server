# Udacity Server Configuration Project
This file outlines the steps needed to configure an Ubuntu server to serve a Flask application with a Python database. These steps are specific to Project 5 of Udacity's Fullstack Web Developer Nanodegree.

## Accessing the Server
IP Address: `52.26.243.255`
SSH Port: `2200`

## Viewing the Web Application
URL: `http://52.26.243.255`

## Steps to Configure Server and Install Software

### Setup User
1. Initially ssh in as root.
2. Create a new user: `adduser <USERNAME>`
3. Add new user to the sudo group: `gpasswd -a <USERNAME> sudo`
4. Add public key to user's account so we can ssh into server as the user
  - in user's home, run: `mkdir .ssh` or `mkdir /home/<USERNAME>/.ssh`
  - copy `.ssh/authorized_keys` from root's home to user's home like so: `cp -r .ssh/authorized_keys /home/grader/.ssh`
  - change owner and group of the new .ssh folder and authorized_keys file to user
    `chgrp grader /home/grader/.ssh`
    `chown grader /home/grader/.ssh`
    `chgrp grader /home/grader/.ssh/authorized_keys`
    `chown grader /home/grader/.ssh/authorized_keys`
  - change folder permissions of .ssh folder: `chmod 700 /home/grader/.ssh`
  - change file permission on authorized_keys: `chmod 644 /home/grader/.ssh/authorized_keys`

### Secure Server
1. Edit `/etc/ssh/sshd_config`
  - Disable password based logins by making sure `Password Authentication` is set to no.
  - Disable remote root login by setting `PermitRootLogin` to no.
  - Set ssh `Port` to `2200`
2. Restart ssh: `sudo service ssh restart`
3. Configure firewall
  - By default, deny all incoming requests: `sudo ufw default deny incoming`
  - Allow all outgoing requests: `sudo ufw default allow outgoing`
  - Allow ssh: `sudo ufw allow 2200/tcp`
  - Allow http: `sudo ufw allow 80/tcp`
  - Allow NTP: `sudo ufw allow ntp`
  - Enable firewall: `sudo ufw enable`
  - Check firewall status: `sudo ufw status`

### Install Software
1. First, update currently installed packages:
  - `sudo apt-get update`
  - `sudo apt-get upgrade`
2. Install packages using `sudo apt-get install`:
  - ntp daemon: `ntp`
  - apache web server: `apache2`
  - mod-wsgi: `libapache2-mod-wsgi`
  - PostgreSQL: `postgresql`
  - Git revision control: `git`
  - Python package handler: `pip`
  - Fail2Ban authentication locking: `fail2ban`
  - Development libraries: `libpq-dev python-dev`
  - Automatic updates: `unattended-upgrades`
3. Install Python packages using `sudo pip install`:
  - Flask web framework: `Flask`
  - http libraries: `httplib2`
  - Google API: `google_api_python_client`
  - SQLAlchemy SQL libraries: `SQLAlchemy`
  - Postgres libraries: `psycopg2`
  - System Monitoring: `Glances`

### Configure Server Timezone
1. Run `sudo dpkg-reconfigure tzdata`
2. On the following screens, select `other` and then `UTC`

### Clone and Configure Web Application
1. Navigate to `/var/www`
2. Clone repository from GitHub here:
  - `sudo git clone <REPOSITORY URL>`
3. Make sure git related files are not accessible from the browser
  - Create an `.htaccess` file in the project root directory
  - add the following line to the file and save: `Redirect 404 /\.git`
4. In order to read and write to the uploads folder:
  - move the `uploads` folder up into the `static` folder
  - change ownership of the `uploads` folder, along with the files already in the folder to the `www-data` user and group. From inside the `uploads` folder, run:

  `sudo chown -R www-data:www-data .`
  - in `application.py`, update the constant `UPLOADS_FOLDER` to the absolute path to the `uploads` folder:

  `UPLOADS_FOLDER = r'/var/www/<PROJECT_FOLDER>/static/uploads/'`
5. Change references to `client_secrets.json` to the absolute path of the file

  `CLIENT_ID = json.loads(open(r'/var/www/<PROJECT_FOLDER>/client_secrets.json', 'r').read())['web']['client_id']`

  `oauth_flow = flow_from_clientsecrets(r'/var/www/<PROJECT_FOLDER>/client_secrets.json', scope='')`

### Configure Apache
1. Create an WSGI file named `catalog.wsgi` to act as an interface between Apache and the application. Place it in the application's root directory:

  ```python
  #!/usr/bin/python
  import sys
  import logging
  logging.basicConfig(stream=sys.stderr)
  sys.path.insert(0,"/var/www/<PROJECT_FOLDER>/")

  from application import app as application
  application.secret_key = '<APP_SECRET_KEY>'
  ```
2. In the WSGI file above, replace the PROJECT_FOLDER and APP_SECRET_KEY to match the application.
3. Create an Apache config file for the app at `/etc/apache2/sites-available/catalog.conf`

  ```
  <VirtualHost *:80>
                  ServerName <SITE_IP_ADDRESS>
                  WSGIScriptAlias / /var/www/<PROJECT_FOLDER>/catalog.wsgi
                  <Directory /var/www/<PROJECT_FOLDER>/>
                          Order allow,deny
                          Allow from all
                  </Directory>
                  Alias /static /var/www/<PROJECT_FOLDER>/static
                  <Directory /var/www/<PROJECT_FOLDER>/static/>
                          Order allow,deny
                          Allow from all
                  </Directory>
                  ErrorLog ${APACHE_LOG_DIR}/error.log
                  LogLevel warn
                  CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>
  ```
4. In the config file above, replace <SITE_IP_ADDRESS> and each instance of <PROJECT_FOLDER> to match the ip address of the app and the folder the application is located in.
5. Restart Apache: `sudo service apache2 restart`

### Set-up Database
1. Create a postgres user named `catalog`. Set a password for the user and give the user access to create databases. Set superuser property for `catalog` to no to limit permissions.
2. Create a database named `catalog`.
3. In the files, database_setup.py, lotsofitems.py, and application.py changed (substituting PASSWORD with catalog's password):

  `engine = create_engine('sqlite:///hockeyshop.db')`

  to

  `engine = create_engine(‘postgresql://catalog:<PASSWORD>@localhost/catalog’)`
4. Setup the database: `sudo python database_setup.py`
5. Populate the database: `sudo python lotsofitems.py`

### Ban Attackers
Configure fail2ban in order to lock out ip addresses that have multiple failed login attempts. We'll leave the defaults for number of failed ssh attempts (6) and lock out time (10 minutes).

1. Copy `/etc/fail2ban/jail.conf` to `/etc/fail2ban/jail.local`
2. Edit `/etc/fail2ban/jail.local` with the following settings (replacing <EMAIL_ADDRESS> with a valid email address):
  - destemail = <EMAIL_ADDRESS>
  - action = %(action_mwl)s
  - change the ssh port to 2200
3. Save the changes, then stop and start the fail2ban service
  - `sudo service fail2ban stop`
  - `sudo service fail2ban start`

### Enable Automatic Security updates
Since we already installed `unattended-upgrades`, we can set up automatic security updates by:

1. Run: `sudo dpkg-reconfigure -plow unattended-upgrades`
2. On the next screen, select yes to automatically download and install stable updates.

### System Monitoring
System monitoring is handled with the installed `Glances` package. To monitor the system, from the command line run: `sudo glances`

## Resources
- [Udacity's Configuring Linux Web Servers Course](https://www.udacity.com/course/configuring-linux-web-servers--ud299)
- [DigitalOcean: Initial Server Setup](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04)
- [DigitalOcean: Additional Recommended Steps for New Ubuntu 14.04 Servers](https://www.digitalocean.com/community/tutorials/additional-recommended-steps-for-new-ubuntu-14-04-servers)
- [StackOverflow: Make .git directory web inaccessible](http://stackoverflow.com/questions/6142437/make-git-directory-web-inaccessible)
- To discover which user and group needs write access to the uploads folder: http://unix.stackexchange.com/questions/41241/how-to-check-which-apache-group-i-can-use-for-the-web-server-to-write
- [DigitalOcean: How To Deploy a Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
- [PostgreSQL: Create User](http://www.postgresql.org/docs/9.1/static/app-createuser.html)
- [Udacity Discussion Forums: Google sign-in problems](https://discussions.udacity.com/t/google-sign-in-problems/28191)
- [DigitalOcean: How To Install and Use Fail2ban on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-fail2ban-on-ubuntu-14-04)
- [Automatic Updates](https://help.ubuntu.com/lts/serverguide/automatic-updates.html)
- [Glances](https://pypi.python.org/pypi/Glances)

