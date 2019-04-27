# Unix Server Configuration

Configuring an Ubuntu server to host a [RESTful Application](https://github.com/kheyer/RESTful-App).

IP Address: 18.237.179.198 

SSH Port: 2200

URL: http://ec2-18-237-179-198.us-west-2.compute.amazonaws.com

# Initial Setup

## 1. Create Server

* Create an Ubuntu 18 server instance on AWS Lightsail
* Download SHH key from AWS
* Move key into /.ssh directory
* Change permissions with `chmod 600 /.ssh/aws_key.rsa`
* Connect to server with `ssh -i /.ssh/aws_key.rsa ubuntu@18.237.179.198`

## 2. Update and Secure Server

#### Update packages
* Update packages with `sudo apt-get update`, `sudo apt-get upgrade`

#### Change SSH Port
* Use `sudo nano /etc/ssh/sshd_config` to change the ssh port from 22 to 2200
* Restart SSH with `sudo service ssh restart`

#### Configure Firewall
* Configure firewall with the following
    `sudo ufw default deny incoming`
	`sudo ufw default allow outgoing`
	`sudo ufw allow 2200/tcp`
	`sudo ufw allow www`
	`sudo ufw allow 123/udp`
	`sudo ufw deny 22`
* Enable firewall with `sudo ufw enable`
* Check firewall with `sudo ufw status` to confirm changes
* In AWS Networking, configure the AWS firewall to allow 80(TCP), 123(UDP), and 2200(TCP)
* In AWS Networking, configure the AWS firewall to deny port 22

## 3. Create User `Grader`
* Create user `grader` with `sudo adduser grader`
* Give `grader` root privilege by running `sudo visudo` and adding `grader  ALL=(ALL:ALL) ALL` to the file
* On local machine, use `ssh-keygen` to create an SSH key pair `grader_key` and `grader_key.pub`
* Copy the contents of `grader_key.pub` with `cat /ssh/grader_key.pub`
* On the Ubuntu instance, run `sudo nano /home/grader/.ssh/authorized_keys` and paste the contents of `grader_key.pub`
* Change permissions `chmod 700 .ssh`, `chmod 644 .ssh/authorized_keys`
* Force SSH login by running `sudo nano /etc/ssh/sshd_config` and setting `PasswordAuthentication no`
* Disable root login by running `sudo nano /etc/ssh/sshd_config` and setting `PermitRootLogin no`
* Restart SSH with `sudo service ssh restart`
* The server now only accepts login through `ssh -i ~/.ssh/grader_key -p 2200 grader@18.237.179.198`

## 4. Install Apache2
* Disconnect from the server and login as `grader`
* Install Apache via `sudo apt-get install apache2`
* Install the `mod_wsgi` package for Python 3 via `sudo apt-get install libapache2-mod-wsgi-py3`
* Enable `mod_wsgi` with `sudo a2enmod wsgi`
* Start server with `sudo service apache2 start`


## 5. Set Up Application Files
* Install git via `sudo apt-get install git`
* Create a new directory for the app via `sudo mkdir /var/www/catalog`
* Make `grader` owner of the directory `sudo chown -R grader:grader catalog`
* Navigate to the `catalog` directory and run `git clone https://github.com/kheyer/RESTful-App`
* Update the following files:
* For `app.py`:
	* Rename the file via `mv app.py __init__.py`
	* At line 35 change `client_secrets.json` to `/var/www/catalog/catalog/client_secrets.json`
	* At line 41 change `engine = create_engine('sqlite:///item_database.db')` to `engine = create_engine('postgresql://catalog:PASSWORD@localhost/catalog')`
	* At line 87 change `client_secrets.json` to `/var/www/catalog/catalog/client_secrets.json`
	* At line 102 change `result = json.loads(h.request(url, 'GET')[1])` to `result = json.loads(h.request(url, 'GET')[1].decode('utf-8')`
* For `client_secrets_fake.json`:
	* Rename via `mv client_secrets_fake.json client_secrets.json`
	* Replace the contents with a real client secrets token from Google APIs
* For `database_setup.py`:
	* At line 70 change `engine = create_engine('sqlite:///item_database.db')` to `engine = create_engine('postgresql://catalog:PASSWORD@localhost/catalog')`
* For `database_populate.py`:
	* At line 6 change `engine = create_engine('sqlite:///item_database.db')` to `engine = create_engine('postgresql://catalog:PASSWORD@localhost/catalog')`

## 6. Create VirtualEnv
* Install pip via `sudo apt-get install python3-pip`
* Install the virtual environment via `sudo pip install virtualenv`
* In the `/catalog` directory, create a new environment: `sudo virtualenv venv`
* Activate the environment: `source venv/bin/activate`
* Change ownership: `sudo chmod -R 777 venv`
* Install libpq for Postgresql: `sudo apt-get install libpq-dev`
* Install python packages: `sudo pip install flask httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils`

## 7. Create Database
* Activate postgres with `sudo su - postgres`, `psql`
* `CREATE USER catalog WITH PASSWORD 'password';`
* `ALTER USER catalog CREATEDB;`
* `CREATE DATABASE catalog WITH OWNER catalog;`
* `\c catalog`
* `REVOKE ALL ON SCHEMA public FROM public;`
* `GRANT ALL ON SCHEMA public TO catalog;`

## 8. Configure OAuth Token
* Update the token used in `client_secrets.json` for the new server
* Under Authorized Javascript Origins, add:
	* http://ec2-18-237-179-198.us-west-2.compute.amazonaws.com
	* http://18.237.179.198
	* http://18.237.179.198.xip.io
* Under Authorized Redirect URIs, add:
	* http://ec2-18-237-179-198.us-west-2.compute.amazonaws.com/login
	* http://ec2-18-237-179-198.us-west-2.compute.amazonaws.com/gconnect
	* http://18.237.179.198.xip.io/login
	* http://18.237.179.198.xip.io/gconnect
* Note: Google only allows redirects to top level URIs, so we cannot add login functionality to `http://18.237.179.198/login` or `http://18.237.179.198/gconnect`. This is why we use the `.xip.io` workaround.

## 9. Set Up WSGI Application
* Run `sudo nano /var/www/catalog/catalog.wsgi` and add
```
activate_this = '/var/www/catalog/catalog/venv/bin/activate_this.py'
with open(activate_this) as file_:
exec(file_.read(), dict(__file__=activate_this))

import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/catalog")
sys.path.insert(1, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```
* Run ` sudo nano /etc/apache2/sites-available/catalog.conf` and add
```
<VirtualHost *:80>
	ServerName 18.237.179.198
	ServerAlias ec2-18-237-179-198.us-west-2.compute.amazonaws.com
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
* In the `/catalog` directory activate the virtual environment `source venv/bin/activate`
* Create and populate the database with `python database_setup.py`, `python database_populate.py`
* Deactivate the virtual environment
* Disable the default Apache site `sudo a2dissite 000-default.conf`
* Run `sudo a2ensite catalog`
* Restart Apache2 `sudo service apache2 restart`


The app can now be accessed from [http://ec2-18-237-179-198.us-west-2.compute.amazonaws.com/](http://ec2-18-237-179-198.us-west-2.compute.amazonaws.com/), [http://18.237.179.198/](http://18.237.179.198/) or [http://18.237.179.198.xip.io/](http://18.237.179.198.xip.io/)


Resources:
[https://www.process.st/checklist/ubuntu-server-setup-process/](https://www.process.st/checklist/ubuntu-server-setup-process/)
	
[https://devops.ionos.com/tutorials/deploy-a-flask-application-on-ubuntu-1404/](https://devops.ionos.com/tutorials/deploy-a-flask-application-on-ubuntu-1404/)
	
[https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
	
[https://realpython.com/kickstarting-flask-on-ubuntu-setup-and-deployment/](https://realpython.com/kickstarting-flask-on-ubuntu-setup-and-deployment/)
	
[https://www.codementor.io/abhishake/minimal-apache-configuration-for-deploying-a-flask-app-ubuntu-18-04-phu50a7ft](https://www.codementor.io/abhishake/minimal-apache-configuration-for-deploying-a-flask-app-ubuntu-18-04-phu50a7ft)
	
[https://linoxide.com/linux-how-to/install-flask-python-ubuntu/](https://linoxide.com/linux-how-to/install-flask-python-ubuntu/)
	
[https://tutorials.ubuntu.com/tutorial/install-and-configure-apache](https://tutorials.ubuntu.com/tutorial/install-and-configure-apache)
	
[https://realpython.com/flask-by-example-part-2-postgres-sqlalchemy-and-alembic/](https://realpython.com/flask-by-example-part-2-postgres-sqlalchemy-and-alembic/)
	
[https://blog.theodo.com/2017/03/developping-a-flask-web-app-with-a-postresql-database-making-all-the-possible-errors/](https://blog.theodo.com/2017/03/developping-a-flask-web-app-with-a-postresql-database-making-all-the-possible-errors/)
	
[https://vsupalov.com/flask-sqlalchemy-postgres/](https://vsupalov.com/flask-sqlalchemy-postgres/)
