# AWS-lightsail-server-configuration
AWS Lightsail configuration summary for Udacity Fullstack Nanodegree.

The live version of this project can be viewed at http://35.182.92.136.xip.io 
NOTE: Xip.io is a free 'wildcard DNS' service which resolves the IP portion of the provided web address against xip.io's custom DNS server.
While you can certainly still access the project at http://35.182.92.136/ - Google OAuth will not function properly, as it will not accept a direct IP address as an Authorized Redirect URI.

## Accessing Lightsail instance as Grader user via SSH
Copy the graderuser SSH key provided in the submission notes into a file, and into the `~\Users\<username>\.ssh` folder. The Key password is also provided in the notes. The Public key has already been installed on the Lightsail instance to allow remote login for Grader user. 
You should now be able to remote into the Lightsail instance on port 2200 using the following command from Bash
```
ssh grader@35.182.92.136 -p 2200 -i ~/.ssh/graderkey
```

## Configurations summary

#### Update all system packages
Run the following 2 commands from within the Lightsail instance to make sure system packages are up to date
```
sudo apt-get update
sudo apt-get upgrade
```

#### Change SSH default port and configure UFW
Modify the SSHD_Config file to listen for SSH on Port 2200
```
sudo nano /etc/ssh/sshd_config
```
Locate the line which says
`Port 22`
and modify it to
`Port 2200`
. Save the file, and restart the sshd service using `sudo service sshd restart`

Now, configure UFW to allow SSH on port 2200, HTTP on port 80, and NTP on port 123. Deny all other incoming connections.
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 2200/tcp 
sudo ufw allow HTTP
sudo ufw allow NTP
```
Enable the UFW service, and verify the configurations were made correctly.
```
sudo ufw enable
sudo ufw status
```

NOTE: you will need to open the same ports on the Adminstrator panel of your Lightsail instance's Networking tab.

#### Adding Grader user, and giving sudo access
While logged into the Lightsail instance, use the following command to create a new User named 'Grader'
```
sudo adduser grader
```
Follow through the onscreen prompts to finish new user setup, and then verify the new user was created successfully using the `finger grader` command.

Give the Grader user sudo access by running the following commands:
```
sudo touch /etc/sudoers.d/grader - create file
sudo nano /etc/sudoers.d/grader - open and modify file
```
The `/etc/sudoers.d/grader` file should now be open (and blank). Copy in the following line and then save the file:
```
grader ALL=(ALL) NOPASSWD:ALL
```
grader user should now be created and have Sudo access

#### Generating SSH key for grader user
On your local Machine, open up a bash terminal and run the following command
```
ssh-keygen
```
When prompted, enter the following file to save the key to
```
/Users/<username>/.ssh/graderkey
```
Now open a separate terminal and SSH into your Lightsail instance. Run the following commands to switch to the grader user and install the public key onto the server
```
sudo su grader
sudo mkdir .ssh
sudo touch .ssh/authorized_keys
sudo nano .ssh/authorized_keys
```
Copy the contents of the `Users/<username>/.ssh/graderkey.pub` file on your local machine, into the open `.ssh/authorized_keys` file on the Lightsail instance. Save and close the file

Set appropriate user permissions in the newly created files
```
sudo chmod 700 .ssh
sudo chmod 644 .ssh/authorized_keys
```

NOTE: For Lightsail, there is an extra step to get the SSH keys working. You will need to go to your Lightsail Account management console, and upload the Public Key (graderkey.pub) to SSH Key Pair section.

#### Forcing SSH key authentication
Open the SSHD config file using the following command
```
sudo nano /etc/ssh/sshd_config
```
Locate the line which says `PasswordAuthentication yes` and change it to `PasswordAuthentication no`. Then, restart the SSH service
```
sudo service ssh restart
```

#### Install neccessary packages to serve website
To setup your Item catalog on an apache server, you'll first need to install te following packages
```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi 
sudo apt-get python python-pip
sudo apt-get git
sudo apt-get postgresql
```

#### Configure PostgresQL Database for Item catalog
Change to Postgres user in terminal
```
sudo su postgres
```
Open PSQL using the command `psql` while logged in as postgres user.
Create the 'catalog' user and set its password
```
CREATE USER catalog;
ALTER ROLE catalog WITH PASSWORD 'grader';
```
Create the Catalog DB
```
CREATE DATABASE catalog;
```
Give 'catalog' user full access to the Catalog DB
```
GRANT ALL PERMISSIONS ON DATABASE catalog TO catalog;
```

#### Clone Item catalog repo and install to Apache directory
In the `/var/www` directory, create the `Itemcatalog` directory using `mkdir Itemcatalog`, and change to this newly created directory. Next, clone the Item Catalog repo into this directory using the command
```
sudo git clone https://github.com/BenHargreaves/Item-catalog.git
```
Next initialize and open the WSGI file using the following command
```
sudo nano /var/www/Itemcatalog/itemcatalogapp.wsgi
```
Enter the following code into the .wsgi file, and then save and close it.
```
#!/usr/bin/python
import sys
import logging

activate_this = '/var/www/Itemcatalog/Itemcatalog/env/bin/activate_this.py'
execfile(activate_this, dict(__file__=activate_this))

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/Itemcatalog/")

from Itemcatalog import app as application
application.secret_key = 'secret_key'
```
The following changes will also need to be made from the Cloned git Repo.
Rename `/Itemcatalog/application.py` to `__init__.py`
In the files `__init__.py`, `database_setup.py` and `itemseeder.py`, change the references to SQLite
```
engine = create_engine('sqlite:///itemcatalog.db')
```
to
```
engine = create_engine('postgresql://catalog:grader@localhost/catalog')
```
SIn the `/var/www/Itemcatalog/Itemcatalog` directory, setup the Virtual environment packages
```
sudo pip install virtualenv
sudo virtualenv env
```
Next, you will need to install the pip packages required to run the Flask app onto the virtual environment.
```
sudo env/bin/pip install flask sqlalchemy oauth2client
sudo env/bin/pip install httplib2 json requests psycopg2 
```

Finally, run the database_setup.py, and Itemseeder.py files to establish the DB and seed it with items
```
sudo env/bin/python database_setup.py
sudo env/bin/python Itemseeder.py
```

#### Enable Virtual host to serve your site
Add a new configuration file to Apache's 'Sites-Available' or 'Sites-enabled' directories to enable our new virtual host.
```
sudo nano /etc/apache2/sites-available/Itemcatalog.conf
```

In this new blank configuration file add the following code, then save and close
```
<VirtualHost *:80>
                ServerName 35.182.92.136
                ServerAdmin admin@mywebsite.com

                WSGIDaemonProcess Itemcatalog user=ubuntu group=ubuntu threads=5
                WSGIScriptAlias / /var/www/Itemcatalog/itemcatalogapp.wsgi

                <Directory /var/www/Itemcatalog/>
                        WSGIProcessGroup Itemcatalog
                        Order allow,deny
                        Allow from all
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

```

Finally, enable the new Site and restart apache using
```
sudo a2ensite Itemcatalog
sudo service apache2 restart
```

#### Sites and Resources used
1. (Huge help!) https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
2. PSQL setup https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
