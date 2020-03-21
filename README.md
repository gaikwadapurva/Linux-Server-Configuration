# Linux-Server-Configuration

## About the project

> A baseline installation of a Linux distribution on a virtual machine and prepare it to host web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers

* IP Address: 13.233.247.44
* SSH Port: 2200
* URL: `http://13.233.247.44/`

**Note:** Until specified, all the commands are issued from the browser console on the AWS site.

### 1. Update all packages

```linux
sudo apt-get update
sudo apt-get upgrade
```

#### 2. Change SSH Port from 22 to 2200

```linux 
$ sudo nano /etc/ssh/sshd_config
```
* Locate the line **port 22** in the file */etc/ssh/sshd_config* and edit it to  **port 2200**, or any other desired port.
* Find the PermitRootLogin line and edit it to no.
* Find the PasswordAuthentication line and edit it to no.
* Save the file and run `sudo service ssh restart`.

##### 3. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

```bash
sudo ufw status
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow www
sudo ufw allow ntp
sudo ufw allow 2200/tcp
sudo ufw enable
```

#### 4. Create a new user grader and Give him `sudo` access

```linux
sudo adduser grader
sudo nano /etc/sudoers.d/grader
```

Then add the following text `grader ALL=(ALL) NOPASSWD:ALL`

#### 5. Setup SSH keys for grader

In order to log into the grader user on the server instance from the command line, it is necessary to generate a key pair either locally or through the Lightsail terminal.  According to the Udacity specifications, generate them locally (IN THE TERMINAL NOW) by first creating a '.ssh' directory to store the keys. Make vagrant user the owner and the vagrant group in order to make the files private enough to meet the Amazon security policy. Moving to the newly created directory '.ssh/', generate a key pair using the 'ssh-keygen' command. These commands need to be issued from inside a vagrant machine on the local command line within the '/home/vagrant' directory.

```linux
/home/vagrant $ mkdir .ssh
/home/vagrant $ chown vagrant:vagrant /home/vagrant/.ssh
/home/vagrant $ cd .ssh
/home/vagrant/.ssh $ ssh-keygen
```

Running the ```ssh-keygen``` command will prompt the user for a file in which to save the newly created keys. Name the file as 'keyPair'. This results in the creation of both 'keyPair' and 'keyPair.pub'.

Read the contents of the 'keyPair.pub' file in order to copy them and move back to the Lightsail browser instance.

```linux
$ cat grader.pub
```

In the instance (accessed through the browser console), switch into the grader user using the command ```$ sudo su - grader``` and create a subdirectory called '.ssh' and set the owner to the grader user and set the permission to read write and execute only to the grader user.

```linux
$ mkdir .ssh
$ chown grader:grader /home/grader/.ssh
$ chmod 700 /home/grader/.ssh
```

'cd' into the '.ssh' directory and create a file called 'authorized_keys', use ```sudo nano``` to edit the 'authorized_keys' file and insert the copied 'keyPair.pub' contents.  Set the permissions of the file to 400.

```linux
$ cd .ssh/
$ touch authorized_keys
$ sudo chmod 400 authorized_keys
$ sudo nano authorized_keys
```

#### 6. Verify if you can login to the instance as grader user from the local machine

Specify the port because considering firewall and ssh configuration from earlier.

```linux
$ ssh -i keyPair grader@13.233.247.44 -p 2200
```


#### 7. Disable root login and force authentication using the key pair generated

Completed this step from the terminal while logged in to the instance as grader.

In the file '/etc/ssh/sshd_config', change "PermitRootLogin without-password" to "PermitRootLogin no" and uncomment the line that reads "PasswordAuthentication no".

Restart ssh using the following command -

```$ sudo service ssh restart```


#### 9. Set the instance timezone

Use the following command to set the instance timezone as required -

```$ sudo dpkg-reconfigure tzdata```


#### 8. Install the required packages

Install Apache and the libapache2-mod-wsgi, and the python set up tools packages and then restarted the Apache service.

```linux
$ sudo apt-get install apache2
$ sudo apt-get install libapache2-mod-wsgi
$ sudo apt-get install python-setuptools
$ sudo service apache2 restart
```


#### 9. Install and configure PostgreSQL

Install PostgreSQL and then open the file '/etc/postgresql/9.5/main/pg_hba.conf' to ensure that remote connections aren't allowed.

```linux
$ sudo apt-get install postgresql
$ sudo nano /etc/postgresql/9.5/main/pg_hba.conf
```

Switched to the postgres user and switch into the interactive postgres mode. Within the interactive prompt, create a new database and user both named catalog and set the password to catalog. Give the catalog user permission to use the catalog database and use ctrl+z to exit the prompt.

```linux
$ sudo su - postgres
postgres $ psql
postgres=# CREATE DATABASE catalog;
# CREATE USER catalog;
# ALTER ROLE catalog WITH PASSWORD 'catalog';
# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
```

**Note:** In your catalog project you should change database engine to

```linux
engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
```


#### 10. Install Git

In order to clone remote repository, install git within the instance using the following command -

```$ sudo apt-get install git```


#### 10. Clone the Catalog app from GitHub and Configure it

Execute the command ```$ sudo a2enmod wsgi``` to enable the mod_wsgi. 'cd' into the '/var/www' directory, create a 'catalog' directory, 'cd' into that directory and create yet another 'catalog' directory and 'cd' within the nested catalog directory.

```linux
$ cd /var/www
/var/www $ mkdir catalog
/var/www $ cd catalog
/var/www/catalog $ mkdir catalog
/var/www/catalog $ cd catalog
```


#### 11. Install psycopg2 and python

* For psycopg2: ```$ sudo apt-get install python-psycopg2```
* For python: ```$ sudo apt-get install python```


#### 12. Clone the catalog app

Clone the Item Catalog app using the following command -

```linux
$ git clone https://github.com/gaikwadapurva/Item-Catalog.git
```


#### 13. Configure apache server

'cd' to the default directory using the command ```cd```.

Create '/etc/apache2/sites-available/catalog.conf' and edit the same.

```linux
sudo nano /etc/apache2/sites-available/catalog.conf
```

Then add the following content:

```
<VirtualHost *:80>
		ServerName 13.233.247.44
		ServerAdmin gaikwadapurva65@gmail.com
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


#### 14. Create a file named catalog.wsgi

'cd' into the '/var/www/catalog' directory and create and edit the 'catalog.wsgi' file:

```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```


#### 15. Reload & Restart Apache Server

```linux
sudo service apache2 reload
sudo service apache2 restart
```

#### 16. Add the mentioned URL to the Authorized Javascript Origins

Added the URL path to the Google Developers console under Credentials/Authorized Javascript Origins.


## Acknowledgments

* [Amazon Lightsail Documentation](https://aws.amazon.com/documentation/lightsail/)
