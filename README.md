# Project 6: Linux Server Configuration

For this project I configured an Ubuntu Linux server on [AWS's Lightsail service](https://lightsail.aws.amazon.com). The server is configured to host a Flask webapp that I completed for the backend section of Udacity's [Full Stack Nanodegree program](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004). The backend project (catalog app) is currently live [here](http://18.188.177.178.xip.io/cars/). The project required several modifications to the baseline Linux installation for improved security, updates and/or addition of packages/libraries, and user configuration. The configuration steps are detailed below.

Site URL: [http://18.188.177.178.xip.io/](http://18.188.177.178.xip.io/)
Public IP: 18.188.177.178
SSH Port: 2200

## User Management
The server was configured to require key authentication for SSH access. I created a 'grader' user for use in project evaluation:

  ```bash
  > sudo adduser grader
  ```
and granted the new user sudo permissions:

  ```bash
  > sudo vim /etc/sudoers.d/grader
  grader ALL=(ALL) NOPASSWD:ALL
  ```
Lastly, I generated a key pair on my local machine, added the public key to the authorized keys on the server, and provided the private key to the grader with the project submission. In addition to adding the 'grader' user, remote login was disabled for the 'root' user by editing `sshd_config`.

### Security
First, all packages were updated with `sudo apt-get update` and 'sudo apt-get upgrade'. I then configured the Uncomplicated Firewall (UFW) as required in the project specification:

  ```bash
  > sudo uf default deny incoming
  > sudo ufw default allow outgoing
  > sudo ufw allow 2200/tcp
  > sudo ufw allow 80/tcp
  > sudo ufw allow 123/tcp
  > sudo ufw enable
  ```
  
## Application Functionality
This server was configured to serve a Flask application. I had to install several bits of software to meet the app's dependencies. These included:
  - git
  - apache2
  - libapache2-mod-wsgi
  - postgresql
  - pip
  - Flask
  - httplib2
  - sqlalchemy
  - requests
  - oauth2client
  - python-psycopg2
  
Once all of the required software was installed, I cloned my [catalog project repo](https://github.com/ewhanley/catalog-app/tree/master/vagrant):
  ```bash
  > cd /var/www
  > sudo mkdir catalog-app
  > sudo git clone https://github.com/ewhanley/catalog-app.git
  > sudo chmod 777 /var/www/catalog-app/vagrant/static/uploads  #Permits app users to upload images
  ``` 
  
### Database Refactor
The original catalog app was configured to use a sqlite database to store user ids and posted content. However, for this project, I reconfigured the app to use postgresql. There were a few steps involved in this change, the first being creation of a new postgresql user and database:
  ```bash
  > sudo su - postgres
  > psql
  postgres=# CREATE USER catalog WITH PASSWORD 'secure_password';
  postgres=# ALTER USER catalog CREATEDB;
  postgres=# CREATE DATABASE catalog with OWNER catalog;
  postgres=# \c catalog
  catalog=# REVOKE ALL ON SCHEMA public FROM public;
  catalog=# GRANT ALL ON SCHEMA public TO catalog;
  ```
  
Following creation of the 'catalog' postgresql user and catalog database, I updated the Flask project to use postgresql engine instead of sqlite per the [SQLAlchemy docs](http://docs.sqlalchemy.org/en/latest/core/engines.html).

### Apache Configuration
I completed the required Apache and WSGI configurations to host my catalog app on the server. If first configured and enabled a new virtaual host:
  ```bash
  > sudo vim /etc/apache2/sites-available/itemsCatalog.conf
  <VirtualHost *:80>
		ServerName 18.188.177.178.xip.io
		ServerAdmin ewhanley@gmail.com
		WSGIScriptAlias / /var/www/FlaskApp/itemsCatalog.wsgi
		<Directory /var/www/catalog-app/vagrant>
			Order allow,deny
			Allow from all
		</Directory>
		<Directory /var/www/catalog-app/vagrant/static/>
			Order allow,deny
			Allow from all
		</Directory>
		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>

  > sudo a2ensite itemsCatalog
  ```
Next, I created the .wsgi file:
  ```bash
  > cd /var/www/catalog-app/vagrant
  > sudo vim itemsCatalog.wsgi
  #!/usr/bin/python
  import sys
  import logging
  logging.basicConfig(stream=sys.stderr)
  sys.path.insert(0, '/var/www/catalog-app/vagrant')
  
  from project import app as application
  application.secret_key='super_secret_key'
  ```
Finally, I restarted Apache:
  ```bash
  > sudo service apache2 restart
  ```
  
### Oauth2 Changes
The catalog app can autheticate users through either Google or Facebook Oauth2. However, some light refactoring was necessary to get these services working with the app deployed on the new Linux server. The first step required changing paths to the oauth client secrets json files to absolute paths. In addition to this change, I had to update the valid redirect URIs for both the Google and Facebook oauth services via their respective developer consoles. However, this still did not get the app's authentication in working order. Google does not permit public IP addresses as redirect URIs but, rather, requires an actual domain. Fortunately, [Basecamp](https://basecamp.com/) provides [http://xip.io/](http://xip.io/), a free wildcard DNS service for IP addresses without associated domains. Hence, in the virtual host configuration above, you'll see the public IP has `.xip.io` appended to it.

### Miscellaneous
In order to prevent unwanted access to the .git folder, I added the following:
  ```bash
  > cd /var/www/catalog-app/vagrant
  > sudo vim .htaccess
  RedirectMatch 404 /\.git
  ```
  
## Resources
  - The Udacity FSND 'Deploying to Linux Servers' course material
  - https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
  - http://docs.sqlalchemy.org/en/latest/core/engines.html
  - https://stackoverflow.com/questions/36109708/google-oauth2-0-web-applications-authorized-redirect-uris-must-end-with-a-pub/36112649
  - http://xip.io/
  - https://stackoverflow.com/questions/6142437/make-git-directory-web-inaccessible



    
