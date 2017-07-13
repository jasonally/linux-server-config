# Project: Linux Server Configuration

### About
This projects uses concepts from the Deploying to Linux Servers section of Udacity's Full Stack Web Developer Nanodegree program to configure an Ubuntu server hosted on Amazon Lightsail. The server is secured from common attack vectors, contains a properly configured PostgreSQL database, and has my previously completed Item Catalog project running using Apache HTTP Server. This README assumes you have some knowledge of the Linux command line as well as the text editor Vim. If you need a refresher on the command line, check out [Udacity's course on the subject](https://www.udacity.com/course/linux-command-line-basics--ud595). For Vim, check out its [official website](http://www.vim.org/).

### Server Details
**IP address:** 52.41.90.159 (go to this address in your browser to visit my deployed Item Catalog app)
**SSH port:** 2200

### Configuration Steps
These initial commands should be run from your Lightsail terminal window.

#### Updating installed packages
* `sudo apt-get update` updates package indexes.
* `sudo apt-get upgrade` upgrades the installed packages.
* If you see messages in the terminal saying updates are available, even after updating and upgrading, run `sudo apt-get dist-upgrade`. [This Ask Ubuntu thread](https://askubuntu.com/questions/858602/ubuntu-says-updates-available-after-update-and-upgrade) explained the issue.
* Run `sudo apt-get autoremove` to remove dependencies no longer used by installed packages. During my setup there weren't any dependencies to remove, but for others this may vary.
* If you see messages indicating the system needs a reboot, run `sudo reboot`.

#### Change timezone to UTC
My server was already set to UTC. If yours isn't, run `sudo timedatectl set-timezone UTC`.

#### Create user grader
* `sudo adduser grader` creates the new user. I gave this user the password `grader`, though once the server is properly configured you shouldn't encounter many scenarios where you'll be required to enter a password. It most definitely won't happen during authentication since we'll require key-based SSH authentication.
* `sudo cat /etc/sudoers` gives you an initial overview of which accounts on the system have sudo access. `sudo ls /etc/sudoers.d` lets you view the additional directory which contains usernames with sudo access.
* `sudo vim /etc/sudoers.d/grader` creates a new file for the grader user. Type `grader ALL=(ALL:ALL) ALL` into the file, save it, and exit it.

#### SSH Keys
As discussed in the Linux Security portion of the Udacity course material, RSA keys for SSH authentication are a better way to secure your server than using passwords. You'll generate your SSH keys using your local machine (i.e., not your Linux server). If you're trying to create SSH keys without having consulted the Udacity material, [DigitalOcean has a tutorial](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2).

* After you create your keys, return to your Linux terminal and run `su - grader`. 
* Enter the user password, then run `mkdir .ssh`.
* Run `vim .ssh/authorized_keys` to create a new file. Go to the terminal on your local machine, copy the contents of the public key you generated above, switch to your Linux terminal, and paste the key into your `authorized_keys` file. Save the file and exit it.
* Set permissions by running `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`.
* Reload SSH using `service ssh restart`.

If everything worked, you should be able to use SSH from your local machine's terminal to connect to the server. Run `ssh -i [pathToPrivateKeyFilename] grader@52.41.90.159` to test.

#### Disable Root Login
* Return to your Ubuntu terminal and run `sudo su - ubuntu` to change to the root user.
* Change line 28 in `/etc/ssh/sshd_config` from `PermitRootLogin without-password` to `PermitRootLogin no`.
* Save the file, exit it, and run `service ssh restart` for the changes to take effect.

After this, close the Ubuntu terminal. SSH from your local machine into the server using your newly-created grader user to complete the remaining configuration steps. If at any point you get error messages while running these commands, try adding `sudo` to the beginning of the command.

#### Change SSH Port
* Edit `/etc/ssh/sshd_config` and add the line `Port 2200` below the line `Port 22`.
* Restart the SSH service using `sudo service ssh restart`, then logout by running `logout`.
* In your Amazon Web Services account, go into the Networking tab in your server settings and add a custom connection using the TCP protocol in port range 2200.
* In your local machine's terminal, Use an updated command to connect to the server: `ssh -i [pathToPrivateKeyFilename] grader@52.41.90.159 -p 2200`.
If you were able to SSH into the server using the new port, you can edit `/etc/ssh/sshd_config` again and remove Port 22 completely from the file. Restart the SSH service again using `sudo service ssh restart`. Finally, you can remove the SSH connection in port range 20 from your AWS settings.

Adding Port 2200, testing to make sure it works, and then removing Port 22 is repetitive, but if you remove Port 22 first and then add Port 2200, you could be locked out of your server if something goes awry. It's also worth noting if these steps work properly, you won't be able to connect to your server using the Connect button in the AWS dashboard. Make sure your grader user is properly configured and has proper accesses, because we'll be using it in your local machine's terminal from here on out.

#### Configure Uncomplicated Firewall (UFW)
* Block incoming connections on all ports by running `sudo ufw default deny incoming`.
* Allow outgoing connections on all ports by running `sudo ufw default allow outgoing`.
* Allow incoming SSH connections on port 2200 with `sudo ufw allow 2200/tcp`.
* Allow incoming HTTP connections on port 80 with `sudo ufw allow www`.
* Allow incoming NTP (Network Time Protocol) connections on port 123 with `sudo ufw allow ntp`.
* Before enabling the firewall you can confirm the rules have been properly added with `sudo ufw show added`.
* To enable the firewall, use `sudo ufw enable`. Finally, use `sudo ufw status` to check the status of the firewall.

#### Install and configure Apache
* Install Apache with `sudo apt-get install apache2`.
* Install mod_wsgi with `sudo apt-get install python-setuptools libapache2-mod-wsgi`.
* Restart Apache with `sudo service apache2 restart`.

#### Install and configure PostgreSQL
* Install PostgreSQL with `sudo apt-get install postgresql`.
* Edit the configuration file located at `/etc/postgresql/9.5/main/pg_hba.conf` to prohibit remote PostgreSQL connections. You'll want to edit the file so only connections from the local host address `127.0.0.1` for IPv4 and `::1` for IPv6 are permitted.
* Log in as the `postgres` user using `sudo su - postgres`.
* Enter into the PostgreSQL shell using `psql`.
* Create a new database named catalog and create a new user named catalog using `postgres=# CREATE DATABASE catalog; and postgres=# CREATE USER catalog;`
* Set a password for the catalog user (I used `catalog`) using `postgres=# ALTER ROLE catalog WITH PASSWORD 'catalog';`
* Grant the catalog user all privileges to the catalog database with `postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;`
* Quit PostgreSQL using `postgres=# \q`
* Exit from the postgres user using the `exit` command.

#### Install Flask, SQLAlchemy, and Other Required Packages
Run the following commands:
* `sudo apt-get install python-psycopg2 python-flask`
* `sudo apt-get install python-sqlalchemy python-pip`
* `sudo pip install oauth2client`
* `sudo pip install requests`
* `sudo pip install httplib2`
* `sudo pip install flask-seasurf`

#### Install Git
* Run `sudo apt-get install git`.

#### Clone and Deploy the Item Catalog Project
I followed steps similar to ones outlined in [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps) for the basic structure of my app.

* Run `cd /var/www` to move into the `/var/www` directory
* Create the application directory by running `sudo mkdir FlaskApp`
* Move inside this directory with `cd FlaskApp`
* Clone the item catalog project to the server with `git clone https://github.com/jasonally/item-catalog.git`
* Rename the project's name using `sudo mv ./item-catalog ./FlaskApp`.

Once you get to this point you'll have to make a series of changes to prepare your project for deployment on a live server. Depending on the layout of your project you may have to make similar changes or you may not have to make these changes at all. Use your knowledge of Linux and Python here -- you can do it. Some of the changes I made were:
* Copy the contents of '/var/www/FlaskApp/FlaskApp/item-catalog/vagrant/catalog' out to '/var/www/FlaskApp/FlaskApp'. [This thread](https://superuser.com/questions/151504/move-folder-contents-into-parent-folder-linux-commandline) showed me the correct commands to use.
* Once you've made the file moves, delete the `/var/www/FlaskApp/FlaskApp/item-catalog/vagrant/catalog`, `/var/www/FlaskApp/FlaskApp/item-catalog/vagrant`, and `/var/www/FlaskApp/FlaskApp/item-catalog` directories using `sudo rm -r [pathToDirectory]` as needed.
* Use `ls -a` to find remaining hidden files and directories. For me this included a `.DS_Store` file as well as a `.git` directory. I removed all of them. In hindsight, I probably shouldn't have deleted the `.git` directory because I lost the ability to sync changes to my GitHub repository. I could have just ensured the folder wouldn't be publicly accessible from the web.
* Edit `database_setup.py`, `helpers.py`, and `lotsoflistsusers.py` by changing `engine = create_engine('sqlite:///readinglistwithusers.db')` to `engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`.
* Run `sudo python database_setup.py` followed by `sudo python lotsoflistsusers.py` to create the new database schema and populate it with default items.

#### Configure and Enable the Virtual Host
Create `FlaskApp.conf` using `sudo vim /etc/apache2/sites-available/FlaskApp.conf`. Add this code to configure the virtual host:
```<VirtualHost *:80>
	ServerName 52.41.90.159
	ServerAdmin jason[dot]ally[at]gmail[dot]com
	WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
	<Directory /var/www/FlaskApp/FlaskApp/>
		Require all granted
	</Directory>
	Alias /static /var/www/FlaskApp/FlaskApp/static
	<Directory /var/www/FlaskApp/FlaskApp/static/>
		Require all granted
	</Directory>
	ErrorLog ${APACHE_LOG_DIR}/error.log
	LogLevel warn
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Enable the virtual host by running `sudo a2ensite FlaskApp`.

#### Update Google OAuth Client Secrets
Edit the `client_secrets.json` file. In the `javascript_origins` field, you'll want to add the IP address of your AWS server. In this case that's `http://52.11.206.40`.

This IP address must also be added to your project's settings in Google. Go to https://console.cloud.google.com/, log in, and select your Item Catalog project. Go into Project Settings --> API Credentials and select your OAuth 2.0 client ID. Add your server's IP address under Authorized JavaScript origins and save the changes.

#### Update Facebook OAuth Client Secrets
Go to https://developers.facebook.com/, log in, and select your Item Catalog app. Select Facebook Login on the lefthand side of the page, and select Settings if you weren't redirected there by default. Add your server's IP address to the list of Valid OAuth redirect URIs and save the changes.

#### Create the .wsgi File
Run `cd /var/www/FlaskApp` to move to the `/var/www/FlaskApp` directory, then create `flaskapp.wsgi` using `sudo vim flaskapp.wsgi`. Add this code to the flaskapp.wsgi file:
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/FlaskApp/")

from FlaskApp import app as application
application.secret_key = 'Add your secret key'
```

#### Restart Apache
Run `sudo service apache2 restart`, then go to http://52.41.90.159 in your browser to view the website.

### Debugging
I received an Internal Server Error the first time I tried to view my website. If this happens to you, don't panic. [This Ask Ubuntu thread](https://askubuntu.com/questions/14763/where-are-the-apache-and-php-log-files) helped me find the Apache error log file, located at `/var/log/apache2/error.log`.
* Check the logs with `sudo cat /var/log/apache2/error.log`. Make the changes you think you need to make, restart Apache, and try reloading the site. This again will probably be an iterative process.
* After checking the error logs realized I needed to edit `/views/auth.py`, `/views/books.py`, `/views/jsons.py`, and `/views/reading_lists.py`. In each file I needed to change `from helpers import *` to `from FlaskApp.helpers import *`.
* I also realized my app wasn't able to find my `client_secrets.json` and `fb_client_secrets.json` files. I fixed this by adding variables to my `/views/auth.py` file like this:
```
g_client_secret = '/var/www/FlaskApp/FlaskApp/' + 'client_secrets.json'
fb_client_secret = '/var/www/FlaskApp/FlaskApp/' + 'fb_client_secrets.json'
```
And referring to these variables throughout the file as needed. Udacity forum mentor Steve Wooding solved this problem by defining the `/var/www/FlaskApp/FlaskApp` file path directly into his .wsgi file, as noted [here](https://github.com/SteveWooding/fullstack-nanodegree-linux-server-config). For more complex applications this is a better approach since it adheres to the [don't repeat youself principle](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).

### Success!
If you've made it this far, your server should be properly configured and displaying your Item Catalog. You've taken a standard Ubuntu installation and made the following changes to further secure it:
* Updated indexes and packages.
* Created a new user with sudo access.
* Created an RSA key for use with SSH authentication for your newly-created user
Disabled root login for the default root user. You don't need it anyway since your newly-created user has sudo access.
* Changed the SSH port to a non-standard port.
* Configured your firewall specify the protocols and ports from which your server accepts incoming connections.
* Prevented remote PostgreSQL connections.

### Bonus Configurations
Wooding included additional packages in [his server configuration](https://github.com/SteveWooding/fullstack-nanodegree-linux-server-config) which I found helpful.

#### NTP Daemon
The NTP daemon continuously adjusts the system clock so there aren't large differences in time. Install it using `sudo apt-get install ntp`. I left the configuration file `/etc/ntp.conf` in its default state, though you can edit it to include additional time servers.

You can check offsets with `sudo ntpq -p`. A run of my offsets looked like this:
```
remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 0.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 1.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 2.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 3.ubuntu.pool.n .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp.ubuntu.com  .POOL.          16 p    -   64    0    0.000    0.000   0.000
+time-a.timefreq .NIST.           1 u  675 1024  377   39.905    0.552   0.608
*ada.selinc.com  .GPS.            1 u  737 1024  377   17.480    0.380   1.040
-clock.xmission. .XMIS.           1 u  329 1024  357   25.224    0.561   0.547
+srcf-ntp.stanfo .shm0.           1 u  960 1024  377   22.063    2.234   0.871
-minime.fdf.net  25.151.162.158   3 u  282 1024  377   60.198   -0.530  14.344
```
Offsets are reported in milliseconds, so you can see the NTP daemon is working correctly. If your server has large offsets, wait a few minutes and run the check offsets command again. The offsets should have decreased in amount.

#### Automatic Updates
[The Ubuntu documentation](https://help.ubuntu.com/lts/serverguide/automatic-updates.html) explains how to enable automatic updates.
* Install the `unattended-upgrades` package by running `sudo apt-get install unattended-upgrades`.
* The file saved at `/etc/apt/apt.conf.d/50unattended-upgrades` shows which updates get installed. I left it at the default of only installing security updates automatically.
* I then changed the configuration file at `/etc/apt/apt.conf.d/10periodic` to specify how often to perform updates and other tasks. The contents now look like this:
```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```
This configuration means the package list will be updated daily and security updates will be downloaded and installed daily. The local download archive will be cleared weekly. You can review unattended package installs via the log file at `/var/log/apt/unattended-upgrades/unattended-upgrades.log`.

#### Monitor for Repeated Login Attempts
I've taken several steps to secure my server but a major issue remains: the server's ability to operate could be hampered by traffic from repeated login attempts, even if they're unsuccessful. Fail2Ban monitors for repeated unsuccessful login attempts and bans IP addresses from making further attempts. [DigitalOcean's guide](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04) on setting up Fail2Ban was very helpful, as was [this Ask Ubuntu thread](https://askubuntu.com/questions/54771/potential-ufw-and-fail2ban-conflicts) clarifying UFW and Fail2Ban can be used in conjunction with one another.
* Run `sudo apt-get install fail2ban` to install Fail2Ban.
* Run `sudo apt-get install sendmail` to install Sendmail, which you'll need for email notifications about banned IP addresses.
* You'll need to copy the Fail2Ban configuration file into a local version which will override it. This way, if there are Fail2Ban updates to the main configuration file, they won't be merged into changes your local file makes. Run `sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local` to create the local version.
Edit the local file `/etc/fail2ban/jail.local`. I made these changes in the `[DEFAULT]` section, which mean an IP address will be banned for 3,600 seconds and email notifications will contain a logging snippet.
```
bantime = 3600
destemail = [my email address]
action = %(action_mwl)s
```
I configured the [ssh] section as follows. banaction triggers a custom action, ufw-ssh, which we'll define in a moment.
```
[ssh]

enabled  = true
banaction = ufw-ssh
port     = 2200
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
```
Create `/etc/fail2ban/action.d/ufw-ssh.conf` and configure it to contain these settings. Taken together, they mean IPs which repeatedly fail to log in using port 2200 will be banned. After the 3,600-second ban, the restriction will be removed.
```
[Definition]
actionstart =
actionstop =
actioncheck =
actionban = ufw insert 1 deny from <ip> to any port 2200
actionunban = ufw delete deny from <ip> to any port 2200
```
The changes will take effect after you stop Fail2Ban using `sudo service fail2ban stop` and restarting the service with `sudo service fail2ban start`. Blocked IP addresses will be in the output of `sudo ufw status` in addition to other UFW status information.

#### System Monitoring
[Glances](https://pypi.python.org/pypi/Glances) is a cross-platform monitoring tool. Install it by running `sudo apt-get install glances`. To add monitoring for Apache and PostgreSQL usage, add this code to the `/etc/glances/glances.conf` file.
```
list_1_description=Apache Server
list_1_regex=.*apache.*
list_2_description=Postgres
list_2_regex=.*postgres.*
```
Try visiting your website after running `glances`. Use some of the features. You should see corresponding metrics in the Glances dashboard change in real time.
