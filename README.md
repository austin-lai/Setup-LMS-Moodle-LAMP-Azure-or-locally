# Setup-LMS-Moodle-LAMP-Azure-or-locally

> Austin Lai | March 12th, 2022

---

<!-- Description -->

Basic scripts and key step to setup Moodle - Learning Management System with LAMP in Azure or locally.

You may pair with IaC (Infrastructure as Code) such as Terraform and Ansible to build full automation of deploying Moodle - Learning Management System.

<!-- /Description -->

<br />

## Table of Contents

<!-- TOC -->

- [Setup-LMS-Moodle-LAMP-Azure-or-locally](#setup-lms-moodle-lamp-azure-or-locally)
    - [Table of Contents](#table-of-contents)
    - [Bash Scripts to create Moodle - Learning Management System with LAMP in Azure or locally.](#bash-scripts-to-create-moodle---learning-management-system-with-lamp-in-azure-or-locally)

<!-- /TOC -->

## Bash Scripts to create Moodle - Learning Management System with LAMP in Azure or locally.

**Disclaimer:** 

- Script does not take security into consideration, may you need to harden the system according to NIST or CIS hardening guide.
- The build only for **development usage** and **NOT** for **PRODUCTION**

```bash
# Once you have base OS image done, on Azure you may use Ubuntu
# Locally you may use RHEL or CentOS
# For our case, we are spinning Ubuntu Azure VM


# Setup user
# Add user into "sudo" group
# Change user password same as username
# Create .ssh directory for sshkey
sudo useradd -m -s /bin/bash -U USERNAME
sudo usermod -aG sudo USERNAME
sudo echo 'USERNAME:USERNAME' | sudo chpasswd
sudo mkdir /home/austin/.ssh


# Create sshkey locally or you may use your own key
# For our case, we are using the root sshkey for our user
# Change the permission of the key and ownership
sudo cp ~/.ssh/authorized_keys /home/USERNAME/.ssh/
sudo chown -R USERNAME:USERNAME /home/USERNAME/.ssh/
sudo chmod 600 /home/USERNAME/.ssh/authorized_keys


# Update system and upgrade
sudo apt update -y && sudo apt full-upgrade -y
sudo apt autoclean && audo apt autoremove
sudo apt update -y


# Install LAMP (Linux + Apache + MySQL + PHP) and require tools
sudo apt install -y apache2 mysql-client mysql-server php libapache2-mod-php wget git mlocate unzip


# Install additional tools require for Moodle LMS
sudo apt install -y graphviz aspell ghostscript clamav php7.4-pspell php7.4-curl php7.4-gd php7.4-intl php7.4-mysql php7.4-xml php7.4-xmlrpc php7.4-ldap php7.4-zip php7.4-soap php7.4-mbstring php7.4-json phpmyadmin  libpoppler-dev poppler-utils


# Restart apache server
sudo service apache2 restart


# Download Moodle LMS installation files
# unzip the package into 
cd ~
wget https://download.moodle.org/stable311/moodle-3.11.3.zip
moodle_zip=`find -type f -name "moodle-*.zip" | sed 's/^\.\///'`
sudo unzip -q $moodle_zip

# Once unzip
# Move Moodle directory to /var/www/html to host as web service for our apache
sudo mv moodle /var/www/html/

# Change the permission and ownership of Moodle directory
sudo chown -R root /var/www/html/moodle
sudo chmod -R 0755 /var/www/html/moodle
sudo ls -la /var/www/html/

# Create moodle data directory to store your moodle data
# Change the permission and ownership of the directory
sudo mkdir /var/moodledata
sudo chown -R www-data /var/moodledata
sudo chmod -R 777 /var/moodledata


# Enable MySQL services
# Setup MySQL with secure installtion
sudo systemctl enable mysql
sudo mysql_secure_installation

# Change the configuration MySQL
sudo sed -i.bak '$a default_storage_engine = innodb' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo sed -i '$a innodb_file_per_table = 1' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo sed -i '$a innodb_file_format = Barracuda' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo sed -i '$a innodb_large_prefix = 1' /etc/mysql/mysql.conf.d/mysqld.cnf

# Restart MySQL
sudo service mysql restart

# Login MySQL as root
sudo mysql -u root -p

# And grant the permission to the new user created dedicated for Moodle LMS only
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION;
CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodle'@'localhost' IDENTIFIED BY 'YOUR_PASSWORD';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodle'@'localhost' WITH GRANT OPTION;  
# OR using this > --- GRANT ALL ON moodle.* TO 'moodle@localhost' IDENTIFIED BY 'EVYD@m00dle.mysql' WITH GRANT OPTION;
FLUSH PRIVILEGES;


################################################################
# Once above step done
# Your Moodle LMS base setup is done
# You may visit the site and it will redirect you to Moodle Web Setup Wizard


# Once completed the Moodle Web Setup Wizard
## Perform customization and securing from WebUI

### Administration > Site administration > Server > Email > Outgoing mail configuration: Set your smtp server and authentication if required (so your Moodle site can send emails). You can also set a norepy email on this page.

### Administration > Site administration > Server > Server > Support contact. Set your support contact email.

### Administration > Site administration > Server > System paths: Set the paths to du, dot and aspell binaries.
    # Path to du: /usr/bin/du
    # Path to apsell: /usr/bin/aspell
    # Path to dot: /usr/bin/dot
     
### Administration > Site administration > Server > HTTP: If you are behind a firewall you may need to set your proxy credentials in the 'Web proxy' section.

### Administration > Site administration > Location > Update timezones: Run this to make sure your timezone information is up to date.


################################################################
# Additional task

## Configure Cron: Moodle's background tasks (e.g. sending out forum emails and performing course backups) are performed by a script which you can set to execute at specific times of the day. This is known as a cron script. Please refer to the Cron instructions. (https://docs.moodle.org/311/en/Cron)
sudo crontab -u www-data -e
* * * * * /usr/bin/php  /var/www/html/moodle/admin/cli/cron.php >/dev/null

## Set up backups: See Site backup (https://docs.moodle.org/311/en/Site_backup) and Automated course backup (https://docs.moodle.org/311/en/Automated_course_backup).

## Secure your Moodle site: Read the Security recommendations (https://docs.moodle.org/311/en/Security_recommendations).

## Increasing the maximum upload size See Installation FAQ Maximum upload file size - how to change it?

## Check mail works : From Site administration > Server > Test outgoing mail configuration, use the link to send yourself a test email. Don't be tempted to skip this step.

## ClamAV things
sudo mkdir /var/quarantine
sudo chown -R www-data /var/quarantine
# Navigate to Site Administration > Plugins > Antivirus plugins > Manage antivirus plugins
# Enable ClamAV antivirus
# Click on Settings
# Set the proper settings
# In previous Moodle branches: Check "Use ClamAV on uploaded files" ClamAV Path : /usr/bin/clamscan Quarantine Directory : /var/quarantine



################################################################
# Register Moodle in Apps Registration under Azure to allow OAuth2 SSO authentication.
# OAuth 2 service - token_endpoint for issuer Microsoft
# https://login.microsoftonline.com/common/oauth2/v2.0/token
# OAuth 2 service - authorization_endpoint
# https://login.microsoftonline.com/common/oauth2/v2.0/authorize



################################################################
# Allow http or https firewall on azure



################################################################
# Create self-sign certificate using Let's Encrypt
sudo snap install --classic certbot
sudo certbot --apache -v
# Congratulations! You have successfully enabled HTTPS on https://moodle-dev.xyz.cloudapp.azure.com
```

<br />

---

> Do let me know any command or step can be improve or you have any question you can contact me via THM message or write down comment below or via FB
