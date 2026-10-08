# Complete PHP Authentication System with Sessions

## LAMP Stack — Classroom Hands-On Guide

**Technology Stack**

* Linux — Ubuntu
* Apache
* MySQL
* PHP
* Browser
* PHP Sessions
* HTML Forms

---

# 1. Learning Objectives

By the end of this practical, students will be able to:

1. Understand how PHP authentication works.
2. Create a user registration/signup system.
3. Store users in MySQL.
4. Hash passwords securely.
5. Create a login system.
6. Verify passwords using PHP.
7. Understand PHP sessions.
8. Store user information in `$_SESSION`.
9. Protect pages from unauthorized users.
10. Implement logout.
11. Destroy sessions correctly.
12. Understand important PHP session settings.
13. Understand session fixation.
14. Use `session_regenerate_id()`.
15. Understand common authentication problems.
16. Debug PHP authentication applications.
17. Understand why passwords should never be stored as plain text.
18. Understand prepared SQL statements and SQL injection.
19. Understand basic authentication security.

---

# 2. Prerequisite

Before starting this practical, students should already have:

* Ubuntu/Linux
* Apache
* MySQL
* PHP
* Basic PHP syntax
* Variables
* Arrays
* Conditions
* Loops
* Functions
* HTML forms
* GET and POST
* Basic MySQL
* Basic SQL

The Apache VirtualHost has already been configured.

Our application will use:

```text
http://dev.local
```

The Apache DocumentRoot is:

```text
/var/www/dev/public
```

---

# 3. Application We Are Going to Build

We will build a small authentication application.

The application will have:

```text
Signup
   ↓
User enters name/email/password
   ↓
PHP validates data
   ↓
Password is hashed
   ↓
User stored in MySQL
   ↓
Login
   ↓
PHP verifies email/password
   ↓
Session created
   ↓
Dashboard accessible
   ↓
Logout
   ↓
Session destroyed
```

The final application will contain:

```text
Signup
Login
Dashboard
Profile
Logout
Session management
Authentication protection
Password hashing
Database interaction
```

---

# 4. Project Directory Structure

Our project will be:

```text
/var/www/dev/
│
├── public/
│   ├── index.php
│   ├── signup.php
│   ├── login.php
│   ├── logout.php
│   ├── dashboard.php
│   └── profile.php
│
├── config/
│   └── database.php
│
├── includes/
│   ├── session.php
│   ├── auth.php
│   ├── header.php
│   └── footer.php
│
└── README.md
```

---

# 5. Understanding the Folder Structure

## public/

This contains files that users access through the browser.

For example:

```text
http://dev.local/signup.php
http://dev.local/login.php
http://dev.local/dashboard.php
```

These files are located in:

```text
/var/www/dev/public/
```

---

## config/

Contains configuration files.

Example:

```text
database.php
```

This file contains database connection information.

It is outside `public/` so that users cannot directly request it from the browser.

---

## includes/

Contains reusable PHP functionality.

For example:

```text
session.php
auth.php
header.php
footer.php
```

These files are included by other PHP files.

---

# 6. Create the Project Directories

Run:

```bash
sudo mkdir -p /var/www/dev/public
sudo mkdir -p /var/www/dev/config
sudo mkdir -p /var/www/dev/includes
```

Create application files:

```bash
sudo touch /var/www/dev/public/index.php
sudo touch /var/www/dev/public/signup.php
sudo touch /var/www/dev/public/login.php
sudo touch /var/www/dev/public/logout.php
sudo touch /var/www/dev/public/dashboard.php
sudo touch /var/www/dev/public/profile.php
```

Create configuration file:

```bash
sudo touch /var/www/dev/config/database.php
```

Create include files:

```bash
sudo touch /var/www/dev/includes/session.php
sudo touch /var/www/dev/includes/auth.php
sudo touch /var/www/dev/includes/header.php
sudo touch /var/www/dev/includes/footer.php
```

Check:

```bash
tree /var/www/dev
```

Expected:

```text
/var/www/dev
├── config
│   └── database.php
├── includes
│   ├── auth.php
│   ├── footer.php
│   ├── header.php
│   └── session.php
└── public
    ├── dashboard.php
    ├── index.php
    ├── login.php
    ├── logout.php
    └── profile.php
```

---

# 7. Set Appropriate Permissions

During classroom development, you can use:

```bash
sudo chown -R $USER:www-data /var/www/dev
```

Then:

```bash
sudo chmod -R 755 /var/www/dev
```

We don't need to give everyone write access.

Avoid:

```bash
chmod -R 777 /var/www/dev
```

Do not use `777` as a normal solution.

---

# 8. Test the PHP Application

Open:

```text
http://dev.local
```

Initially, `index.php` may be empty.

Put a simple test in:

```text
/var/www/dev/public/index.php
```

```php
<?php

echo "PHP Authentication Application";
```

Open:

```text
http://dev.local
```

Expected:

```text
PHP Authentication Application
```

---

# 9. Create the Database

Open MySQL:

```bash
sudo mysql
```

Create database:

```sql
CREATE DATABASE auth_demo
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Check:

```sql
SHOW DATABASES;
```

Select database:

```sql
USE auth_demo;
```

---

# 10. Create the Users Table

Create:

```sql
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Check:

```sql
DESCRIBE users;
```

Expected important columns:

```text
id
name
email
password_hash
created_at
```

---

# 11. Why Do We Store password_hash Instead of password?

Never create:

```text
password
```

and store:

```text
123456
```

This is dangerous.

Instead:

```text
User password
      ↓
password_hash()
      ↓
Long secure hash
      ↓
Database
```

For example, the database may contain something similar to:

```text
$2y$10$...
```

The original password cannot simply be read from the database.

---

# 12. Create a Database User

It is better not to use the MySQL `root` account from our PHP application.

Create a dedicated application user:

```sql
CREATE USER 'authuser'@'localhost'
IDENTIFIED BY 'DevPassword123!';
```

Give access only to our database:

```sql
GRANT ALL PRIVILEGES
ON auth_demo.*
TO 'authuser'@'localhost';
```

Apply privileges:

```sql
FLUSH PRIVILEGES;
```

Check:

```sql
SHOW GRANTS FOR 'authuser'@'localhost';
```

Exit:

```sql
EXIT;
```

---

# 13. Database Connection

File:

```text
/var/www/dev/config/database.php
```

The purpose of this file is to establish the MySQL connection.

Example using MySQLi:

```php
<?php

$host = "localhost";
$username = "authuser";
$password = "YOUR_PASSWORD_HERE";
$database = "auth_demo";

$conn = new mysqli(
    $host,
    $username,
    $password,
    $database
);

if ($conn->connect_error) {
    die("Database connection failed.");
}

$conn->set_charset("utf8mb4");
```

Important:

Do not display the actual database password in error messages.

Bad:

```php
die("Database password is: " . $password);
```

Good:

```php
die("Database connection failed.");
```

---

# 14. Test the Database Connection

Temporarily create:

```text
/var/www/dev/public/dbtest.php
```

Put:

```php
<?php

require_once __DIR__ . '/../config/database.php';

echo "Database connection successful.";
```

Open:

```text
http://dev.local/dbtest.php
```

Expected:

```text
Database connection successful.
```

After testing, remove the file:

```bash
sudo rm /var/www/dev/public/dbtest.php
```

Why?

Because temporary diagnostic files should not remain publicly accessible.

---

# 15. Understand `__DIR__`

You will see:

```php
__DIR__
```

This represents the directory of the current PHP file.

For example:

```text
/var/www/dev/public/signup.php
```

Inside `signup.php`:

```php
__DIR__
```

represents:

```text
/var/www/dev/public
```

Therefore:

```php
require_once __DIR__ . '/../config/database.php';
```

means:

```text
/var/www/dev/public
        ↓
..
        ↓
/var/www/dev
        ↓
config/database.php
```

This is much safer than depending on the current working directory.

---

# 16. Signup Page

File:

```text
/var/www/dev/public/signup.php
```

The signup page should:

1. Display a form.
2. Accept name.
3. Accept email.
4. Accept password.
5. Accept password confirmation.
6. Validate input.
7. Check whether email already exists.
8. Hash password.
9. Insert user into database.
10. Redirect user to login.

Basic form:

```html
<form method="POST">

    <label>Name</label>
    <input type="text" name="name">

    <label>Email</label>
    <input type="email" name="email">

    <label>Password</label>
    <input type="password" name="password">

    <label>Confirm Password</label>
    <input type="password" name="confirm_password">

    <button type="submit">Create Account</button>

</form>
```

---

# 17. GET vs POST

When the signup page is opened:

```text
GET /signup.php
```

The form is displayed.

When the user submits the form:

```text
POST /signup.php
```

The data is sent to PHP.

We can check:

```php
if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    // process signup

}
```

This is called request method checking.

---

# 18. Validate Signup Input

We should never blindly trust user input.

Example:

```php
$name = trim($_POST['name'] ?? '');
$email = trim($_POST['email'] ?? '');
$password = $_POST['password'] ?? '';
$confirmPassword = $_POST['confirm_password'] ?? '';
```

Check name:

```php
if ($name === '') {
    $error = "Name is required.";
}
```

Check email:

```php
if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
    $error = "Invalid email address.";
}
```

Check password:

```php
if (strlen($password) < 8) {
    $error = "Password must contain at least 8 characters.";
}
```

Check confirmation:

```php
if ($password !== $confirmPassword) {
    $error = "Passwords do not match.";
}
```

---

# 19. Check Whether Email Already Exists

We should not allow:

```text
student@example.com
student@example.com
```

twice.

Use a prepared statement.

Example:

```php
$stmt = $conn->prepare(
    "SELECT id FROM users WHERE email = ?"
);

$stmt->bind_param("s", $email);

$stmt->execute();

$result = $stmt->get_result();
```

Check:

```php
if ($result->num_rows > 0) {
    $error = "Email already registered.";
}
```

---

# 20. Why Prepared Statements?

Never build SQL like:

```php
$sql = "SELECT * FROM users WHERE email = '$email'";
```

This can introduce SQL injection.

Instead:

```php
$stmt = $conn->prepare(
    "SELECT * FROM users WHERE email = ?"
);
```

Then:

```php
$stmt->bind_param("s", $email);
```

The `?` is a placeholder.

Conceptually:

```text
SQL
 |
 |---- placeholder ?
 |
User input
 |
 |---- safely bound
```

---

# 21. Password Hashing

Use:

```php
$passwordHash = password_hash(
    $password,
    PASSWORD_DEFAULT
);
```

Then store:

```text
$passwordHash
```

not:

```text
$password
```

---

# 22. Insert New User

Example:

```php
$stmt = $conn->prepare(
    "INSERT INTO users (name, email, password_hash)
     VALUES (?, ?, ?)"
);

$stmt->bind_param(
    "sss",
    $name,
    $email,
    $passwordHash
);

$stmt->execute();
```

After successful signup:

```php
header("Location: login.php");
exit;
```

Important:

Always use:

```php
exit;
```

after a redirect.

---

# 23. Complete Signup Flow

The complete process is:

```text
Browser
   |
   | POST
   ↓
signup.php
   |
   ├── Validate name
   ├── Validate email
   ├── Validate password
   ├── Compare passwords
   ├── Check duplicate email
   ├── Hash password
   ├── INSERT database
   |
   ↓
login.php
```

---

# 24. Login Page

File:

```text
/var/www/dev/public/login.php
```

Login form:

```html
<form method="POST">

    <label>Email</label>
    <input type="email" name="email">

    <label>Password</label>
    <input type="password" name="password">

    <button type="submit">Login</button>

</form>
```

---

# 25. Login Process

When user submits:

```text
Email
Password
```

PHP should:

```text
Receive email/password
        ↓
Find user by email
        ↓
Retrieve password_hash
        ↓
password_verify()
        ↓
Correct?
   /          \
 YES          NO
  |            |
Session       Error
  |
Dashboard
```

---

# 26. Find User During Login

Use:

```php
$stmt = $conn->prepare(
    "SELECT id, name, email, password_hash
     FROM users
     WHERE email = ?"
);

$stmt->bind_param("s", $email);

$stmt->execute();

$result = $stmt->get_result();

$user = $result->fetch_assoc();
```

---

# 27. Verify Password

Do NOT do:

```php
if ($password === $user['password_hash'])
```

Instead:

```php
if (password_verify(
    $password,
    $user['password_hash']
)) {
    // correct password
}
```

PHP handles the password hash verification.

---

# 28. What Happens During Login?

Suppose user enters:

```text
Email:
ali@example.com

Password:
mypassword123
```

Database contains:

```text
email:
ali@example.com

password_hash:
$2y$10$................
```

PHP performs:

```php
password_verify(
    "mypassword123",
    "$2y$10$................"
);
```

If correct:

```text
true
```

Otherwise:

```text
false
```

---

# 29. Introduce Sessions

Now we reach the most important concept.

HTTP is stateless.

For example:

```text
Request 1
Browser → PHP

Request 2
Browser → PHP

Request 3
Browser → PHP
```

PHP does not automatically know that all three requests came from the same logged-in user.

Sessions solve this problem.

---

# 30. What Is a PHP Session?

A session allows PHP to store information associated with a user.

Example:

```php
$_SESSION['user_id'] = 25;
```

Now PHP can remember:

```text
User ID = 25
```

during subsequent requests.

---

# 31. Start a Session

Before using:

```php
$_SESSION
```

we need:

```php
session_start();
```

Example:

```php
<?php

session_start();

$_SESSION['user_id'] = 25;
```

---

# 32. Important Rule

`session_start()` must happen before output.

Bad:

```php
echo "Hello";

session_start();
```

This can cause:

```text
Headers already sent
```

Correct:

```php
<?php

session_start();

echo "Hello";
```

---

# 33. Create `session.php`

File:

```text
/var/www/dev/includes/session.php
```

Basic version:

```php
<?php

session_start();
```

Then pages can use:

```php
require_once __DIR__ . '/../includes/session.php';
```

This avoids writing session-starting code repeatedly.

---

# 34. Session Data After Successful Login

After password verification:

```php
session_start();

$_SESSION['user_id'] = $user['id'];
$_SESSION['user_name'] = $user['name'];
$_SESSION['user_email'] = $user['email'];
```

Now the session contains information about the logged-in user.

---

# 35. Session Fixation

A very important security issue is session fixation.

An attacker may try to make a victim use a session ID known to the attacker.

Therefore, after successful authentication, regenerate the session ID.

Use:

```php
session_regenerate_id(true);
```

Then create authentication session variables:

```php
session_regenerate_id(true);

$_SESSION['user_id'] = $user['id'];
$_SESSION['user_name'] = $user['name'];
$_SESSION['user_email'] = $user['email'];
```

Important concept:

```text
Before login
      |
Existing session
      |
Successful login
      |
session_regenerate_id()
      |
New session ID
      |
Authenticated session
```

---

# 36. Login Success

The login process should therefore look like:

```php
if (password_verify(
    $password,
    $user['password_hash']
)) {

    session_regenerate_id(true);

    $_SESSION['user_id'] = $user['id'];
    $_SESSION['user_name'] = $user['name'];
    $_SESSION['user_email'] = $user['email'];

    header("Location: dashboard.php");
    exit;
}
```

---

# 37. Protect the Dashboard

File:

```text
/var/www/dev/public/dashboard.php
```

At the top:

```php
<?php

require_once __DIR__ . '/../includes/session.php';

if (!isset($_SESSION['user_id'])) {

    header("Location: login.php");
    exit;
}
```

Now only authenticated users can access the dashboard.

---

# 38. Why Is This Important?

Without authentication protection:

```text
http://dev.local/dashboard.php
```

could be opened by anyone.

With authentication protection:

```text
Not logged in
      ↓
dashboard.php
      ↓
login.php
```

Logged in:

```text
Logged in
    ↓
dashboard.php
```

---

# 39. Create `auth.php`

File:

```text
/var/www/dev/includes/auth.php
```

This file can contain the reusable authentication check.

Concept:

```php
<?php

require_once __DIR__ . '/session.php';

if (!isset($_SESSION['user_id'])) {

    header("Location: /login.php");
    exit;
}
```

However, because the application structure separates `public` and `includes`, use the appropriate URL/path for your application.

Then protected pages can simply include:

```php
require_once __DIR__ . '/../includes/auth.php';
```

---

# 40. Dashboard

The dashboard can display:

```php
<h1>Dashboard</h1>

<p>
Welcome,
<?php echo htmlspecialchars($_SESSION['user_name']); ?>
</p>
```

Use:

```php
htmlspecialchars()
```

when displaying user-controlled data in HTML.

---

# 41. Why `htmlspecialchars()`?

Suppose a user enters:

```text
<script>alert('Hello')</script>
```

If we directly output it:

```php
echo $name;
```

it could become HTML/JavaScript.

Instead:

```php
echo htmlspecialchars($name);
```

The browser treats it as text.

This helps prevent Cross-Site Scripting (XSS).

---

# 42. Profile Page

File:

```text
/var/www/dev/public/profile.php
```

Protect it:

```php
require_once __DIR__ . '/../includes/auth.php';
```

Retrieve the current user's ID:

```php
$userId = $_SESSION['user_id'];
```

Then retrieve the user from the database:

```sql
SELECT id, name, email, created_at
FROM users
WHERE id = ?
```

Use a prepared statement.

The page can display:

```text
Name
Email
Account creation date
```

---

# 43. Important Concept: Session vs Database

Students often confuse these.

Database:

```text
Long-term storage
```

Session:

```text
Temporary information about the current browser/user session
```

For example:

Database:

```text
users
--------------------------------
id
name
email
password_hash
created_at
```

Session:

```text
$_SESSION['user_id']
$_SESSION['user_name']
$_SESSION['user_email']
```

The session does NOT replace the database.

---

# 44. Logout

File:

```text
/var/www/dev/public/logout.php
```

Logout should:

1. Start session.
2. Remove session variables.
3. Remove session cookie if appropriate.
4. Destroy session.
5. Redirect to login.

Basic:

```php
<?php

session_start();

$_SESSION = [];

session_destroy();

header("Location: login.php");
exit;
```

---

# 45. Why Use `$_SESSION = []`?

This:

```php
session_destroy();
```

destroys the server-side session data.

But clearing:

```php
$_SESSION = [];
```

also removes the current session variables from the PHP request.

Therefore use both.

---

# 46. More Complete Logout

A more complete logout implementation can also clear the session cookie:

```php
<?php

session_start();

$_SESSION = [];

if (ini_get("session.use_cookies")) {

    $params = session_get_cookie_params();

    setcookie(
        session_name(),
        '',
        time() - 42000,
        $params["path"],
        $params["domain"],
        $params["secure"],
        $params["httponly"]
    );
}

session_destroy();

header("Location: login.php");
exit;
```

This teaches students that logout is more than simply redirecting to another page.

---

# 47. Session Cookie

PHP sessions normally use a cookie in the browser.

Conceptually:

```text
Browser
   |
   | Session Cookie
   |
   ↓
Apache/PHP
   |
   | Session ID
   |
   ↓
Server-side Session
```

The browser does not normally store the complete `$_SESSION` array.

Instead, it stores the session identifier.

---

# 48. Important Session Settings

Important PHP session settings include:

```text
session.use_strict_mode
session.use_cookies
session.cookie_httponly
session.cookie_secure
session.cookie_samesite
session.gc_maxlifetime
```

---

# 49. `session.cookie_httponly`

Recommended:

```ini
session.cookie_httponly = 1
```

This tells the browser that JavaScript should not normally access the session cookie.

Concept:

```text
PHP Session Cookie
       |
       ├── Browser sends it to server
       |
       └── JavaScript cannot normally read it
```

This helps reduce some cookie theft risks through client-side scripts.

---

# 50. `session.cookie_secure`

Example:

```ini
session.cookie_secure = 1
```

This means the browser should send the cookie only over HTTPS.

Important:

For local development using:

```text
http://dev.local
```

do not enable this blindly.

If the application is using plain HTTP, a Secure cookie will not be sent over that HTTP connection.

In production:

```text
HTTPS
+
Secure cookies
```

should normally be used.

---

# 51. `session.cookie_samesite`

A useful setting is:

```ini
session.cookie_samesite = Lax
```

This provides protection against some cross-site request scenarios while still allowing normal navigation.

Students should understand:

```text
SameSite
    ↓
controls when browser sends cookies
```

---

# 52. `session.use_strict_mode`

Recommended:

```ini
session.use_strict_mode = 1
```

This helps PHP reject uninitialized session IDs rather than accepting arbitrary IDs.

---

# 53. Check Current Session Configuration

Use:

```bash
php -i | grep session
```

Or:

```bash
php --ini
```

You can also create a temporary PHP file:

```php
<?php

phpinfo();
```

Then search for:

```text
session
```

Remove the file after testing.

---

# 54. CLI vs Apache PHP Configuration

Important real-world issue:

```bash
php -i
```

shows CLI PHP configuration.

Apache may use a different PHP configuration depending on the setup.

Therefore:

```text
CLI PHP
```

and:

```text
Apache PHP
```

should not automatically be assumed to use identical configuration.

Students should learn to verify the configuration used by the actual web application.

---

# 55. Where Should Session Settings Be Configured?

There are several possibilities.

### php.ini

Global PHP configuration.

Example:

```ini
session.use_strict_mode = 1
session.cookie_httponly = 1
session.cookie_samesite = Lax
```

### Runtime PHP

Some settings can be changed with:

```php
ini_set(...)
```

### Application-level configuration

Some session behavior can be controlled in application code.

For classroom purposes, explain the difference between:

```text
Global configuration
        ↓
php.ini

Application configuration
        ↓
PHP code
```

---

# 56. Session Lifetime

PHP sessions have garbage collection settings.

One important setting is:

```ini
session.gc_maxlifetime
```

It controls how long session data may remain before PHP's garbage collection considers it eligible for cleanup.

Do not teach this as:

> "The user will always be logged out exactly after this many seconds."

That is not necessarily how PHP session garbage collection works.

---

# 57. Test the Complete Application

Students should test this sequence.

### Test 1

Open:

```text
http://dev.local/signup.php
```

Create:

```text
Name: Ali
Email: ali@example.com
Password: password123
```

---

### Test 2

Check database:

```sql
USE auth_demo;

SELECT id, name, email, password_hash, created_at
FROM users;
```

The password should NOT appear as:

```text
password123
```

It should be a hash.

---

### Test 3

Go to:

```text
http://dev.local/login.php
```

Use correct credentials.

Expected:

```text
Login successful
→ dashboard.php
```

---

### Test 4

Open another browser/private window.

Try:

```text
http://dev.local/dashboard.php
```

Expected:

```text
Redirect → login.php
```

---

### Test 5

Login.

Then:

```text
logout.php
```

Expected:

```text
Session destroyed
→ login.php
```

---

### Test 6

After logout, try:

```text
http://dev.local/dashboard.php
```

Expected:

```text
Redirect → login.php
```

---

# 58. Test Invalid Password

Suppose the correct password is:

```text
password123
```

Try:

```text
wrongpassword
```

Expected:

```text
Invalid email or password.
```

Do not tell the attacker:

```text
Email exists but password is wrong.
```

A generic authentication error is preferable.

---

# 59. Test Duplicate Email

Create:

```text
ali@example.com
```

again.

Expected:

```text
Email already registered.
```

This demonstrates why the database has:

```sql
UNIQUE
```

on email.

---

# 60. Test SQL Injection

Students should understand why this is dangerous.

A malicious user might try input such as:

```text
' OR '1'='1
```

The application should not authenticate the user.

Prepared statements help protect against this class of SQL injection.

---

# 61. Test XSS

For example, enter a name like:

```html
<script>alert('XSS')</script>
```

Then display it using:

```php
htmlspecialchars()
```

The browser should display the text instead of executing it as JavaScript.

---

# 62. Common Problem: "Headers Already Sent"

Students may see:

```text
Cannot modify header information -
headers already sent
```

This usually happens when PHP sends output before:

```php
header(...)
```

or:

```php
session_start()
```

Bad:

```php
echo "Hello";

session_start();
```

Bad:

```php
<html>
<?php
session_start();
?>
```

Correct:

```php
<?php

session_start();

header("Location: login.php");
exit;
```

---

# 63. Common Problem: Login Works but Dashboard Redirects to Login

Possible causes:

```text
session_start() missing
```

or:

```text
$_SESSION['user_id'] not set
```

or:

```text
session cookie not being stored
```

or:

```text
different session configuration
```

Debug:

```php
session_start();

var_dump($_SESSION);
```

Use this only during development.

Remove debugging output afterward.

---

# 64. Common Problem: Session Does Not Persist

Check:

1. Is `session_start()` present?
2. Is it before output?
3. Is the browser accepting cookies?
4. Is the session cookie being created?
5. Are session settings correct?
6. Is the application using the expected PHP configuration?
7. Are redirects occurring correctly?

Browser developer tools can help inspect cookies.

---

# 65. Common Problem: Redirect Does Not Work

Check:

```php
header("Location: login.php");
exit;
```

Make sure there is no output before `header()`.

Also verify the URL/path.

---

# 66. Common Problem: Database Connection Fails

Check:

```text
MySQL running?
Database exists?
Username correct?
Password correct?
Database name correct?
PHP MySQL extension installed?
```

Check MySQL:

```bash
sudo systemctl status mysql
```

Check PHP:

```bash
php -m | grep mysqli
```

---

# 67. Common Problem: PHP Error Not Visible

During development, errors can be enabled temporarily.

Example:

```php
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);
```

Do not expose detailed PHP errors to users in production.

Production should log errors rather than display sensitive internal details.

---

# 68. Apache Logs

Check Apache error log:

```bash
sudo tail -f /var/log/apache2/error.log
```

If your VirtualHost has its own error log, use that file instead.

Apache access log:

```bash
sudo tail -f /var/log/apache2/access.log
```

These are very useful when debugging PHP applications.

---

# 69. Authentication Architecture

At this point students should understand:

```text
                    Browser
                       |
                       |
                       ↓
                  Apache/PHP
                       |
          +------------+------------+
          |                         |
          ↓                         ↓
       Session                  MySQL
          |                         |
          |                    users table
          |                         |
          +------------+------------+
                       |
                       ↓
                  Application
```

---

# 70. Complete Authentication Flow

```text
                  START
                    |
                    ↓
              Signup Page
                    |
                    ↓
             Enter User Data
                    |
                    ↓
               Validation
                    |
                    ↓
           Check Existing Email
                    |
                    ↓
             Hash Password
                    |
                    ↓
              Insert User
                    |
                    ↓
               Login Page
                    |
                    ↓
          Enter Email + Password
                    |
                    ↓
             Find User
                    |
                    ↓
          password_verify()
                    |
              +-----+-----+
              |           |
            FAIL        SUCCESS
              |           |
              ↓           ↓
          Show Error   Regenerate ID
                          |
                          ↓
                     Create Session
                          |
                          ↓
                       Dashboard
                          |
                          ↓
                        Logout
                          |
                          ↓
                  Destroy Session
                          |
                          ↓
                       Login
```

---

# 71. Final Project Structure

The final project should look like:

```text
/var/www/dev/
│
├── config/
│   └── database.php
│
├── includes/
│   ├── auth.php
│   ├── footer.php
│   ├── header.php
│   └── session.php
│
└── public/
    ├── dashboard.php
    ├── index.php
    ├── login.php
    ├── logout.php
    ├── profile.php
    └── signup.php
```

---

# 72. Recommended Responsibility of Each File

| File            | Responsibility              |
| --------------- | --------------------------- |
| `index.php`     | Application home page       |
| `signup.php`    | Registration                |
| `login.php`     | Login                       |
| `logout.php`    | Logout                      |
| `dashboard.php` | Protected dashboard         |
| `profile.php`   | Protected user profile      |
| `database.php`  | MySQL connection            |
| `session.php`   | Start/configure session     |
| `auth.php`      | Protect authenticated pages |
| `header.php`    | Common HTML header          |
| `footer.php`    | Common HTML footer          |

---

# 73. Security Checklist

Before considering the application complete:

* [ ] Passwords are hashed.
* [ ] `password_hash()` is used.
* [ ] `password_verify()` is used.
* [ ] SQL prepared statements are used.
* [ ] Email has a UNIQUE constraint.
* [ ] User input is validated.
* [ ] Output is escaped using `htmlspecialchars()`.
* [ ] Sessions are started correctly.
* [ ] Session ID is regenerated after login.
* [ ] Protected pages check authentication.
* [ ] Logout destroys the session.
* [ ] Session cookies use appropriate security settings.
* [ ] HTTPS is used in production.
* [ ] Database credentials are not exposed publicly.
* [ ] Debug information is not shown in production.
* [ ] Temporary test files are removed.

---

# 74. Important Security Concepts Students Should Know

At the end of this practical, students should know the difference between:

### Authentication

"Who are you?"

Example:

```text
Login
```

### Authorization

"What are you allowed to do?"

Example:

```text
Admin can delete users.
Normal user cannot.
```

### Session

"How does the server remember that you are logged in?"

### Password hashing

"How do we safely store passwords?"

### SQL injection

"How can malicious SQL input attack the database?"

### XSS

"How can malicious scripts be injected into web pages?"

### Session fixation

"How can attackers abuse session identifiers?"

---

# 75. Authentication vs Authorization

This distinction is extremely important.

Suppose:

```text
Ali logs in.
```

Authentication answers:

```text
Is Ali really authenticated?
```

Authorization answers:

```text
Can Ali access the admin panel?
```

Therefore:

```text
Authentication
       ↓
Who are you?

Authorization
       ↓
What can you access?
```

We will later extend our application with roles.

For example:

```text
users
--------------------
id
name
email
password_hash
role
```

Possible roles:

```text
user
admin
```

---

# 76. Suggested Classroom Exercises

## Exercise 1 — Signup

Create a signup page that:

* Accepts name.
* Accepts email.
* Accepts password.
* Confirms password.
* Validates input.
* Hashes password.
* Saves user to MySQL.

---

## Exercise 2 — Login

Create a login page that:

* Accepts email.
* Accepts password.
* Finds the user.
* Verifies password.
* Creates a session.
* Redirects to dashboard.

---

## Exercise 3 — Protected Page

Create:

```text
dashboard.php
```

Only authenticated users should access it.

---

## Exercise 4 — Logout

Create:

```text
logout.php
```

After logout, the user must not be able to access:

```text
dashboard.php
```

using the previous session.

---

## Exercise 5 — Profile

Create:

```text
profile.php
```

Display:

```text
Name
Email
Account creation date
```

Retrieve the information from MySQL.

Do not simply trust session data for all profile information.

---

# 77. Debugging Exercise

Give students these problems intentionally:

### Problem A

Remove:

```php
session_start();
```

Ask:

> Why does authentication fail?

---

### Problem B

Remove:

```php
session_regenerate_id(true);
```

Ask:

> What security protection have we removed?

---

### Problem C

Store the password directly.

Ask:

> Why is this dangerous?

---

### Problem D

Use SQL string concatenation.

Ask:

> What security vulnerability can this create?

---

### Problem E

Remove:

```php
htmlspecialchars()
```

Ask:

> What happens when a malicious HTML/script value is displayed?

---

### Problem F

Remove:

```php
exit;
```

after a redirect.

Ask:

> Why is `exit` recommended after `header()`?

---

# 78. Browser Testing

Students should use browser developer tools.

Open:

```text
Developer Tools
→ Application/Storage
→ Cookies
```

Look for the PHP session cookie.

Depending on PHP configuration, it will commonly be named:

```text
PHPSESSID
```

Students should understand:

```text
Browser
   |
   | PHPSESSID
   ↓
Server
   |
   ↓
PHP session
```

---

# 79. Database Testing

Check registered users:

```sql
SELECT
    id,
    name,
    email,
    created_at
FROM users;
```

Do not normally display:

```text
password_hash
```

unless you are specifically demonstrating hashing.

To inspect it during the security lesson:

```sql
SELECT id, email, password_hash
FROM users;
```

Students should observe that the password is not stored as plain text.

---

# 80. Useful MySQL Commands

Show databases:

```sql
SHOW DATABASES;
```

Use database:

```sql
USE auth_demo;
```

Show tables:

```sql
SHOW TABLES;
```

Describe table:

```sql
DESCRIBE users;
```

Show table creation:

```sql
SHOW CREATE TABLE users;
```

Query users:

```sql
SELECT * FROM users;
```

Delete a test user:

```sql
DELETE FROM users
WHERE email = 'ali@example.com';
```

Count users:

```sql
SELECT COUNT(*) FROM users;
```

---

# 81. Useful Linux Commands

Check Apache:

```bash
sudo systemctl status apache2
```

Check MySQL:

```bash
sudo systemctl status mysql
```

Check PHP:

```bash
php -v
```

Check PHP modules:

```bash
php -m
```

Check MySQL extension:

```bash
php -m | grep mysqli
```

Check Apache errors:

```bash
sudo tail -f /var/log/apache2/error.log
```

Check project:

```bash
tree /var/www/dev
```

---

# 82. Real-World Lessons

Students should understand that authentication is not simply:

```php
if ($password == $databasePassword)
```

A real authentication system requires:

```text
Input validation
       +
Database
       +
Password hashing
       +
Password verification
       +
Sessions
       +
Session security
       +
Authorization
       +
Output escaping
       +
SQL injection protection
       +
Error handling
       +
Cookie security
```

---

# 83. Final Mental Model

The most important concept students should remember:

```text
                USER
                 |
                 ↓
              SIGNUP
                 |
                 ↓
          Password Hashing
                 |
                 ↓
              MySQL
                 |
                 ↓
               LOGIN
                 |
                 ↓
        Password Verification
                 |
                 ↓
       Session ID Generated
                 |
                 ↓
           $_SESSION
                 |
                 ↓
       Protected Application
                 |
                 ↓
              LOGOUT
                 |
                 ↓
        Session Destroyed
```

---


# 86. Final Questions for Students

At the end of the lab, students should be able to answer:

1. What is authentication?
2. What is authorization?
3. Why do we need sessions?
4. What does `session_start()` do?
5. Where is the PHP session ID stored?
6. What is `PHPSESSID`?
7. Why should passwords not be stored directly?
8. What does `password_hash()` do?
9. What does `password_verify()` do?
10. Why do we use prepared statements?
11. What is SQL injection?
12. Why do we use `htmlspecialchars()`?
13. What is XSS?
14. Why do we use `session_regenerate_id(true)`?
15. What happens during logout?
16. What is `session.cookie_httponly`?
17. What is `session.cookie_secure`?
18. What is `session.cookie_samesite`?
19. Why should `session.use_strict_mode` be enabled?
20. Why should database configuration be outside `public/`?
21. Why should production applications use HTTPS?
22. What is the difference between authentication and authorization?
23. Why should we not use MySQL root credentials in an application?
24. Why should we not use `chmod 777`?
25. Why should detailed PHP errors not be displayed in production?

---
