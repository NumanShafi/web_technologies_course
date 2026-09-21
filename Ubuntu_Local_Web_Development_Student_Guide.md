# Ubuntu Local Web Development Environment — Complete Student Guide

## Apache + PHP + MySQL + phpMyAdmin

This guide explains how to build a **safe local PHP development environment on Ubuntu** using:

- Apache 2 — web server
- PHP — server-side programming language/runtime
- MySQL — relational database server
- phpMyAdmin — browser-based MySQL administration tool
- Apache virtual host — local website configuration
- `/etc/hosts` — local hostname mapping

The goal is to create a development website such as:

```text
http://dev.local
```

with:

```text
Browser
   ↓
Apache
   ↓
PHP
   ↓
MySQL
```

and optionally:

```text
Browser
   ↓
Apache
   ↓
phpMyAdmin
   ↓
MySQL
```

---

# 1. Important Safety Rules

This guide is designed for a **local development machine**.

Before changing system configuration, understand these rules.

## 1.1 Do not modify boot configuration

Nothing in this guide requires changing:

- GRUB
- `/etc/fstab`
- `/boot`
- initramfs
- kernel parameters
- systemd boot targets

Do **not** run commands such as:

```bash
sudo update-grub
sudo grub-install
sudo mkinitramfs
```

for this setup.

They are unrelated to Apache/PHP/MySQL.

---

## 1.2 Do not remove existing packages unnecessarily

We install the packages required for the development environment.

We do not recommend commands such as:

```bash
sudo apt autoremove
```

during this exercise.

An unrelated package could be removed if it is no longer considered a dependency.

---

## 1.3 Inspect before changing existing configuration

Before changing an Apache configuration file, inspect it:

```bash
cat /etc/apache2/sites-available/000-default.conf
```

Before changing PHP configuration:

```bash
php --ini
```

Before changing MySQL:

```bash
sudo systemctl status mysql
```

The principle is:

> **Inspect → Understand → Change → Test**

---

## 1.4 Back up configuration before editing

For important configuration files, make a backup first.

Example:

```bash
sudo cp /etc/apache2/apache2.conf /etc/apache2/apache2.conf.bak
```

For a virtual-host file:

```bash
sudo cp /etc/apache2/sites-available/000-default.conf \
        /etc/apache2/sites-available/000-default.conf.bak
```

You normally do not need to back up every file in the system.

---

# 2. What Are We Building?

At the end, the student should have:

```text
Ubuntu
│
├── Apache
│   └── dev.local
│       └── /var/www/dev/public
│
├── PHP
│   └── index.php
│
├── MySQL
│   └── devdb
│       └── devuser
│
└── phpMyAdmin
    └── browser-based database administration
```

Example request:

```text
http://dev.local
```

Apache receives the request and finds:

```text
/var/www/dev/public/index.php
```

PHP executes the PHP code.

If PHP needs database data, it connects to:

```text
MySQL → devdb
```

---

# 3. Understanding the Packages

## 3.1 Apache2

Package:

```bash
apache2
```

Apache is a web server.

Its job is to:

1. Listen for HTTP requests.
2. Receive requests from browsers.
3. Find the requested resource.
4. Serve static files.
5. Pass PHP files to PHP processing.
6. Return the result to the browser.

Typical HTTP port:

```text
80
```

HTTPS normally uses:

```text
443
```

Important Apache directories:

```text
/etc/apache2/
```

Main configuration:

```text
/etc/apache2/apache2.conf
```

Available websites:

```text
/etc/apache2/sites-available/
```

Enabled websites:

```text
/etc/apache2/sites-enabled/
```

Available modules:

```text
/etc/apache2/mods-available/
```

Enabled modules:

```text
/etc/apache2/mods-enabled/
```

Logs:

```text
/var/log/apache2/
```

---

# 4. PHP

PHP is a server-side programming language.

Example:

```php
<?php

echo "Hello World";
```

The browser does not normally receive the PHP source code.

Instead:

```text
Browser
   ↓
Apache
   ↓
PHP
   ↓
HTML response
   ↓
Browser
```

PHP is useful for:

- Web applications
- APIs
- Database applications
- Authentication systems
- Form processing
- Server-side business logic

---

# 5. MySQL

MySQL is a relational database management system.

It stores structured data in:

```text
Database
 ├── Tables
 │    ├── Rows
 │    └── Columns
```

For example:

```text
devdb
 ├── students
 ├── courses
 └── registrations
```

A database is not the same thing as a database user.

For example:

```text
Database:
devdb

User:
devuser
```

The user receives permissions to work with the database.

---

# 6. phpMyAdmin

phpMyAdmin is a web-based administration application for MySQL/MariaDB.

Instead of writing every SQL command manually, a student can use a browser to:

- Create databases
- Create tables
- Browse records
- Insert records
- Update records
- Delete records
- Run SQL queries
- Manage users and privileges

Important:

> phpMyAdmin is not MySQL.

The relationship is:

```text
Browser
   ↓
Apache
   ↓
PHP
   ↓
phpMyAdmin
   ↓
MySQL
```

---

# 7. Step 1 — Update Ubuntu Packages

Run:

```bash
sudo apt update
```

This refreshes the local package index.

It does **not** install upgrades.

Then:

```bash
sudo apt upgrade -y
```

This upgrades installed packages for which updates are available.

### Why do we do this?

We want the system package information and installed packages to be reasonably current before installing the development stack.

Check Ubuntu:

```bash
lsb_release -a
```

Check kernel:

```bash
uname -r
```

These commands are read-only.

---

# 8. Step 2 — Install Apache

Install:

```bash
sudo apt install apache2 -y
```

Check the service:

```bash
sudo systemctl status apache2
```

Enable Apache to start automatically during normal system startup:

```bash
sudo systemctl enable apache2
```

Start it if it is not running:

```bash
sudo systemctl start apache2
```

Verify:

```bash
sudo systemctl status apache2
```

---

# 9. Understanding systemctl

`systemctl` manages systemd services.

Common commands:

```bash
sudo systemctl start apache2
```

Start now.

```bash
sudo systemctl stop apache2
```

Stop now.

```bash
sudo systemctl restart apache2
```

Stop and start again.

```bash
sudo systemctl reload apache2
```

Ask Apache to reload configuration without a full restart.

```bash
sudo systemctl enable apache2
```

Enable automatic startup.

```bash
sudo systemctl disable apache2
```

Disable automatic startup.

```bash
sudo systemctl status apache2
```

Show current status.

For configuration changes, prefer:

```bash
sudo systemctl reload apache2
```

when a reload is sufficient.

---

# 10. Test Apache

Open:

```text
http://localhost
```

You should see the Apache default page.

You can also test from the terminal:

```bash
curl http://localhost
```

Check which process is listening on port 80:

```bash
sudo ss -ltnp | grep ':80'
```

---

# 11. Apache Modules

Apache has a modular architecture.

Useful modules for our application include:

```text
rewrite
headers
ssl
```

Enable them:

```bash
sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod ssl
```

Then:

```bash
sudo apache2ctl configtest
```

Expected:

```text
Syntax OK
```

Only after the configuration test succeeds:

```bash
sudo systemctl reload apache2
```

---

# 12. What Is mod_rewrite?

`mod_rewrite` allows Apache to rewrite URLs.

For example, an application may want:

```text
http://dev.local/student/25
```

instead of:

```text
http://dev.local/index.php?id=25
```

Frameworks such as Laravel and many custom PHP applications commonly use rewrite rules.

---

# 13. What Is mod_headers?

`mod_headers` allows Apache to modify HTTP headers.

Headers can control things such as:

- Cache behavior
- Security policies
- CORS
- Content handling

Example:

```apache
Header set X-Development-Environment "local"
```

Do not add security headers blindly to production. Understand what each header does first.

---

# 14. What Is mod_ssl?

`mod_ssl` provides Apache integration for TLS/HTTPS.

HTTPS normally uses:

```text
443
```

For a basic local HTTP exercise, SSL is not required.

We enable it here because students should understand that Apache supports HTTPS configuration.

Do not create production certificates from this guide.

---

# 15. Step 3 — Install MySQL

Install:

```bash
sudo apt install mysql-server -y
```

Check:

```bash
sudo systemctl status mysql
```

Enable automatic startup:

```bash
sudo systemctl enable mysql
```

If necessary:

```bash
sudo systemctl start mysql
```

Test:

```bash
sudo mysql
```

Exit:

```sql
EXIT;
```

---

# 16. MySQL Architecture

Think of MySQL as a server.

```text
PHP Application
      |
      | username + password
      v
 MySQL Server
      |
      +---- devdb
      |
      +---- other databases
```

The MySQL server can contain many databases.

Example:

```text
MySQL Server
│
├── devdb
├── university
├── ecommerce
└── testdb
```

Each database can contain many tables.

---

# 17. mysql_secure_installation

Run:

```bash
sudo mysql_secure_installation
```

This utility helps apply common security settings.

The exact questions can vary with the MySQL/Ubuntu version, so do not blindly assume every prompt will look exactly like an older tutorial.

Typical settings for a local development machine are:

- Remove anonymous users → Yes
- Disallow remote root login → Yes
- Remove test database → Yes
- Reload privilege tables → Yes

If asked about password validation, understand that password policy affects passwords created later.

---

# 18. Root User vs Application User

This is extremely important.

A database administrator account such as:

```text
root
```

has very high privileges.

An application should generally **not** use the MySQL root account.

Instead create:

```text
devuser
```

and grant only the privileges the application requires.

Recommended model:

```text
MySQL root
    ↓
Administration only

devuser
    ↓
Application access
    ↓
devdb
```

This is called the **principle of least privilege**.

---

# 19. Database vs User vs Role

These concepts are different.

## Database

A logical container for tables and other database objects.

Example:

```text
devdb
```

## User

An account that authenticates to MySQL.

Example:

```text
'devuser'@'localhost'
```

## Privilege

Permission to perform an operation.

Examples:

```text
SELECT
INSERT
UPDATE
DELETE
CREATE
DROP
ALTER
```

## Role

A named collection of privileges that can be assigned to users.

For example:

```text
app_developer
```

could contain:

```text
SELECT
INSERT
UPDATE
DELETE
```

The role can then be assigned to multiple users.

---

# 20. MySQL Account Names

MySQL accounts have both:

```text
username
```

and:

```text
host
```

Example:

```sql
'devuser'@'localhost'
```

This means:

```text
Username = devuser
Host = localhost
```

These are different accounts:

```text
'devuser'@'localhost'
'devuser'@'127.0.0.1'
'devuser'@'%'
```

Do not assume they are interchangeable.

---

# 21. Create the Development Database

Open MySQL:

```bash
sudo mysql
```

Run:

```sql
CREATE DATABASE devdb
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Why `utf8mb4`?

It provides broad Unicode support and is appropriate for modern applications.

Check databases:

```sql
SHOW DATABASES;
```

---

# 22. Create the Application User

Create:

```sql
CREATE USER 'devuser'@'localhost'
IDENTIFIED BY 'DevPassword123!';
```

This creates an account.

It does **not** automatically give the account access to `devdb`.

That is the next step.

---

# 23. Grant Database Privileges

Run:

```sql
GRANT ALL PRIVILEGES
ON devdb.*
TO 'devuser'@'localhost';
```

Meaning:

```text
devdb.*
```

means:

> All tables/objects within the `devdb` database covered by the applicable privileges.

The user receives privileges on that database.

Then:

```sql
FLUSH PRIVILEGES;
```

On modern MySQL, `CREATE USER` and `GRANT` update the privilege system directly; `FLUSH PRIVILEGES` is generally not required after normal account-management statements. It is harmless in this local exercise, but students should know that it is not a magic command that grants permissions.

Exit:

```sql
EXIT;
```

---

# 24. Verify the User

Login:

```bash
mysql -u devuser -p -h localhost devdb
```

Enter:

```text
DevPassword123!
```

If successful, you are connected as:

```text
devuser
```

to:

```text
devdb
```

---

# 25. Check Current User

Inside MySQL:

```sql
SELECT USER();
```

Also:

```sql
SELECT CURRENT_USER();
```

These are useful for understanding MySQL authentication.

---

# 26. See User Accounts

As an administrator:

```bash
sudo mysql
```

Then:

```sql
SELECT User, Host
FROM mysql.user;
```

This shows MySQL accounts.

Do not modify rows directly in `mysql.user`.

Use SQL account-management statements such as:

```sql
CREATE USER
ALTER USER
DROP USER
GRANT
REVOKE
```

---

# 27. See Privileges of a User

Run:

```sql
SHOW GRANTS FOR 'devuser'@'localhost';
```

You may see something similar to:

```text
GRANT USAGE ON *.* TO `devuser`@`localhost`
GRANT ALL PRIVILEGES ON `devdb`.* TO `devuser`@`localhost`
```

The important part is:

```text
devdb.*
```

The user is not necessarily an administrator of every database.

---

# 28. Why Not Grant All Databases?

Avoid:

```sql
GRANT ALL PRIVILEGES ON *.* TO 'devuser'@'localhost';
```

unless you intentionally want an administrative account.

For an application:

```text
Better:
devdb.*

Not normally:
*.*
```

The narrower permission reduces accidental or malicious damage.

---

# 29. Common MySQL Privileges

## SELECT

Read data:

```sql
SELECT * FROM students;
```

## INSERT

Add data:

```sql
INSERT INTO students(name) VALUES ('Ali');
```

## UPDATE

Modify data:

```sql
UPDATE students
SET name = 'Ahmed'
WHERE id = 1;
```

## DELETE

Delete rows:

```sql
DELETE FROM students
WHERE id = 1;
```

## CREATE

Create database objects such as tables.

## ALTER

Modify table structure.

## DROP

Delete database objects.

`DROP` should be treated carefully because it can destroy objects.

---

# 30. Creating a More Restricted Application User

Instead of:

```sql
GRANT ALL PRIVILEGES ON devdb.* ...
```

a read/write application could receive:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON devdb.*
TO 'devuser'@'localhost';
```

This is often a better starting point.

If the application needs schema-management privileges during development, additional privileges can be granted deliberately.

---

# 31. Revoke a Privilege

Example:

```sql
REVOKE DELETE
ON devdb.*
FROM 'devuser'@'localhost';
```

Check:

```sql
SHOW GRANTS FOR 'devuser'@'localhost';
```

---

# 32. Drop a User

Only if you intentionally want to remove the account:

```sql
DROP USER 'devuser'@'localhost';
```

This does not mean:

```text
DROP DATABASE devdb
```

The database and user are separate objects.

---

# 33. Drop a Database

This is destructive:

```sql
DROP DATABASE devdb;
```

Never execute it casually.

For teaching, students should understand the difference between:

```text
DROP USER
```

and:

```text
DROP DATABASE
```

---

# 34. MySQL Roles

Roles are useful when several users need the same privileges.

Create a role:

```sql
CREATE ROLE 'developer_role';
```

Grant permissions to the role:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON devdb.*
TO 'developer_role';
```

Create a user:

```sql
CREATE USER 'student1'@'localhost'
IDENTIFIED BY 'StudentPassword123!';
```

Assign the role:

```sql
GRANT 'developer_role'
TO 'student1'@'localhost';
```

Set it as the default role:

```sql
SET DEFAULT ROLE 'developer_role'
TO 'student1'@'localhost';
```

Check:

```sql
SHOW GRANTS FOR 'student1'@'localhost';
```

This demonstrates an important enterprise concept:

```text
User
  ↓
Role
  ↓
Privileges
  ↓
Database objects
```

---

# 35. Check MySQL Server Information

Inside MySQL:

```sql
SELECT VERSION();
```

Check databases:

```sql
SHOW DATABASES;
```

Check tables:

```sql
SHOW TABLES;
```

Check current database:

```sql
SELECT DATABASE();
```

---

# 36. Step 4 — Install PHP

Install:

```bash
sudo apt install php libapache2-mod-php php-mysql php-cli php-curl php-gd php-mbstring php-xml php-zip php-intl -y
```

Check:

```bash
php -v
```

---

# 37. Explanation of PHP Packages

## php

Core PHP runtime.

Used to execute PHP programs.

---

## libapache2-mod-php

Allows Apache to process PHP through the Apache PHP module.

This is one way to connect Apache and PHP.

---

## php-mysql

Provides MySQL database support for PHP.

It enables PHP applications to use MySQL through supported PHP database APIs/extensions.

---

## php-cli

PHP Command Line Interface.

Allows:

```bash
php script.php
```

Useful for:

- Scripts
- Cron jobs
- Framework commands
- Testing
- Development tools

---

## php-curl

Provides cURL support.

Useful for:

- Calling APIs
- HTTP requests
- Web services
- Integrating external systems

---

## php-gd

Image-processing support.

Useful for:

- Image resizing
- Image generation
- Image manipulation

---

## php-mbstring

Multibyte string handling.

Important for applications that process languages and Unicode text.

---

## php-xml

XML-related PHP functionality.

Many frameworks and libraries depend on XML support.

---

## php-zip

ZIP archive support.

Useful for:

- Reading ZIP files
- Creating ZIP archives
- Composer packages and other tooling

---

## php-intl

Internationalization functionality.

Useful for:

- Locale-aware formatting
- Dates
- Numbers
- Unicode
- International applications

---

# 38. Check PHP Modules

Run:

```bash
php -m
```

Search for MySQL:

```bash
php -m | grep -i mysql
```

Check configuration:

```bash
php --ini
```

---

# 39. Important Note About PHP Versions

Ubuntu package versions depend on the Ubuntu release and enabled repositories.

Do not hard-code a version such as:

```text
php8.2
```

unless that is actually the PHP version installed on the student's system.

Use:

```bash
php -v
```

and:

```bash
apt policy php
```

to inspect the installed/available version.

---

# 40. Create the Development Website

Create:

```bash
sudo mkdir -p /var/www/dev/public
```

Set ownership for local development:

```bash
sudo chown -R "$USER":"$USER" /var/www/dev
```

This lets the current user edit the project without repeatedly using `sudo`.

---

# 41. Why Use a `public` Directory?

We use:

```text
/var/www/dev/public
```

as the web-accessible document root.

Structure:

```text
/var/www/dev/
└── public/
    └── index.php
```

This is useful because application code can later live outside the public directory.

For example:

```text
/var/www/dev/
├── app/
├── config/
├── storage/
└── public/
    └── index.php
```

Only `public` is exposed directly by Apache.

This is a common application architecture.

---

# 42. Create index.php

Create:

```bash
nano /var/www/dev/public/index.php
```

Add:

```php
<?php

phpinfo();
```

Save the file.

---

# 43. Configure Apache Virtual Host

Create:

```bash
sudo nano /etc/apache2/sites-available/dev.conf
```

Use:

```apache
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
```

---

# 44. Understanding the VirtualHost

## `<VirtualHost *:80>`

Apache listens for HTTP requests on port 80.

```text
*:80
```

means requests arriving on port 80 for the configured server can be matched to this virtual host.

---

## ServerName

```apache
ServerName dev.local
```

This identifies the local hostname.

---

## DocumentRoot

```apache
DocumentRoot /var/www/dev/public
```

This tells Apache where website files are located.

---

## Directory

```apache
<Directory /var/www/dev/public>
```

Controls Apache access to this filesystem directory.

---

## AllowOverride All

Allows `.htaccess` files to override permitted Apache configuration directives.

This is useful for many PHP frameworks.

For simple sites, it is not always necessary.

---

## Require all granted

Allows Apache to serve content from this directory.

---

## ErrorLog

```apache
ErrorLog ${APACHE_LOG_DIR}/dev_error.log
```

Stores errors for this virtual host.

---

## CustomLog

```apache
CustomLog ${APACHE_LOG_DIR}/dev_access.log combined
```

Stores HTTP access records.

---

# 45. Test Apache Configuration Before Reloading

This is one of the most important safety steps.

Run:

```bash
sudo apache2ctl configtest
```

Expected:

```text
Syntax OK
```

If you get an error, **do not reload Apache yet**.

Fix the configuration first.

---

# 46. Enable the Site

Run:

```bash
sudo a2ensite dev.conf
```

Then test again:

```bash
sudo apache2ctl configtest
```

Then:

```bash
sudo systemctl reload apache2
```

---

# 47. Should We Disable the Default Site?

The original guide uses:

```bash
sudo a2dissite 000-default.conf
```

This is normally safe for a development machine, but it is not necessary to create the application.

A safer teaching approach is:

1. Enable `dev.conf`.
2. Test it.
3. Confirm `dev.local` works.
4. Only then decide whether the default site should be disabled.

If the machine already hosts another website, **do not disable existing sites without understanding them**.

Check enabled sites:

```bash
ls -la /etc/apache2/sites-enabled/
```

Disable the default site only when appropriate:

```bash
sudo a2dissite 000-default.conf
```

Then:

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

---

# 48. Configure `/etc/hosts`

Open:

```bash
sudo nano /etc/hosts
```

Add:

```text
127.0.0.1 dev.local
```

Now the local computer resolves:

```text
dev.local
```

to:

```text
127.0.0.1
```

Test:

```bash
getent hosts dev.local
```

Expected result should include:

```text
127.0.0.1 dev.local
```

---

# 49. Test PHP

Open:

```text
http://dev.local
```

You should see the PHP information page.

Alternatively:

```bash
curl http://dev.local
```

---

# 50. PHP Information and Security

`phpinfo()` is useful for development.

It reveals detailed information about:

- PHP version
- Extensions
- Configuration
- Environment
- Server interface

Do not leave a public `phpinfo()` page exposed on an Internet-facing production server.

For local development it is useful for troubleshooting.

After testing, replace it with a normal application page.

---

# 51. Test PHP → MySQL

Create:

```bash
nano /var/www/dev/public/db-test.php
```

Use:

```php
<?php

$conn = new mysqli(
    'localhost',
    'devuser',
    'DevPassword123!',
    'devdb'
);

if ($conn->connect_error) {
    die("Failed: " . $conn->connect_error);
}

echo "Connected to MySQL successfully!";
```

Visit:

```text
http://dev.local/db-test.php
```

Expected:

```text
Connected to MySQL successfully!
```

---

# 52. Important Security Lesson About Passwords

The previous example puts the password directly into PHP:

```php
'DevPassword123!'
```

This is acceptable only as a simple classroom test.

Do not use this approach in a serious application.

Better approaches include:

- Environment variables
- Secret management
- Framework configuration
- Protected configuration files
- Deployment secret stores

For example, an application might ultimately use:

```text
DB_HOST
DB_DATABASE
DB_USERNAME
DB_PASSWORD
```

instead of hard-coding credentials.

---

# 53. Check MySQL From the Command Line

Use:

```bash
mysql -u devuser -p -h localhost devdb
```

Enter:

```text
DevPassword123!
```

Then:

```sql
SELECT DATABASE();
```

Expected:

```text
devdb
```

---

# 54. Test Database Operations

Inside MySQL:

```sql
CREATE TABLE students (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL
);
```

Insert:

```sql
INSERT INTO students (name, email)
VALUES ('Ali', 'ali@example.com');
```

Read:

```sql
SELECT * FROM students;
```

Update:

```sql
UPDATE students
SET name = 'Ahmed'
WHERE id = 1;
```

Delete:

```sql
DELETE FROM students
WHERE id = 1;
```

---

# 55. Database User Testing

Check privileges:

```sql
SHOW GRANTS FOR 'devuser'@'localhost';
```

Test whether the user can create a table:

```sql
CREATE TABLE test_table (
    id INT PRIMARY KEY
);
```

Then:

```sql
DROP TABLE test_table;
```

Only perform destructive operations on test objects.

---

# 56. Installing phpMyAdmin

Install:

```bash
sudo apt install phpmyadmin -y
```

During installation, the package may ask configuration questions.

The exact prompts depend on the Ubuntu/phpMyAdmin package version.

If asked for a web server and Apache is listed, select:

```text
apache2
```

If asked whether to configure a database using `dbconfig-common`, understand what it is doing before accepting.

---

# 57. What Is dbconfig-common?

`dbconfig-common` is a Debian/Ubuntu mechanism used by packages that require databases.

It can help automate:

- Database creation
- Database user creation
- Schema installation
- Package database configuration

The problem is not that it is inherently bad.

The problem is that automatic database configuration can fail when:

- Password policies reject generated passwords
- Existing users/databases conflict
- A previous installation is incomplete
- Configuration files contain stale values
- The package expects a different MySQL configuration

When troubleshooting, inspect the error instead of repeatedly reinstalling packages.

---

# 58. Access phpMyAdmin

Depending on the package configuration, use:

```text
http://localhost/phpmyadmin
```

or:

```text
http://dev.local/phpmyadmin
```

Check Apache configuration if necessary.

---

# 59. Inspect phpMyAdmin Apache Configuration

Check:

```bash
ls -la /etc/phpmyadmin/
```

Look for:

```text
apache.conf
```

If required, a configuration can be enabled with:

```bash
sudo a2enconf phpmyadmin
```

Then:

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Do not create duplicate configuration entries without checking whether phpMyAdmin is already configured.

Check:

```bash
ls -la /etc/apache2/conf-enabled/
```

---

# 60. Avoid Blindly Creating Symlinks

The command:

```bash
sudo ln -s /etc/phpmyadmin/apache.conf \
    /etc/apache2/conf-available/phpmyadmin.conf
```

can create a duplicate/conflicting configuration if the package has already installed the configuration.

First inspect:

```bash
ls -la /etc/apache2/conf-available/ | grep phpmyadmin
```

and:

```bash
ls -la /etc/apache2/conf-enabled/ | grep phpmyadmin
```

Only create a missing configuration link when you have confirmed it is actually absent and required by the installed package.

---

# 61. Manual phpMyAdmin Database Setup

Use manual setup only if the package installation actually requires it.

Open MySQL:

```bash
sudo mysql
```

Create the phpMyAdmin configuration database:

```sql
CREATE DATABASE phpmyadmin
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Create a dedicated phpMyAdmin control user:

```sql
CREATE USER 'phpmyadmin'@'localhost'
IDENTIFIED BY 'PhpMyAdmin#2025';
```

Grant privileges to its own database:

```sql
GRANT ALL PRIVILEGES
ON phpmyadmin.*
TO 'phpmyadmin'@'localhost';
```

---

# 62. Do Not Give phpMyAdmin Unnecessary Global Privileges

A previous shortcut sometimes suggested is:

```sql
GRANT ... ON *.* ...
```

This gives very broad privileges and is not necessary just to make phpMyAdmin work as an administrative interface.

Understand the distinction:

```text
phpMyAdmin application
        ≠
MySQL root account
        ≠
devuser
```

For normal development, you can often log into phpMyAdmin using the database user you actually want to administer, such as:

```text
devuser
```

If phpMyAdmin requires its configuration storage/control account, configure that account according to the phpMyAdmin package's supported configuration.

---

# 63. Import phpMyAdmin's Configuration Schema

If your installed phpMyAdmin package provides:

```text
/usr/share/phpmyadmin/sql/create_tables.sql
```

and the manual configuration actually requires the control database, import it with:

```bash
sudo mysql phpmyadmin < /usr/share/phpmyadmin/sql/create_tables.sql
```

Verify:

```bash
sudo mysql
```

Then:

```sql
USE phpmyadmin;
SHOW TABLES;
```

---

# 64. phpMyAdmin Configuration

Inspect:

```bash
ls -la /etc/phpmyadmin/
```

Do not blindly replace package-managed configuration files.

If a file such as:

```text
config-db.php
```

exists, inspect it:

```bash
sudo cat /etc/phpmyadmin/config-db.php
```

Package-managed configuration should normally be changed carefully because package upgrades may modify or regenerate parts of it.

---

# 65. Database Users You May Encounter

A typical development system might have:

```text
root@localhost
```

Purpose:

```text
Database administration
```

Then:

```text
devuser@localhost
```

Purpose:

```text
Application development
```

And possibly:

```text
phpmyadmin@localhost
```

Purpose:

```text
phpMyAdmin configuration/control database
```

The exact accounts installed by packages can vary.

Always inspect the actual system:

```sql
SELECT User, Host FROM mysql.user;
```

---

# 66. What Should the Application User Be Allowed to Do?

For a simple development application:

```text
devuser
   |
   +-- SELECT
   +-- INSERT
   +-- UPDATE
   +-- DELETE
```

For development where the application itself creates/modifies schema:

```text
devuser
   |
   +-- SELECT
   +-- INSERT
   +-- UPDATE
   +-- DELETE
   +-- CREATE
   +-- ALTER
   +-- INDEX
```

For production applications, permissions should be reviewed more carefully.

---

# 67. Check All Databases

As administrator:

```bash
sudo mysql
```

Then:

```sql
SHOW DATABASES;
```

---

# 68. Check Tables in devdb

```sql
USE devdb;
SHOW TABLES;
```

Describe a table:

```sql
DESCRIBE students;
```

Alternative:

```sql
SHOW CREATE TABLE students;
```

---

# 69. Check Database Size

A useful query:

```sql
SELECT
    table_schema AS database_name,
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2)
        AS size_mb
FROM information_schema.tables
GROUP BY table_schema
ORDER BY size_mb DESC;
```

This is useful for learning database monitoring.

---

# 70. Apache Logs

Apache logs are extremely important.

List:

```bash
ls -lh /var/log/apache2/
```

View errors:

```bash
sudo tail -f /var/log/apache2/error.log
```

View access requests:

```bash
sudo tail -f /var/log/apache2/access.log
```

For our virtual host:

```bash
sudo tail -f /var/log/apache2/dev_error.log
```

and:

```bash
sudo tail -f /var/log/apache2/dev_access.log
```

---

# 71. MySQL Logs

Depending on Ubuntu/MySQL configuration, logs can be found in different locations.

Inspect the service:

```bash
sudo systemctl status mysql
```

Inspect journal logs:

```bash
sudo journalctl -u mysql
```

Follow recent logs:

```bash
sudo journalctl -u mysql -f
```

---

# 72. PHP Configuration

Find the active configuration:

```bash
php --ini
```

For CLI configuration:

```bash
php -i | grep 'Loaded Configuration File'
```

Apache's PHP configuration can differ from CLI configuration depending on the PHP integration and Ubuntu package setup.

A useful test page is:

```php
<?php
phpinfo();
```

---

# 73. Useful Apache Commands

List loaded modules:

```bash
apache2ctl -M
```

Check virtual hosts:

```bash
sudo apache2ctl -S
```

Test configuration:

```bash
sudo apache2ctl configtest
```

List enabled sites:

```bash
ls -la /etc/apache2/sites-enabled/
```

List available sites:

```bash
ls -la /etc/apache2/sites-available/
```

List enabled modules:

```bash
ls -la /etc/apache2/mods-enabled/
```

---

# 74. Useful MySQL Commands

Check service:

```bash
sudo systemctl status mysql
```

Login as local administrative account:

```bash
sudo mysql
```

Login as application user:

```bash
mysql -u devuser -p -h localhost devdb
```

Check version:

```sql
SELECT VERSION();
```

Check current user:

```sql
SELECT USER();
```

Check authenticated account:

```sql
SELECT CURRENT_USER();
```

List databases:

```sql
SHOW DATABASES;
```

List users:

```sql
SELECT User, Host FROM mysql.user;
```

Check privileges:

```sql
SHOW GRANTS FOR 'devuser'@'localhost';
```

---

# 75. Useful Ubuntu Network Commands

Check listening ports:

```bash
sudo ss -ltnp
```

Check Apache:

```bash
sudo ss -ltnp | grep ':80'
```

Check HTTPS:

```bash
sudo ss -ltnp | grep ':443'
```

Check MySQL:

```bash
sudo ss -ltnp | grep ':3306'
```

Remember:

```text
80   → HTTP
443  → HTTPS
3306 → MySQL
```

Do not expose MySQL to the network just because port 3306 exists.

For a local application, keeping MySQL bound locally is generally preferable.

---

# 76. Localhost vs 127.0.0.1

These are related but not always identical in application behavior.

Examples:

```text
localhost
127.0.0.1
```

For MySQL, connection behavior can differ because `localhost` may use a local Unix socket while `127.0.0.1` uses TCP.

Check:

```bash
mysql -u devuser -p -h localhost devdb
```

and:

```bash
mysql -u devuser -p -h 127.0.0.1 devdb
```

If authentication behaves differently, inspect MySQL accounts:

```sql
SELECT User, Host FROM mysql.user;
```

---

# 77. Understanding the MySQL Host Part

Suppose you have:

```sql
'devuser'@'localhost'
```

This is not necessarily equivalent to:

```sql
'devuser'@'127.0.0.1'
```

You can create the second account explicitly if your application requires TCP authentication to `127.0.0.1`:

```sql
CREATE USER 'devuser'@'127.0.0.1'
IDENTIFIED BY 'DevPassword123!';
```

Then grant only the required privileges:

```sql
GRANT ALL PRIVILEGES
ON devdb.*
TO 'devuser'@'127.0.0.1';
```

Do not create multiple accounts unless you understand why they are needed.

---

# 78. Remote Database Access

The normal local setup is:

```text
PHP
 ↓
MySQL on same machine
```

Example:

```bash
mysql -u devuser -p -h localhost devdb
```

If connecting from another server:

```text
Application Server
       |
       | TCP
       v
Database Server
```

This requires additional configuration:

- MySQL network binding
- Firewall
- MySQL account host permissions
- Network routing
- TLS/security considerations

Do not simply use:

```text
'devuser'@'%'
```

and open port 3306 to the Internet.

That is unnecessarily dangerous.

For production, use a private network/VPN/firewall and strong authentication controls.

---

# 79. If Remote Access Is Required

First inspect the current MySQL bind configuration.

Possible locations include:

```text
/etc/mysql/mysql.conf.d/mysqld.cnf
```

Inspect:

```bash
sudo grep -R "bind-address" /etc/mysql/ 2>/dev/null
```

Do not change it blindly.

After any MySQL configuration change:

```bash
sudo mysql --help
```

and your installed configuration should be validated according to the actual MySQL version.

Then:

```bash
sudo systemctl restart mysql
```

Check:

```bash
sudo systemctl status mysql
```

And:

```bash
sudo ss -ltnp | grep 3306
```

Only expose the database on networks where it is intentionally required.

---

# 80. Firewall Considerations

Check UFW:

```bash
sudo ufw status
```

For a local-only development machine, do not open MySQL to the network unnecessarily.

If Apache must be accessible through the firewall, a common configuration is:

```bash
sudo ufw allow 'Apache'
```

Before changing firewall rules on a remote server, make sure you understand how you are connected to the server so that you do not lock yourself out.

---

# 81. File Permissions

Check:

```bash
ls -la /var/www/dev
```

and:

```bash
ls -la /var/www/dev/public
```

The web server needs read access to public files.

For local development, owning the project as your user is convenient:

```bash
sudo chown -R "$USER":"$USER" /var/www/dev
```

Do not solve every permission problem with:

```bash
chmod -R 777
```

Avoid this.

It gives excessive permissions and hides the real cause of the problem.

---

# 82. Common Permission Model

A useful model is:

```text
Developer user
   ↓
owns project files

Apache
   ↓
reads public files
```

If an application needs to write files, give write permission only to the directories that actually require it.

---

# 83. Troubleshooting Apache 404

If:

```text
http://dev.local
```

returns 404:

Check hostname:

```bash
getent hosts dev.local
```

Check virtual host:

```bash
sudo apache2ctl -S
```

Check document root:

```bash
ls -la /var/www/dev/public
```

Check Apache log:

```bash
sudo tail -50 /var/log/apache2/dev_error.log
```

---

# 84. Troubleshooting Apache 403

Check:

```bash
ls -ld /var/www
ls -ld /var/www/dev
ls -ld /var/www/dev/public
```

Check virtual host configuration:

```bash
sudo apache2ctl -S
```

Check:

```apache
Require all granted
```

Then:

```bash
sudo apache2ctl configtest
```

---

# 85. Troubleshooting PHP Not Executing

If the browser downloads PHP source instead of executing it, inspect:

```bash
php -v
```

Check Apache modules:

```bash
apache2ctl -M | grep -i php
```

Depending on the Ubuntu/PHP version and integration, PHP may be configured differently.

Restart/reload only after checking configuration.

---

# 86. Troubleshooting MySQL Login

If:

```bash
mysql -u devuser -p -h localhost devdb
```

fails:

Check that MySQL is running:

```bash
sudo systemctl status mysql
```

Check account:

```bash
sudo mysql
```

Then:

```sql
SELECT User, Host FROM mysql.user;
```

Check privileges:

```sql
SHOW GRANTS FOR 'devuser'@'localhost';
```

---

# 87. Troubleshooting "Access Denied"

Typical causes:

```text
Wrong password
Wrong username
Wrong host part
User does not exist
User lacks database privileges
Application is connecting to another database
```

Check:

```sql
SHOW GRANTS FOR 'devuser'@'localhost';
```

---

# 88. Troubleshooting Apache Configuration

Always run:

```bash
sudo apache2ctl configtest
```

before:

```bash
sudo systemctl reload apache2
```

If it fails:

```bash
sudo systemctl status apache2
```

and:

```bash
sudo journalctl -u apache2 -n 100 --no-pager
```

Then inspect the Apache configuration.

---

# 89. A Safe Configuration Workflow

Use this workflow whenever modifying Apache:

```text
1. Inspect
      ↓
2. Back up important configuration
      ↓
3. Edit
      ↓
4. apache2ctl configtest
      ↓
5. Reload
      ↓
6. Test website
      ↓
7. Check logs
```

Do not restart blindly after every edit.

---

# 90. What Does `reload` vs `restart` Mean?

Reload:

```bash
sudo systemctl reload apache2
```

Apache rereads configuration while keeping the service running.

Restart:

```bash
sudo systemctl restart apache2
```

Apache is stopped and started again.

For configuration changes, a successful reload is often preferable.

If a module or package installation requires a restart, then use restart.

---

# 91. Verify Everything

Run:

```bash
php -v
```

Apache:

```bash
sudo systemctl is-active apache2
```

MySQL:

```bash
sudo systemctl is-active mysql
```

PHP modules:

```bash
php -m
```

Apache configuration:

```bash
sudo apache2ctl configtest
```

Virtual hosts:

```bash
sudo apache2ctl -S
```

Website:

```bash
curl -I http://dev.local
```

Database:

```bash
mysql -u devuser -p -h localhost devdb
```

---

# 92. Complete Installation Command Summary

Update:

```bash
sudo apt update
sudo apt upgrade -y
```

Apache:

```bash
sudo apt install apache2 -y
sudo systemctl enable apache2
sudo systemctl start apache2
```

Apache modules:

```bash
sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod ssl
sudo apache2ctl configtest
sudo systemctl reload apache2
```

MySQL:

```bash
sudo apt install mysql-server -y
sudo systemctl enable mysql
sudo systemctl start mysql
```

PHP:

```bash
sudo apt install php libapache2-mod-php php-mysql php-cli php-curl php-gd php-mbstring php-xml php-zip php-intl -y
```

Verify:

```bash
php -v
```

---

# 93. Complete Database Setup

Login:

```bash
sudo mysql
```

Run:

```sql
CREATE DATABASE devdb
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;

CREATE USER 'devuser'@'localhost'
IDENTIFIED BY 'DevPassword123!';

GRANT ALL PRIVILEGES
ON devdb.*
TO 'devuser'@'localhost';

FLUSH PRIVILEGES;

SHOW GRANTS FOR 'devuser'@'localhost';

EXIT;
```

For a classroom/local environment, this is straightforward.

For a real application, use a strong unique password and store it securely rather than committing it into source code.

---

# 94. Complete Apache Virtual Host

File:

```text
/etc/apache2/sites-available/dev.conf
```

Content:

```apache
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
```

Enable:

```bash
sudo a2ensite dev.conf
```

Test:

```bash
sudo apache2ctl configtest
```

Reload:

```bash
sudo systemctl reload apache2
```

---

# 95. Complete `/etc/hosts` Entry

Add:

```text
127.0.0.1 dev.local
```

Test:

```bash
getent hosts dev.local
```

Then:

```text
http://dev.local
```

---

# 96. Recommended Final Project Structure

```text
/var/www/dev/
│
├── public/
│   ├── index.php
│   └── db-test.php
│
├── app/
│
├── config/
│
├── storage/
│
└── README.md
```

Only:

```text
public/
```

is exposed directly through Apache.

---

# 97. What Happens When the Student Opens dev.local?

The complete request flow is:

```text
Student enters:

http://dev.local
       |
       v
/etc/hosts
       |
       v
127.0.0.1
       |
       v
Apache :80
       |
       v
ServerName dev.local
       |
       v
VirtualHost dev.conf
       |
       v
/var/www/dev/public
       |
       v
index.php
       |
       v
PHP
       |
       v
HTML response
       |
       v
Browser
```

If PHP connects to MySQL:

```text
PHP
 |
 | devuser + password
 v
MySQL
 |
 v
devdb
 |
 v
tables
```

---

# 98. Important Security Lessons for Students

Never assume that because a machine is used for development, every security practice is unnecessary.

Students should learn:

### Use least privilege

Do not use MySQL root from the application.

### Do not expose databases unnecessarily

Do not open port 3306 to the Internet without a specific architecture and security controls.

### Do not use `chmod 777`

Fix ownership and permissions correctly.

### Do not hard-code production passwords

Use environment variables/secrets.

### Do not expose phpinfo publicly

Remove or protect it after testing.

### Do not expose phpMyAdmin unnecessarily

phpMyAdmin is an administration interface and should not be casually exposed to the public Internet.

### Validate configuration before reloading

Apache:

```bash
sudo apache2ctl configtest
```

---

# 99. Safe Maintenance Checklist

Before modifying Apache:

```bash
sudo apache2ctl configtest
```

After installing packages:

```bash
sudo systemctl status apache2
sudo systemctl status mysql
```

Check ports:

```bash
sudo ss -ltnp
```

Check Apache:

```bash
sudo apache2ctl -S
```

Check PHP:

```bash
php -v
php -m
```

Check MySQL:

```bash
sudo mysql -e "SELECT VERSION();"
```

Check website:

```bash
curl -I http://dev.local
```

---

# 100. Student Exercises

## Exercise 1 — Apache

1. Install Apache.
2. Start Apache.
3. Enable Apache.
4. Open `http://localhost`.
5. Find the Apache access log.
6. Find the Apache error log.

---

## Exercise 2 — PHP

Create:

```php
<?php

echo "Hello from PHP";
```

Save as:

```text
/var/www/dev/public/hello.php
```

Visit:

```text
http://dev.local/hello.php
```

---

## Exercise 3 — MySQL

Create:

```text
students
```

with:

```text
id
name
email
department
```

Insert at least five students.

Run:

```sql
SELECT * FROM students;
```

---

## Exercise 4 — Privileges

Run:

```sql
SHOW GRANTS FOR 'devuser'@'localhost';
```

Explain what each privilege means.

---

## Exercise 5 — Least Privilege

Create:

```sql
CREATE USER 'readonly'@'localhost'
IDENTIFIED BY 'ReadonlyPassword123!';
```

Grant only:

```sql
SELECT
```

on:

```text
devdb.*
```

Then test whether `readonly` can:

```text
SELECT
INSERT
UPDATE
DELETE
```

Document the results.

---

## Exercise 6 — Roles

Create:

```sql
CREATE ROLE 'student_role';
```

Grant:

```sql
SELECT, INSERT, UPDATE, DELETE
```

on:

```text
devdb.*
```

Then create:

```text
student1
student2
```

Assign the role to both users.

Verify their grants.

---

# 101. Final Architecture

After completing the guide, the student should understand this architecture:

```text
                    Ubuntu
                      |
        +-------------+-------------+
        |                           |
      Apache                      MySQL
        |                           |
    dev.local                    devdb
        |                           |
 /var/www/dev/public           Tables
        |
       PHP
        |
        +--------------------------+
        |
    phpMyAdmin
        |
        v
      MySQL
```

And the security model:

```text
MySQL root
   |
   +---- Administrative operations

devuser
   |
   +---- Application operations
   |
   +---- devdb only

phpMyAdmin control account
   |
   +---- phpMyAdmin configuration/control database
```

---

# 102. Final Verification Checklist

The environment is ready when all of the following work:

```text
[ ] Apache installed
[ ] Apache running
[ ] Apache enabled
[ ] http://localhost works
[ ] rewrite module enabled
[ ] headers module enabled
[ ] SSL module enabled
[ ] PHP installed
[ ] php -v works
[ ] PHP MySQL extension available
[ ] MySQL installed
[ ] MySQL running
[ ] devdb exists
[ ] devuser exists
[ ] devuser can connect to devdb
[ ] devuser privileges are understood
[ ] dev.local resolves to 127.0.0.1
[ ] Apache virtual host works
[ ] http://dev.local works
[ ] PHP page executes
[ ] PHP can connect to MySQL
[ ] phpMyAdmin works if installed
[ ] Apache configuration passes configtest
[ ] No unnecessary boot configuration was changed
```

---

# 103. Key Commands Cheat Sheet

## Apache

```bash
sudo systemctl status apache2
sudo systemctl start apache2
sudo systemctl stop apache2
sudo systemctl restart apache2
sudo systemctl reload apache2
sudo systemctl enable apache2

sudo apache2ctl configtest
sudo apache2ctl -S
sudo apache2ctl -M
```

## PHP

```bash
php -v
php -m
php --ini
```

## MySQL

```bash
sudo systemctl status mysql
sudo mysql
mysql -u devuser -p -h localhost devdb
```

Inside MySQL:

```sql
SHOW DATABASES;
SELECT User, Host FROM mysql.user;
SHOW GRANTS FOR 'devuser'@'localhost';
SELECT USER();
SELECT CURRENT_USER();
SELECT DATABASE();
```

## Apache logs

```bash
sudo tail -f /var/log/apache2/error.log
sudo tail -f /var/log/apache2/access.log
sudo tail -f /var/log/apache2/dev_error.log
sudo tail -f /var/log/apache2/dev_access.log
```

## Network

```bash
sudo ss -ltnp
getent hosts dev.local
curl -I http://dev.local
```

---

# 104. Final Conceptual Summary

The most important concepts from this practical are:

1. **Apache serves web requests.**
2. **PHP executes server-side application code.**
3. **MySQL stores relational data.**
4. **phpMyAdmin is a web-based administration tool for MySQL.**
5. **A database and a database user are different objects.**
6. **A MySQL account includes both a username and a host.**
7. **Privileges determine what a user can do.**
8. **Roles group privileges for easier management.**
9. **Application users should normally have fewer privileges than administrators.**
10. **`root` should not normally be used by web applications.**
11. **Apache virtual hosts allow multiple websites/configurations on one server.**
12. **`/etc/hosts` provides local hostname resolution for `dev.local`.**
13. **`apache2ctl configtest` should be used before applying Apache configuration changes.**
14. **Logs are essential for troubleshooting.**
15. **Avoid unnecessary changes to boot configuration and existing services.**
16. **Never use broad permissions such as `chmod -R 777` as a generic fix.**
17. **Never expose MySQL or phpMyAdmin to the Internet without understanding the security architecture.**
18. **Use least privilege and separate administrative accounts from application accounts.**

This setup provides a controlled foundation for learning PHP web development, SQL, database design, authentication, Apache configuration, and basic Linux server administration without making unnecessary changes to the system's boot configuration.
