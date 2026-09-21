sudo apt update && sudo apt upgrade -y
sudo apt install apache2 -y

sudo systemctl enable apache2
sudo systemctl status apache2

Test: Open browser → http://localhost (should show Apache default page)


sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod ssl
sudo systemctl restart apache2

sudo apt install mysql-server -y
sudo systemctl enable mysql
sudo mysql_secure_installation

Answer prompts:

Validate password component → N (for local dev, keep simple)
Set root password → Y (remember it)
Remove anonymous users → Y
Disallow root login remotely → Y
Remove test database → Y
Reload privilege tables → Y

sudo mysql

CREATE DATABASE devdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'devuser'@'localhost' IDENTIFIED BY 'DevPassword123!';
GRANT ALL PRIVILEGES ON devdb.* TO 'devuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;

sudo apt install php libapache2-mod-php php-mysql php-cli php-curl php-gd php-mbstring php-xml php-zip php-json php-intl -y
php -v


Configure Apache for PHP

sudo nano /etc/apache2/sites-available/dev.conf

<VirtualHost *:80>
    ServerName dev.local
    DocumentRoot /var/www/dev/public

    <Directory /var/www/dev/public>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/dev_error.log
    CustomLog ${APACHE_LOG_DIR}/dev_access.log combined
</VirtualHost>

sudo a2ensite dev.conf
sudo a2dissite 000-default.conf
sudo systemctl reload apache2


sudo nano /etc/hosts

Add: 127.0.0.1 dev.local

sudo mkdir -p /var/www/dev/public
sudo chown -R $USER:$USER /var/www/dev

nano /var/www/dev/public/index.php

<?php
phpinfo();

Visit: http://dev.local → should show PHP info.

<?php
$conn = new mysqli('localhost', 'devuser', 'devpassword', 'devdb');
if ($conn->connect_error) die("Failed: " . $conn->connect_error);
echo "Connected to MySQL successfully!";

sudo apt install phpmyadmin -y

Choose apache2 when prompted
Choose Yes to configure with dbconfig-common
Set a phpMyAdmin password
Access: http://localhost/phpmyadmin
or 
xdg-open http://localhost/phpmyadmin



If you get a 404, add:

sudo ln -s /etc/phpmyadmin/apache.conf /etc/apache2/conf-available/phpmyadmin.conf
sudo a2enconf phpmyadmin
sudo systemctl reload apache2

To access the Db from one server to other. if on localhost than following:
mysql -u devuser -p -h localhost devdb

# Enter password: DevPassword123!
=====================================================================================

 Manual phpMyAdmin Setup (if apt keeps failing)
If reconfiguring keeps breaking, skip dbconfig and set it up by hand. Fastest path:

Step 1: Create the database + user manually
bash
sudo mysql
sql
-- Create phpMyAdmin's control database
CREATE DATABASE phpmyadmin CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create a strong password user (won't fail the policy)
CREATE USER 'phpmyadmin'@'localhost' IDENTIFIED BY 'PhpMyAdmin#2025';

-- Give it full rights to its control DB
GRANT ALL PRIVILEGES ON phpmyadmin.* TO 'phpmyadmin'@'localhost';

-- Also let it see other DBs (optional, so you can browse devdb)
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER, INDEX, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, EVENT, TRIGGER ON *.* TO 'phpmyadmin'@'localhost' WITH GRANT OPTION;

FLUSH PRIVILEGES;
EXIT;
Step 2: Import phpMyAdmin's schema
bash
sudo mysql phpmyadmin < /usr/share/phpmyadmin/sql/create_tables.sql
Step 3: Make phpMyAdmin use these credentials
bash
sudo nano /etc/phpmyadmin/config-db.php
Replace contents with:

php
<?php
$dbuser='phpmyadmin';
$dbpass='PhpMyAdmin#2025';
$basepath='';
$dbname='phpmyadmin';
$dbserver='localhost';
$dbport='3306';
$dbtype='mysql';
Save (Ctrl+O, Enter, Ctrl+X).

Step 4: Restart and test
bash
sudo systemctl restart apache2 mysql
Open http://localhost/phpmyadmin → log in with phpmyadmin / PhpMyAdmin#2025.

🧠 Why This Happened — Quick Lesson
When you ran mysql_secure_installation, you enabled the validate_password component with LOW policy. Even LOW enforces:

Policy	Rules
LOW	length ≥ 8
MEDIUM	length ≥ 8 + number + uppercase + lowercase + special char
STRONG	MEDIUM + not in dictionary file
phpMyAdmin's dbconfig-common step auto-generates a password, or asks you for one — and if it fails the policy, MySQL rejects the CREATE USER, and the installer aborts.

For local dev, the cleanest fix: loosen the policy (Option A) OR just supply a strong password (Option B).

====================================================================================================================

