# Linux Server Configuration

This is the documentation from my process of configuring an Ubuntu server on
Amazon Lightsail as part of [Udacity's Full Stack Web Developer Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004).

**NB: This server is being deployed for assessment and will then be deactivated.**

## Connecting to the server

- IP address: 35.161.33.2
- SSH port: 2200

## Item Catalog web application

Load the hosted web app at http://ec2-35-161-33-2.us-west-2.compute.amazonaws.com

## Software installed and configuration changes made

1. Secure your server:
    1. Update all currently installed packages:
        1. `sudo apt-get update`
        1. `sudo apt-get upgrade`
        1. `sudo apt-get dist-upgrade` (https://askubuntu.com/questions/64291/should-i-upgrade-kernel-packages-on-ec2-instances)

    1. Change the SSH port from 22 to 2200:
        1. `sudo nano /etc/ssh/sshd_config` - edit Port line and save
        1. In Lightsail dashboard, go to Networking and add Custom TCP 2200
        1. `sudo service sshd restart`
        1. Exit and reconnect with ssh command using `-p 2200`

    1. Configure the Uncomplicated Firewall (UFW) to only allow incoming
    connections for SSH (port 2200), HTTP (port 80), and NTP (port 123):
        1. `sudo ufw status` (verify inactive)
        1. `sudo ufw default deny incoming`
        1. `sudo ufw default allow outgoing`
        1. `sudo ufw allow 2200/tcp`
        1. `sudo ufw allow www`
        1. `sudo ufw allow ntp`
        1. `sudo ufw enable`

1. Give `grader` access:
    1. `sudo adduser grader`
    1. `sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader`
    1. `sudo nano /etc/sudoers.d/grader` and replace `ubuntu` with `grader`
    1. `grader` is unable to connect via SSH at this stage, so we need to
    create and install a public key - `su grader`, `mkdir ~/.ssh`, `nano ~/.ssh/authorized_keys`, paste the key created locally, then
    `chmod 700 ~/.ssh` and `chmod 644 ~/.ssh/authorized_keys`
    1. Exit, then SSH in as grader using the matching private key

1. Prepare to deploy your project:
    1. Configure the local timezone - `sudo dpkg-reconfigure tzdata`, select
    "None of the Above" and "UTC"
    1. Install and configure Apache to serve a Python mod_wsgi application -
    `sudo apt-get install apache2` and `sudo apt-get install libapache2-mod-wsgi-py3`
    1. Install PostgreSQL - `sudo apt-get install postgresql`
    1. Verify only local access config: `sudo nano /etc/postgresql/VERSION/main/pg_hba.conf`
    1. Create a new database user - `sudo -u postgres psql` (https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04), `CREATE USER catalog WITH PASSWORD 'catalog';`,
    `CREATE DATABASE catalog WITH OWNER catalog;`, `\q`
    1. Install git - `sudo apt-get install git`

1. Deploy the Item Catalog project:
    1. `cd /var/www`, `sudo git clone https://github.com/tobiasziegler/fsnd-p4-item-catalog.git catalog`,
    `cd catalog`
    1. Configure Python dependencies: (https://www.digitalocean.com/community/tutorials/how-to-use-postgresql-with-your-django-application-on-ubuntu-16-04):
        1. `sudo apt-get install python3-pip`
        1. `sudo pip3 install flask sqlalchemy psycopg2 python-slugify oauth2client`
    1. Adjust the app for server deployment:
        1. Update database connections in `application.py`, `models.py` and
    `populate_database.py` to `postgresql://catalog:catalog@localhost/catalog`
        1. Run `python3 populate_database.py` to load sample data
        1. Load client secret file using absolute path by editing
        `application.py` (https://stackoverflow.com/questions/44742566/wsgi-cant-find-file-in-same-directory-in-app):
            ```
            # Load your Google OAuth client details - see README.md for setup
            import os
            PROJECT_ROOT = os.path.realpath(os.path.dirname(__file__))
            json_url = os.path.join(PROJECT_ROOT, 'google_client_secret.json')
            CLIENT_ID = json.loads(
                open(json_url, 'r').read())['web']['client_id']
            APPLICATION_NAME = "Item Catalog App"
            ```
    1. Configure Apache to access the site (https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps):
        1. `sudo nano /etc/apache2/sites-available/catalog.conf`:
            ```
            <VirtualHost *:80>
                ServerName ec2-35-161-33-2.us-west-2.compute.amazonaws.com

                WSGIScriptAlias / /var/www/catalog/catalog.wsgi

                <Directory /var/www/catalog>
                    Order allow,deny
                    Allow from all
                </Directory>
                Alias /static /var/www/catalog/static
                <Directory /var/www/catalog/static/>
                    Order allow,deny
                    Allow from all
                </Directory>
                <Directorymatch "^/.*/\.git/">
                    Order deny,allow
                    Deny from all
                </Directorymatch>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
            </VirtualHost>
            ```
        1. `sudo a2ensite catalog`
        1. `sudo nano catalog.wsgi` (http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/):
            ```
            #!/usr/bin/python3
            import sys
            import logging
            logging.basicConfig(stream=sys.stderr)
            sys.path.insert(0,"/var/www/catalog/")

            from application import app as application
            application.secret_key = 'super_secret_key'
            ```
        1. `sudo service apache2 restart`

Extra sources of help fixing issues along the way:
- https://stackoverflow.com/questions/15728508/vagrant-wsgi-virtualenv-and-typeerror-module-object-is-not-callable
- https://stackoverflow.com/questions/18069716/why-am-i-getting-module-object-is-not-callable-in-python-3
- https://serverfault.com/questions/128069/how-do-i-prevent-apache-from-serving-the-git-directory
