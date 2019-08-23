# Get started on Lightsail
- Log in/Create Amazon Web Services account
- Create an Lightsail instance
- Choose Ubuntu as instance image
- Choose an image hostname and start it up

# SSH into the instance
- Go to Account, click on the SSH keys tab and download the default ssh keys. The name of the private key might vary and I will annotate it to be `<key_name>.pem` to be generic. For me, the key name is `LightsailDefaultKey-us-west-2.pem`
- In the command prompt, `cd ~/.ssh/` to navigate to the ssh configuration folder. Create a file named `<key_name>.rsa` by typing `touch <key_name>.rsa`
- Paste the content of the private key to `<key_name>.rsa`. You can do this using `cat <path>/<key_name>.pem > <key_name>.rsa`
- SSH into the Ubuntu server by typing `ssh -i <key_name>.rsa <user_name>@<ip_address>`. For me, the command is `ssh -i ~/.ssh/light_sail.rsa ubuntu@35.161.228.242`


# Secure your server
- Update all currently installed packages
	```
	sudo apt-get update
	```
	and
	```sh
	sudo apt-get upgrade
	```

- Change ssh port from 22 to 2200
	- Edit `/etc/ssh/sshd_config` by using `sudo <editior> /etc/ssh/sshd_config` and change the line `Port 22` to `Port 2200`
	- Restart SSH by typing `sudo service ssh restart`
	
- Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
	- Disable all incoming requests: `sudo ufw default deny incoming`
	- Allow all outgoing requests: `sudo ufw default allow outgoing`
	- Allow SSH connections through port 2200: `sudo ufw allow 2200/tcp`
	- Allow HTTP connections through port 80: `sudo ufw allow www`
	- Allow NTP connections through port 123: `sudo ufw allow 123/udp`
	- Enable firewall: `sudo ufw enable`
	- Verify the rules and the firewall are configured correctly: `sudo ufw status`
	- In the Networking/Firewall section of the Amazon Lightsail page, delete the existing SSH applications
	- In the same section, create two applications. Set the new applications to be custom and input the ports to be 2200 (UCP) and 123 (UDP).
	- Verify the ssh port has been changed to port 2200: Log out of the terminal and then type `ssh -i <key_name>.rsa <user_name>@<ip_address> -p 2200` to login again without any problem. For me, the command is `ssh -i ~/.ssh/light_sail.rsa ubuntu@35.161.228.242 -p 2200`
- Disable root login
  - Run `sudo <editor> /etc/ssh/sshd_config` and edit the value of `PermitRootLogin` to no
# Give grader access
- Create a new user account named grader: `sudo adduser grader`
- Give grader the permission to sudo: `sudo <editor> /etc/sudoers.d/grader` and add include the line `grader ALL=(ALL) NOPASSWD: ALL`
- Create an SSH key pair for grader using the ssh-keygen tool:
	- In your local machine, type `ssh-keygen -t rsa` to generate a ssh key pair in directory `~/.ssh`
	- Copy the content of the generated public key with name `<grader_key>.pub`. For me, the filename is `grader_rsa.pub`
	- In the virtual machine, create a .ssh folder: `sudo mkdir /home/grader/.ssh`
	- Type `sudo <editor> /home/grader/.ssh/authorized_keys` and paste the content
	- Restart ssh by typing `sudo service ssh restart`
	- Logout of the terminal and login again as grader: `ssh -i <grader_key> grader@<ip_address> -p 2200`. For me, For me, the command is `ssh -i ~/.ssh/grader_rsa grader@35.161.228.242 -p 2200`
	
# Prepare to deploy your project
- Configure the local timezone to UTC
	- Type `sudo dpkg-reconfigure tzdata`, choose `None of the above` and then `UTC`
- Install and configure Apache to serve a Python mod_wsgi application: 
	- Install Apache: `sudo apt-get install apache2`
    - Install the apache-wsgi package: `sudo apt-get install libapache2-mod-wsgi`
- Install and configure PostgreSQL
	- Install PostgreSQL: `sudo apt-get install postgresql`
	- Make sure PostgreSQL does not allow remote access:
		- View the config file: `sudo cat /etc/postgresql/9.5/main/pg_hba.conf`
		- Verify the bottom of the file looks like this:
		```sh
		# Database administrative login by Unix domain socket
		local   all             postgres                                peer

		# TYPE  DATABASE        USER            ADDRESS                 METHOD

		# "local" is for Unix domain socket connections only
		local   all             all                                     peer
		# IPv4 local connections:
		host    all             all             127.0.0.1/32            md5
		# IPv6 local connections:
		host    all             all             ::1/128                 md5
		```
	- Create a user named catalog that has limited permissions to your catalog application database
		- Change to user postgres: `sudo su postgres` and then type `psql`
		- Create Catalog user: `# CREATE USER catalog;`
		- Set password for Catalog user: `# ALTER ROLE catalog with password 'password';`
		- Create Catalog database: `# CREATE DATABASE catalog;`
		- Grant Catalog database permission to Catalog user: `# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;`
		- Quite psql: `# \q`
		- Switch back to original user: `exit`
- Install git: `sudo apt-get install git`
- Clone and setup your Item Catalog project
	- Make directory for the git project: `sudo mkdir /var/www/item_catalog`
	- Make grader own the directory: `sudo chown -R grader:grader /var/www/item_catalog`
	- `cd` to the app directory: `cd /var/www/item_catalog`
	- Clone the project: `git clone https://github.com/jscu/Item-Catalog.git ItemCatalog`
	- Create a wsgi file with file named item_catalog.wsgi:
		- Run `sudo <editior> ItemCatalog/item_catalog.wsgi` with contents:
		```python
		import sys
		import logging
		logging.basicConfig(stream=sys.stderr)
		sys.path.insert(0, "/var/www/item_catalog")

		from ItemCatalog import app as application
		```
	- Set up a virtual host:
		- Run `sudo <editor> /etc/apache2/sites-available/item_catalog.conf` with contents:
		```python
		   <VirtualHost *:80>
              ServerName 35.161.228.242
              ServerAlias itemcatalog.ddns.net
              ServerAdmin jscu14@gmail.com
              WSGIScriptAlias / /var/www/item_catalog/ItemCatalog/item_catalog.wsgi
              <Directory /var/www/item_catalog/ItemCatalog/>
                Require all granted
              </Directory>
              Alias /static /var/www/item_catalog/ItemCatalog/static
              <Directory /var/www/item_catalog/ItemCatalog/static/>
                Require all granted
              </Directory>
              ErrorLog ${APACHE_LOG_DIR}/error.log
              LogLevel warn
              CustomLog ${APACHE_LOG_DIR}/access.log combined
            </VirtualHost>
		```
		- Enable virutal host: `sudo a2ensite item_catalog`
	- Set up pip: `sudo apt-get install python-pip`
	- Install other dependencies: 
		- Run `sudo apt-get install libpq-dev python-dev`
		- Also run `
			sudo pip install flask oauth2client httplib2 requests sqlalchemy psycopg2
		`
	- Set up DNS (Thanks to No-IP)
		- Go to `https://www.noip.com/`
		- Create an user account
		- Associate the public IP address with a hostname
		- In my case: the DNS maps 35.161.228.242 to itemcatalog.ddns.net
		- Note: The free DNS entry only lasts for 30 days if there is no confirmation
	- Change OAuth 2 configurations
		- Go to Google developer console
		- In the client credentials section, add `http://itemcatalog.ddns.net` as the Authorized JavaScript origins
		- Also add `http://itemcatalog.ddns.net/login` to the Authorized redirect URIs
# Run the application
- Rename application.py: `mv application.py __init__.py`
- Change the path of `client_secrets.json` in `__init__.py` to `/var/www/item_catalog/ItemCatalog/client_secrets.json` 
- Change `app.run(host="0.0.0.0", port=8000, debug=True)` in `__init__.py` to `app.run()`
- In `init_db.py`, change the database engine to `engine = create_engine('postgresql://catalog:<password>@localhost/catalog')`
- Create database and populate with data: `python3 create_test_data.py`
- Restart Apache: `sudo service apache2 restart`

# Access the application
- Go to http://itemcatalog.ddns.net or http://35.161.228.242
- Note that login can only be performed by using http://itemcatalog.ddns.net
