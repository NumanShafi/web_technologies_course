# PHP and MySQL: Authentication, Authorization, and Role-Based Access Control (RBAC)

**Course:** Web Technologies / LAMP  
**Environment:** Ubuntu, Apache, PHP, MySQL  
**Local website:** `http://dev.local`  
**Project directory:** `/var/www/dev`  
**Public web directory:** `/var/www/dev/public`

---

## 1. Welcome to the next stage of our PHP application

In the previous lessons, we built the foundations of a PHP web application:

- A signup page that stores a user's account in MySQL.
- A login page that checks the user's email and password.
- Password hashing with `password_hash()`.
- Password verification with `password_verify()`.
- Prepared SQL statements to reduce SQL-injection risk.
- PHP sessions to remember a logged-in user.
- A protected dashboard.
- Logout functionality.

Now imagine that our application is used by a university.

A student logs in and sees their own dashboard. An administrator logs in and sees a page for managing user accounts. Both people have successfully logged in, but they should not have the same permissions.

That is the problem we will solve in this lesson.

### Learning outcomes

By the end of this practical lesson, you should be able to:

1. Explain authentication and authorization in your own words.
2. Add a role to each user account in MySQL.
3. store the logged-in user's identity and role in a PHP session.
4. Protect pages so only permitted users can open them.
5. Build an administrator-only page that lists users.
6. Prevent a user from changing another user's profile by changing a URL.
7. Test the application using student and administrator accounts.
8. Explain why hiding a button is not sufficient security.

> **Important:** This lesson extends the existing `dev.local` project. Do not create another Apache virtual host if the existing one already points to `/var/www/dev/public`.

---

## 2. Authentication vs. authorization

These concepts are related, but they answer different questions.

| Concept | Question | Example |
|---|---|---|
| Authentication | Who are you? | The user signs in with their email and password. |
| Authorization | What are you allowed to do? | A student can view their profile, but only an administrator can manage all users. |

Think of a university building:

- **Authentication:** The security guard checks your identity card.
- **Authorization:** Your card determines whether you may enter the library, staff room, or server room.

Logging in successfully does **not** automatically mean that a person is allowed to perform every action.

### A small scenario

Our application has two roles:

- `student`: can access the normal dashboard and their own profile.
- `admin`: can access the normal dashboard and the user-management page.

A visitor who has not logged in should not access either protected area.

We will enforce these rules on the **server**, using PHP. A user can edit HTML or manually type a URL, so the server must check permissions every time a protected page is requested.

---

## 3. Check the existing project

Open a terminal and inspect the project:

```bash
ls -la /var/www/dev
ls -la /var/www/dev/public
```

Your project may contain files such as:

```text
/var/www/dev/
├── config/
│   └── database.php
├── includes/
│   └── ...
└── public/
    ├── index.php
    ├── signup.php
    ├── login.php
    ├── dashboard.php
    └── logout.php
```

The exact filenames in your existing project may differ. **Keep your working signup and login pages.** We will add a few new files and make small changes to existing ones.

Check that the website still opens:

```text
http://dev.local
```

If your virtual host is already working, no new virtual-host configuration is needed.

### Folder responsibilities

- `public/`: files the browser is allowed to request directly.
- `config/`: database configuration and other application settings.
- `includes/`: reusable PHP functions and shared page components.

Keeping configuration outside `public/` helps prevent visitors from requesting those files directly through the web server.

---

## 4. Step 1 — Add a role to the users table

Our database already contains a `users` table from the signup/login lesson. We will add a `role` column rather than create a second users table.

Open MySQL using the method you normally use in your lab. For example:

```bash
sudo mysql
```

Select the database used by your PHP application. In this example, the database is called `devdb`:

```sql
USE devdb;
DESCRIBE users;
```

Inspect the result. We expect a table with fields similar to:

- `id`: unique identifier for each user.
- `name`: user's display name.
- `email`: user's email address.
- `password`: password hash, not the original password.

Your existing table might use different names, such as `full_name` or `password_hash`. If so, keep your real column names and adjust the PHP examples accordingly.

Add the role column:

```sql
ALTER TABLE users
ADD COLUMN role VARCHAR(20) NOT NULL DEFAULT 'student';
```

Check the result:

```sql
DESCRIBE users;

SELECT id, name, email, role
FROM users;
```

All existing users should now have the role `student`, because that is the default.

### Why use a default role?

A newly registered account must not become an administrator automatically. If signup allows a visitor to submit `role=admin`, anyone could create an administrator account.

Therefore:

- Public signup should create ordinary accounts only.
- The database default should be `student`.
- An administrator role should be assigned only through a trusted administrative process.

**Never trust a role sent by a browser form.**

### Create a test administrator

For a classroom exercise, we need one administrator account. First, register a normal account through your existing signup page. Then identify that account:

```sql
SELECT id, name, email, role
FROM users;
```

Use the account's actual email address in the following statement:

```sql
UPDATE users
SET role = 'admin'
WHERE email = 'admin@example.com';
```

Replace `admin@example.com` with the email of the account you just registered. Verify the change:

```sql
SELECT id, name, email, role
FROM users;
```

Only the selected test account should show `admin`.

> In a real production system, administrator provisioning should be performed through a controlled process. Do not provide a public signup form that lets users choose their own role.

Exit MySQL:

```sql
EXIT;
```

---

## 5. Step 2 — Review the database connection

The examples below assume that the project has a database connection file at:

```text
/var/www/dev/config/database.php
```

Use your existing connection file if you already have one. Do not create multiple competing database connections just for this lesson.

For reference, a PDO connection file can look like this:

```php
<?php
// /var/www/dev/config/database.php

$host = 'localhost';
$dbname = 'devdb';
$username = 'devuser';
$password = 'CHANGE_THIS_TO_YOUR_DATABASE_PASSWORD';

$dsn = "mysql:host={$host};dbname={$dbname};charset=utf8mb4";

$options = [
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    PDO::ATTR_EMULATE_PREPARES   => false,
];

$pdo = new PDO($dsn, $username, $password, $options);
```

DSN: Stands for Data Source Name.
PDO: Stands for PHP Data Objects.

<img width="871" height="775" alt="image" src="https://github.com/user-attachments/assets/600b248f-e281-4fe2-9853-4d3f0c9372cc" />


Update the username and password to match your local development database. Do not copy a placeholder password literally.

If your existing application uses `mysqli` rather than PDO, either adapt the examples to `mysqli` or use the same database-access style consistently throughout your project. Do not mix connection variables without understanding which file creates them.

### Keep credentials private

The database configuration belongs outside the public web directory. Do not print database passwords on a page or commit real production credentials to a public Git repository.

---

## 6. Step 3 — Create reusable authentication helpers

We do not want to repeat the same login and role-checking logic in every PHP page. We will put the shared functions in one file.

Create the file:

```bash
sudo nano /var/www/dev/includes/auth.php
```

Add:

```php
<?php
// /var/www/dev/includes/auth.php

if (session_status() !== PHP_SESSION_ACTIVE) {
    // Use secure cookie settings when the site is served over HTTPS.
    // For this local HTTP-only exercise, secure is false.
    session_set_cookie_params([
        'httponly' => true,
        'secure'   => false,
        'samesite' => 'Lax',
    ]);

    session_start();
}

/**
 * Return true if a user is logged in.
 */
function isLoggedIn(): bool
{
    return isset($_SESSION['user_id']);
}

/**
 * Require a logged-in user.
 * Redirect visitors who are not signed in to the login page.
 */
function requireLogin(): void
{
    if (!isLoggedIn()) {
        header('Location: /login.php');
        exit;
    }
}

/**
 * Require a particular role.
 * A user who is logged in but has the wrong role receives HTTP 403.
 */
function requireRole(string $requiredRole): void
{
    requireLogin();

    if (($_SESSION['role'] ?? '') !== $requiredRole) {
        http_response_code(403);
        exit('403 Forbidden: You do not have permission to access this page.');
    }
}

/**
 * Escape text before displaying it in HTML.
 */
function e(?string $value): string
{
    return htmlspecialchars($value ?? '', ENT_QUOTES, 'UTF-8');
}
```

Save the file.

### Understand the important parts

**`session_start()`**

Loads the session so PHP can read the current user's session data. It must run before PHP sends page output.

**`isLoggedIn()`**

Checks whether the session contains a user ID. A session is not the same thing as a database password; it is how our application remembers the authenticated user between requests.

**`requireLogin()`**

Stops a visitor from continuing to a protected page if they are not logged in.

**`requireRole('admin')`**

First requires login, then checks the user's role. A student should receive a `403 Forbidden` response if they request an administrator-only page.

**`exit`**

Stops execution. Without it, PHP could redirect a visitor and then continue executing the protected page.

**`e()`**

Escapes text before placing it into HTML. This helps prevent user-supplied text from being interpreted as HTML or JavaScript.

### Important session note

The `secure` cookie option is `false` only because this lesson uses a local HTTP address. On a real HTTPS deployment, set it to `true`. If you already have a shared, carefully configured session bootstrap, use that instead of configuring sessions in multiple places.

---

## 7. Step 4 — Update login to save the user's role

Your login page should already verify the password hash. We only need to make sure the query also retrieves the user's role and the session stores it.

The following is a complete example using PDO. If your existing login page has a different layout, preserve your HTML and adapt the PHP logic.

Create or update:

```text
/var/www/dev/public/login.php
```

```php
<?php
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth.php';

$error = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $email = trim($_POST['email'] ?? '');
    $password = $_POST['password'] ?? '';

    if ($email === '' || $password === '') {
        $error = 'Please enter both email and password.';
    } else {
        $stmt = $pdo->prepare(
            'SELECT id, name, email, password, role
             FROM users
             WHERE email = :email
             LIMIT 1'
        );

        $stmt->execute(['email' => $email]);
        $user = $stmt->fetch();

        if ($user && password_verify($password, $user['password'])) {
            // Prevent session fixation after successful authentication.
            session_regenerate_id(true);

            $_SESSION['user_id'] = (int) $user['id'];
            $_SESSION['name'] = $user['name'];
            $_SESSION['email'] = $user['email'];
            $_SESSION['role'] = $user['role'];

            header('Location: /dashboard.php');
            exit;
        }

        // Keep the message generic; do not reveal which field was incorrect.
        $error = 'Invalid email or password.';
    }
}
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Login - Dev Portal</title>
</head>
<body>
    <h1>Login</h1>

    <?php if ($error !== ''): ?>
        <p role="alert"><?= e($error) ?></p>
    <?php endif; ?>

    <form method="post" action="/login.php">
        <label for="email">Email</label>
        <input
            type="email"
            id="email"
            name="email"
            autocomplete="username"
            required
        >

        <br><br>

        <label for="password">Password</label>
        <input
            type="password"
            id="password"
            name="password"
            autocomplete="current-password"
            required
        >

        <br><br>

        <button type="submit">Log in</button>
    </form>

    <p><a href="/signup.php">Create an account</a></p>
</body>
</html>
```

### Why does the query retrieve `role`?

This part retrieves the role from the database:

```sql
SELECT id, name, email, password, role
FROM users
WHERE email = :email
LIMIT 1
```

After the password is verified, the application stores the role in the session:

```php
$_SESSION['role'] = $user['role'];
```

The role is loaded from the database, not from a form field supplied by the browser.

### Check your password column

This example assumes the password hash is stored in a column called `password`. If your schema uses `password_hash`, change both the SQL selection and the `password_verify()` reference to use that actual column.

Do not store plain-text passwords. Existing accounts should continue to use the hashes created by `password_hash()`.

---

## 8. Step 5 — Protect the dashboard

A dashboard should not be accessible to a visitor who has not logged in.

Create or update:

```text
/var/www/dev/public/dashboard.php
```

```php
<?php
require_once __DIR__ . '/../includes/auth.php';

requireLogin();

$name = $_SESSION['name'] ?? 'User';
$email = $_SESSION['email'] ?? '';
$role = $_SESSION['role'] ?? 'student';
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Dashboard - Dev Portal</title>
</head>
<body>
    <h1>Welcome, <?= e($name) ?>!</h1>

    <p>You are logged in as <strong><?= e($role) ?></strong>.</p>
    <p>Email: <?= e($email) ?></p>

    <h2>Your dashboard</h2>
    <p>This page is available to every authenticated user.</p>

    <ul>
        <li><a href="/profile.php">My profile</a></li>

        <?php if ($role === 'admin'): ?>
            <li><a href="/admin/users.php">Manage users (admin only)</a></li>
        <?php endif; ?>
    </ul>

    <p><a href="/logout.php">Log out</a></p>
</body>
</html>
```

Notice the conditional navigation:

```php
<?php if ($role === 'admin'): ?>
    <li><a href="/admin/users.php">Manage users</a></li>
<?php endif; ?>
```

This makes the interface friendlier because students do not see the admin link.

**But this condition is not the security boundary.** A student can manually type `/admin/users.php` into the address bar. The admin page itself must check the role.

---

## 9. Step 6 — Build an administrator-only user list

Create an admin folder and file:

```bash
sudo mkdir -p /var/www/dev/public/admin
sudo nano /var/www/dev/public/admin/users.php
```

Add the following:

```php
<?php
require_once __DIR__ . '/../../config/database.php';
require_once __DIR__ . '/../../includes/auth.php';

// This check happens on the server before any user data is displayed.
requireRole('admin');

$stmt = $pdo->query(
    'SELECT id, name, email, role
     FROM users
     ORDER BY id DESC'
);

$users = $stmt->fetchAll();
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Manage Users - Dev Portal</title>
</head>
<body>
    <h1>Manage Users</h1>
    <p>Only administrators should be able to view this page.</p>

    <p><a href="/dashboard.php">Back to dashboard</a></p>

    <?php if (!$users): ?>
        <p>No users were found.</p>
    <?php else: ?>
        <table border="1" cellpadding="8">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Name</th>
                    <th>Email</th>
                    <th>Role</th>
                </tr>
            </thead>
            <tbody>
                <?php foreach ($users as $user): ?>
                    <tr>
                        <td><?= (int) $user['id'] ?></td>
                        <td><?= e($user['name']) ?></td>
                        <td><?= e($user['email']) ?></td>
                        <td><?= e($user['role']) ?></td>
                    </tr>
                <?php endforeach; ?>
            </tbody>
        </table>
    <?php endif; ?>
</body>
</html>
```

### What makes this page administrator-only?

This line is the key:

```php
requireRole('admin');
```

The permission check runs before the query that retrieves user records. If a student requests this page, PHP stops execution and responds with `403 Forbidden`.

### Why use a prepared statement for the user list?

This particular query contains no user-supplied value, so a direct query is acceptable. Whenever values come from a request or user input, use a prepared statement instead of concatenating the value into SQL.

### Why escape the displayed fields?

Names and emails are user-supplied data. Escaping them with `e()` reduces the risk of stored cross-site scripting (XSS), where someone saves HTML or JavaScript in a field and it runs when another user views the page.

---

## 10. Step 7 — Let users view only their own profile

Authorization is not only about student vs. admin. It is also about **which record** a user is allowed to access.

Imagine that the profile URL is:

```text
http://dev.local/profile.php?id=12
```

If a student changes `12` to `13`, should they see another student's private information? No.

For a personal profile page, the safest simple design is to use the logged-in user's ID from the session, not an ID supplied in the URL.

Create:

```text
/var/www/dev/public/profile.php
```

```php
<?php
require_once __DIR__ . '/../config/database.php';
require_once __DIR__ . '/../includes/auth.php';

requireLogin();

$userId = (int) $_SESSION['user_id'];

$stmt = $pdo->prepare(
    'SELECT id, name, email, role
     FROM users
     WHERE id = :id
     LIMIT 1'
);

$stmt->execute(['id' => $userId]);
$user = $stmt->fetch();

if (!$user) {
    // The session refers to an account that no longer exists.
    $_SESSION = [];
    session_destroy();

    header('Location: /login.php');
    exit;
}
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>My Profile - Dev Portal</title>
</head>
<body>
    <h1>My Profile</h1>

    <p><strong>ID:</strong> <?= (int) $user['id'] ?></p>
    <p><strong>Name:</strong> <?= e($user['name']) ?></p>
    <p><strong>Email:</strong> <?= e($user['email']) ?></p>
    <p><strong>Role:</strong> <?= e($user['role']) ?></p>

    <p><a href="/dashboard.php">Back to dashboard</a></p>
</body>
</html>
```

### The central security idea

This is safer than trusting a URL parameter:

```php
$userId = (int) $_SESSION['user_id'];
```

The identity comes from the authenticated session. A user cannot choose another account simply by changing a query-string value.

If you later build an admin page that opens another user's profile, it should perform a separate admin authorization check before accessing that record.

---

## 11. Step 8 — Check logout behavior

Your existing logout page should clear the session and expire the session cookie. A simple example is:

```php
<?php
require_once __DIR__ . '/../includes/auth.php';

$_SESSION = [];

if (ini_get('session.use_cookies')) {
    $params = session_get_cookie_params();

    setcookie(
        session_name(),
        '',
        time() - 42000,
        $params['path'],
        $params['domain'],
        $params['secure'],
        $params['httponly']
    );
}

session_destroy();

header('Location: /login.php');
exit;
```

Save this as:

```text
/var/www/dev/public/logout.php
```

If your existing logout file already does this correctly, keep it rather than replacing it unnecessarily.

---

## 12. Step 9 — Make sure signup cannot assign roles

Open your existing signup page. The registration logic should insert the new account without accepting a role from the visitor.

A safe insert looks like this:

```php
$passwordHash = password_hash($password, PASSWORD_DEFAULT);

$stmt = $pdo->prepare(
    'INSERT INTO users (name, email, password)
     VALUES (:name, :email, :password)'
);

$stmt->execute([
    'name'     => $name,
    'email'    => $email,
    'password' => $passwordHash,
]);
```

Because the database column has the default value `student`, a newly created account receives the student role.

Do **not** do this:

```php
// UNSAFE DESIGN: do not accept the role from a public signup form.
$role = $_POST['role'];
```

Also do not add a public dropdown that lets people select `student` or `admin`. A malicious visitor can modify a request even if a form field is hidden.

### A useful classroom question

**Question:** If the signup page does not show an administrator option, can a visitor still try to submit `role=admin` manually?

**Answer:** Yes. The browser is controlled by the visitor. Security must be enforced by server-side code and database rules, not by the appearance of the form.

---

## 13. Step 10 — Test the application systematically

Use two separate accounts:

- **Account A:** an ordinary student account.
- **Account B:** the test administrator account whose role you changed in MySQL.

Open a private/incognito browser window for the second account, or log out before switching users. This avoids confusion caused by an existing session.

### Test checklist

- [ ] Open `/dashboard.php` while logged out. It should redirect to `/login.php`.
- [ ] Open `/admin/users.php` while logged out. It should redirect to `/login.php`.
- [ ] Log in as a student. The dashboard should display the `student` role.
- [ ] As a student, the admin navigation link should not be shown.
- [ ] As a student, manually open `/admin/users.php`. The server should return `403 Forbidden`.
- [ ] Log out and log in as the administrator.
- [ ] The dashboard should display the `admin` role.
- [ ] The administrator should see the user-management link.
- [ ] Open `/admin/users.php` as the administrator. The user list should appear.
- [ ] Open `/profile.php` as a student. It should display that student's profile.
- [ ] Log out and try to reopen the profile. You should be redirected to login.
- [ ] Register a brand-new account. Its role should be `student` by default.

Verify roles in MySQL when needed:

```sql
SELECT id, name, email, role
FROM users
ORDER BY id;
```

### Expected behavior

| Request | Logged-out visitor | Student | Admin |
|---|---|---|---|
| Dashboard | Redirect to login | Allowed | Allowed |
| Own profile | Redirect to login | Allowed | Allowed |
| Admin user list | Redirect to login | `403 Forbidden` | Allowed |
| Create account | Allowed | Allowed | Allowed |

This table describes our lesson's current design. A real application may place restrictions on signup, profile updates, and account management that are more specific.

---

## 14. Troubleshooting common problems

### Problem A: `Unknown column 'role'`

The role column was not added to the database selected by the PHP application.

Run:

```sql
USE devdb;
DESCRIBE users;
```

If `role` is missing, run the `ALTER TABLE` statement from Step 1. Confirm that PHP connects to the same database.

### Problem B: `Undefined array key "role"`

The login query may not select `role`, or the session assignment may be missing. Ensure that the query includes `role` and successful login runs:

```php
$_SESSION['role'] = $user['role'];
```

Log out and log in again after making this change, because the existing session may not contain the role.

### Problem C: Every user is a student

That is expected for existing accounts until an administrator is assigned manually for this classroom exercise. Verify the role in MySQL and update only the intended test account.

### Problem D: The admin link is missing

Check the database role and the session role. After changing a role in MySQL, log out and log in again so the login code loads the new role into the session.

### Problem E: A student can still open the admin page

Check that `/admin/users.php` includes the correct `auth.php` path and calls:

```php
requireRole('admin');
```

The check must execute before any sensitive query or output. Hiding the link in the dashboard is not enough.

### Problem F: `headers already sent`

PHP sent output before a redirect or session header. Ensure that PHP code calling `session_start()` or `header()` runs before HTML, whitespace, or debugging output.

### Problem G: Database connection errors

Confirm the database name, username, password, and privileges in your existing configuration file. Avoid displaying credentials or detailed connection errors to visitors.

---

## 15. A short security discussion

RBAC is a useful starting point, but it is not the whole of application security.

### Rule 1: Check permission on every protected request

A user may bypass the navigation menu and type a URL directly. Every sensitive page and action needs its own server-side permission check.

### Rule 2: Do not trust browser-submitted identity or role

The browser must not decide which user ID or role is trusted. Use the authenticated session for identity and trusted server-side logic for permissions.

### Rule 3: Protect actions, not just pages

If you later add buttons to delete accounts, change roles, or update records, check the administrator's permission in the PHP code that performs the action—not only on the page that displays the button.

### Rule 4: Validate and escape data

Use prepared statements for SQL values. Validate data before saving it, and escape output for the HTML context in which it is displayed.

### Rule 5: Use HTTPS in deployed applications

HTTPS protects data in transit and allows the session cookie to use the `Secure` attribute. Local development over HTTP is convenient, but it is not a model for a public production deployment.

### Rule 6: Be careful with role changes

A role stored in a session may remain unchanged until the user logs in again. For a small classroom app, logging in again after a role change is sufficient for the exercise. A larger application may need to re-check permissions against the database or invalidate sessions when roles change.

### Rule 7: Use CSRF protection for state-changing forms

When we add profile updates or admin actions, we should use a CSRF token and validate it on the server. A logged-in browser should not unknowingly submit a sensitive action just because another site caused it to send a request.

---

## 16. Optional challenge: add an admin-only message

This small exercise reinforces the difference between showing a link and protecting the destination.

1. Create `public/admin/report.php`.
2. Include the shared authentication helpers.
3. Require the `admin` role before displaying the page.
4. Show the message: `Welcome to the administrator report.`
5. Test the page as a student and as an administrator.

**Expected result:** a student receives `403 Forbidden`, while an administrator can view the report.

Do not solve this by hiding the link alone. The page itself must enforce the role check.

---

## 17. Optional challenge: introduce a second role

Extend the application with a `teacher` role.

For example:

- `student`: view their own profile.
- `teacher`: view their own profile and a teacher-only resource.
- `admin`: access the user list and the teacher-only resource.

For this exercise, a simple role check can still use `requireRole('teacher')`. Consider what happens if an administrator should also be allowed to access the teacher resource. You may need a helper that accepts a list of permitted roles.

Do not change the public signup form to let people select their own role.

---

## 18. What we learned

We have extended a basic PHP login application into a role-aware application:

- Authentication establishes who has signed in.
- Authorization determines what that user may do.
- A `role` column records the user's application role.
- The login process places the trusted role in the session.
- `requireLogin()` protects pages for authenticated users.
- `requireRole('admin')` protects administrator-only pages.
- A profile page uses the session's user ID rather than trusting an ID from the URL.
- Server-side checks are essential even when navigation links are hidden.
- Public signup must never allow a visitor to grant themselves administrator privileges.

The main lesson is simple: **being logged in does not mean being allowed to do everything.**

---

## 19. Next lecture: CRUD operations with PHP and MySQL

In the next lesson, we can build on this application by learning CRUD:

- **Create:** add a record.
- **Read:** retrieve and display records.
- **Update:** edit a record.
- **Delete:** remove a record.

We will use prepared statements, HTML forms, validation, and server-side authorization. For example, students may update their own profile, while an administrator can manage user accounts. Before implementing deletion or other state-changing actions, we will add the appropriate authorization and CSRF protections.

### Before the next class

Make sure that:

1. Signup and login still work.
2. A new user receives the `student` role.
3. The administrator can open `/admin/users.php`.
4. A student cannot open `/admin/users.php`.
5. A logged-out visitor cannot open the dashboard or profile page.

Bring any error messages or screenshots to the next class so we can debug them together.
