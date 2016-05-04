#Linux Server
Udacity Full Stack Web Development - Project 5: Linux Web Server


##Synopsis

This is my fifth and final of a series of projects through Udacity's **Full Stack Web Development** course, through which I have been learning web development. This is an application using Google App Engine.


## Set-Up Instructions:

### Step 1: I created an instance of Ubuntu using Amazon Web Services EC3 (Check out this guide for instructions: [Getting Started with Amazon EC2 Linux Instances](https://raw.githubusercontent.com/ferristocrat/udacity_frontend_p5_neighborhood-map/master/images/screenshot.PNG "Getting Started with Amazon EC2 Linux Instances"))

In the process of creating the Ubuntu instance, you will be prompted to create a private key.  Create a private key, and save it to ~/.ssh folder (in the example I name the file udacity_p4.pem) for use in logging into the server.

##### Step 2: Original Login
```
ssh -i ~/.ssh/udacity_p4.pem ubuntu@ec2-52-87-241-179.compute-1.amazonaws.com
```

##### Step 3: Add first user (grader).  Enter any password you like, but in this example I just used "grader" as the password.

```
sudo adduser grader
```

##### Step 4: Update sshd_config so that users can login using a password

1. Open sshd_config to edit in nano

```
sudo nano /etc/ssh/sshd_config
```

2. Allow password authentication (change "no" to "yes" around line 54)

```
PasswordAuthentication yes
```

3. Restart for change to go through

```
sudo restart ssh

```

##### Step 5: Give grader the permission to sudo

1. Copy default user file (90-cloud-init-users) and save as "grader"

```
sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader
```

2. Open the new file to edit in nano

```
sudo nano /etc/sudoers.d/grader
```

3. Make minor edit to file (change "ubuntu" to "grader") like so:

```
grader ALL=(ALL) NOPASSWD:ALL
```

##### Step 6: Update all currently installed packages

```
sudo apt-get update
sudo apt-get upgrade
```

##### Step 7: Change SSH port from 22 to 2200

1. Once again, open sshd_config to edit in nano

```
sudo nano /etc/ssh/sshd_config
```

2. Modify the following line

```
# What ports, IPs and protocols we listen for
Port 22 <---change port to what you need it to be
```


##### Step 8: Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow ntp
sudo ufw enable
```

Ensure the firewall was set up correctly by typing the following command

```
sudo ufw status
```

##### Step 9: Install and configure Apache to serve a Python mod_wsgi application

```
sudo apt-get install apache2
```

##### Step 10: Install mod_wsgi

1. Install mod_wsgi

```
sudo apt-get install libapache2-mod-wsgi
```

2. Configure Apache to handle requests using the WSGI modeule

Open the the /etc/apache2/sites-enabled/000-default.conf file for editing

```
sudo nano /etc/apache2/sites-enabled/000-default.conf
```

Add the following line at the end of the <VirtualHost *:80> block, right before the closing </VirtualHost> line:

```
WSGIScriptAlias / /var/www/html/myapp.wsgi
```

3. Restart Apache

```
sudo apache2ctl restart
```