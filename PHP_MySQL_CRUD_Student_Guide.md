# PHP and MySQL CRUD Operations Using PDO
## Student Practical Guide — Ubuntu, Apache, PHP, and MySQL

## 1. Introduction

In previous lessons, we learned how to:

- Create user accounts and log in.
- Store information in PHP sessions.
- Hash and verify passwords.
- Use PDO to connect PHP with MySQL.
- Protect pages that require authentication.
- Apply role-based access control (RBAC).
- Allow or deny access according to a user's role.

In this lesson, we will use these concepts to develop a simple **Student Management System**.

The application will allow authorized users to create, view, edit, and delete student records.

### Learning objectives

By the end of this practical, students should be able to:

1. Explain CRUD operations.
2. Design a MySQL table for student records.
3. Insert data using PHP and PDO.
4. Retrieve and display database records.
5. Update existing records.
6. Delete records safely.
7. Validate form data on the server.
8. Use prepared statements to prevent SQL injection.
9. Protect pages using the authentication helpers from the previous lesson.
10. Escape output to reduce the risk of cross-site scripting (XSS).

---

## 2. What is CRUD?

CRUD represents the four basic operations performed on stored data.

| Operation | Meaning | Example |
|---|---|---|
| Create | Add a new record | Register a new student |
| Read | Retrieve existing records | Display all students |
| Update | Modify a record | Change a student's email |
| Delete | Remove a record | Delete a student record |

Most database-driven applications use CRUD operations. Examples include inventory systems, university portals, hospital systems, online stores, and employee management applications.

---

## 3. Existing development environment

We will continue using the virtual host configured in the previous lessons.

The existing configuration points to:

`/var/www/dev/public`

The website is accessible through:

`http://dev.local`

**Do not create another Apache virtual host.** Continue using the existing one.

Before starting, verify that Apache is running:

```bash
sudo systemctl status apache2
```

If necessary, start it:

```bash
sudo systemctl start apache2
```

Open the existing website in your browser and verify that `http://dev.local` works.

### Existing project structure

Your previous project may contain files similar to these:

```text
/var/www/dev/
├── config/
│   └── database.php
├── includes/
│   └── auth.php
└── public/
    ├── login.php
    ├── signup.php
    ├── dashboard.php
    └── logout.php
```

The exact filenames may differ slightly in your existing project. Reuse your working database connection and authentication helpers instead of creating duplicate versions.

We will add a `students` directory under `public`.

After completing the lesson, the relevant structure will be:

```text
/var/www/dev/
├── config/
│   └── database.php
├── includes/
│   └── auth.php
└── public/
    ├── login.php
    ├── dashboard.php
    ├── logout.php
    └── students/
        ├── index.php
        ├── create.php
        ├── edit.php
        └── delete.php
```

---

## 4. Step 1 — Create the students table

First, create a table to store student information.

Open the MySQL command line using the database account configured for your local development environment:

```bash
mysql -u devuser -p devdb
```

Enter the password for that database account.

Run the following SQL statement:

```sql
CREATE TABLE students (
    id INT AUTO_INCREMENT PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    department VARCHAR(100) NOT NULL,
    semester TINYINT UNSIGNED NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Understanding the SQL syntax

**`id INT AUTO_INCREMENT PRIMARY KEY`**

- `INT` stores an integer.
- `AUTO_INCREMENT` generates a new identifier automatically.
- `PRIMARY KEY` uniquely identifies each record.

For example, the first student might have ID `1`, the second ID `2`, and so on.

**`VARCHAR(100)`**

Stores text with a maximum length of 100 characters.

**`NOT NULL`**

Requires a value. A required form field alone is not sufficient; the database also enforces this constraint.

**`UNIQUE`**

Prevents two student records from having the same email address.

**`TINYINT UNSIGNED`**

Stores a small non-negative integer. We will use it for the semester number.

**`DEFAULT CURRENT_TIMESTAMP`**

Automatically records the time when a row is created.

Verify the table:

```sql
DESCRIBE students;
```

Leave MySQL:

```sql
EXIT;
```

### Important distinction

The `users` table from the authentication lesson stores login accounts. The new `students` table stores student records.

For this introductory exercise, these are separate tables. A student record is not automatically a login account.

---

## 5. Step 2 — Understand the database connection

The previous lesson introduced PDO, which provides a consistent interface for accessing databases.

Our examples will use the PDO connection already configured in the project.

For reference, a connection commonly looks like this:

```php
<?php

// Database connection settings for the local development environment.
$host = 'localhost';
$dbname = 'devdb';
$username = 'devuser';
$password = 'YOUR_LOCAL_DATABASE_PASSWORD';

// The DSN identifies the database driver, host, database, and character set.
$dsn = "mysql:host=$host;dbname=$dbname;charset=utf8mb4";

// Create a PDO object that PHP will use to communicate with MySQL.
$pdo = new PDO($dsn, $username, $password, [
    // Throw an exception when a database operation fails.
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,

    // Return retrieved rows as associative arrays.
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,

    // Ask PDO to use native prepared statements where supported.
    PDO::ATTR_EMULATE_PREPARES => false,
]);
```

Do not replace a working project connection without checking how your existing application includes it.

### Understanding `=>`

In PHP arrays, `=>` associates a key with a value.

For example:

```php
[
    'name' => 'Ali',
    'department' => 'Computer Science'
]
```

Here, `name` is a key and `Ali` is its value.

The PDO options use the same key-value structure.

### Understanding the PDO options

- `PDO::ATTR_ERRMODE` identifies the error-handling configuration option.
- `PDO::ERRMODE_EXCEPTION` tells PDO to throw exceptions for database errors.
- `PDO::ATTR_DEFAULT_FETCH_MODE` controls how retrieved rows are returned.
- `PDO::FETCH_ASSOC` returns each row as an associative array, with column names as keys.
- `PDO::ATTR_EMULATE_PREPARES => false` disables emulated prepared statements where supported by the driver.

For example, with associative fetching, a row can be accessed like this:

```php
echo $student['full_name'];
```

The array key is the database column name.

**Security note:** Keep database credentials outside publicly accessible files and never publish real passwords in a repository.

---

## 6. Step 3 — Create the students directory

Create the directory:

```bash
sudo mkdir -p /var/www/dev/public/students
```

The `-p` option creates missing parent directories if necessary and does not complain if the directory already exists.

The directory will contain four PHP files:

- `index.php` — list students.
- `create.php` — add a student.
- `edit.php` — update a student.
- `delete.php` — delete a student.

Create the first file:

```bash
sudo nano /var/www/dev/public/students/index.php
```

Use your normal development ownership and permissions. Avoid making the whole project writable by everyone.

---

## 7. Step 4 — Read and display student records

Add the following code to `index.php`.

```php
<?php

// Include the existing authentication helpers.
// __DIR__ represents the directory containing this PHP file.
// dirname(__DIR__, 2) moves from public/students to /var/www/dev.
require_once dirname(__DIR__, 2) . '/includes/auth.php';

// Require the user to be logged in before viewing student records.
// If the existing project uses a different helper name, use that helper.
requireLogin();

// Include the database connection.
// This example assumes config/database.php creates a PDO object named $pdo.
require_once dirname(__DIR__, 2) . '/config/database.php';

// This query contains no user-supplied SQL values.
// Therefore, PDO::query() is suitable for this fixed query.
$stmt = $pdo->query(
    'SELECT id, full_name, email, department, semester, created_at
     FROM students
     ORDER BY id DESC'
);

// Retrieve all matching rows as an array of associative arrays.
$students = $stmt->fetchAll();

// A helper for safely displaying values inside HTML.
// htmlspecialchars() converts special HTML characters into safe entities.
function escapeHtml($value)
{
    return htmlspecialchars(
        (string) $value,
        ENT_QUOTES,
        'UTF-8'
    );
}
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">

    <!-- Makes the page adapt to the screen width of a mobile device. -->
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <title>Student Management</title>
</head>
<body>

    <h1>Student Management System</h1>

    <p>
        <a href="create.php">Add New Student</a>
    </p>

    <?php if (isset($_GET['message'])): ?>
        <!-- Escape query-string values before displaying them. -->
        <p><?= escapeHtml($_GET['message']) ?></p>
    <?php endif; ?>

    <?php if (count($students) > 0): ?>

        <table border="1" cellpadding="8">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Full Name</th>
                    <th>Email</th>
                    <th>Department</th>
                    <th>Semester</th>
                    <th>Created At</th>
                    <th>Actions</th>
                </tr>
            </thead>

            <tbody>

                <?php foreach ($students as $student): ?>
                    <tr>
                        <td><?= escapeHtml($student['id']) ?></td>
                        <td><?= escapeHtml($student['full_name']) ?></td>
                        <td><?= escapeHtml($student['email']) ?></td>
                        <td><?= escapeHtml($student['department']) ?></td>
                        <td><?= escapeHtml($student['semester']) ?></td>
                        <td><?= escapeHtml($student['created_at']) ?></td>

                        <td>
                            <a href="edit.php?id=<?= escapeHtml($student['id']) ?>">
                                Edit
                            </a>

                            |

                            <a href="delete.php?id=<?= escapeHtml($student['id']) ?>">
                                Delete
                            </a>
                        </td>
                    </tr>
                <?php endforeach; ?>

            </tbody>
        </table>

    <?php else: ?>

        <p>No student records are available.</p>

    <?php endif; ?>

    <p>
        <a href="../dashboard.php">Back to Dashboard</a>
    </p>

</body>
</html>
```

### Understanding the new PHP syntax

**`require_once`**

Includes another PHP file. `once` prevents the same file from being included repeatedly in the same request.

**`dirname(__DIR__, 2)`**

`__DIR__` is the directory of the current PHP file. In this example, it points to `public/students`.

Moving up two directory levels reaches the project root, `/var/www/dev`.

This avoids depending on the shell's current working directory.

**`$pdo->query()`**

Executes a fixed SQL query. It is appropriate here because the SQL statement does not contain values supplied by the user.

When SQL includes user-provided values, use prepared statements instead.

**`fetchAll()`**

Retrieves all rows returned by the query. For a large database, use pagination rather than loading every record at once.

**`isset($_GET['message'])`**

Checks whether the `message` query parameter exists and is not `null`.

**`<?= ... ?>`**

A short PHP syntax for displaying an expression. It is equivalent to:

```php
<?php echo ...; ?>
```

**`foreach ($students as $student)`**

Loops through the list. On each iteration, `$student` represents one database row.

**`count($students) > 0`**

Checks whether the array contains at least one row.

**`?` in the URL**

In a URL such as `edit.php?id=5`, the question mark begins the query string. It is not a PHP ternary operator in this context.

### Why use `htmlspecialchars()`?

Database data is not automatically safe to place inside HTML. For example, a malicious name might contain HTML or JavaScript markup.

Escaping output helps the browser display such content as text rather than interpreting it as markup.

Escape data when displaying it, even if you also validate the input.

---

## 8. Step 5 — Create a new student

Create the file:

```bash
sudo nano /var/www/dev/public/students/create.php
```

Add the following code:

```php
<?php

require_once dirname(__DIR__, 2) . '/includes/auth.php';
requireLogin();

require_once dirname(__DIR__, 2) . '/config/database.php';

function escapeHtml($value)
{
    return htmlspecialchars((string) $value, ENT_QUOTES, 'UTF-8');
}

// Start with an empty error message.
$error = '';

// Store form values so the form can be repopulated after validation fails.
$fullName = '';
$email = '';
$department = '';
$semester = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    // Trim removes whitespace from the beginning and end of strings.
    $fullName = trim($_POST['full_name'] ?? '');
    $email = trim($_POST['email'] ?? '');
    $department = trim($_POST['department'] ?? '');
    $semesterInput = trim($_POST['semester'] ?? '');

    // Validate required text fields.
    if ($fullName === '' || $department === '') {
        $error = 'Name and department are required.';

    } elseif (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        // Validate the email format on the server.
        $error = 'Please enter a valid email address.';

    } elseif (
        $semesterInput === '' ||
        filter_var($semesterInput, FILTER_VALIDATE_INT) === false ||
        (int) $semesterInput < 1 ||
        (int) $semesterInput > 12
    ) {
        // Convert the semester to an integer only after validation.
        $error = 'Semester must be an integer between 1 and 12.';

    } else {

        try {

            // Placeholders represent values supplied separately to PDO.
            $sql = 'INSERT INTO students
                    (full_name, email, department, semester)
                    VALUES
                    (:full_name, :email, :department, :semester)';

            $stmt = $pdo->prepare($sql);

            // Bind the validated values to the SQL placeholders.
            $stmt->execute([
                'full_name' => $fullName,
                'email' => $email,
                'department' => $department,
                'semester' => (int) $semesterInput
            ]);

            // Redirect after a successful insert.
            // This prevents accidental form resubmission on refresh.
            header('Location: index.php?message=' . urlencode('Student added successfully.'));
            exit;

        } catch (PDOException $exception) {

            // Do not display database exception details to the visitor.
            // A duplicate email is one possible reason for failure.
            $error = 'Could not add the student. Check that the email is unique.';
        }
    }
}
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Add Student</title>
</head>
<body>

    <h1>Add Student</h1>

    <?php if ($error !== ''): ?>
        <p><?= escapeHtml($error) ?></p>
    <?php endif; ?>

    <form method="POST" action="create.php">

        <label for="full_name">Full Name:</label>
        <input
            type="text"
            id="full_name"
            name="full_name"
            maxlength="100"
            value="<?= escapeHtml($fullName) ?>"
            required
        >
        <br><br>

        <label for="email">Email:</label>
        <input
            type="email"
            id="email"
            name="email"
            maxlength="150"
            value="<?= escapeHtml($email) ?>"
            required
        >
        <br><br>

        <label for="department">Department:</label>
        <input
            type="text"
            id="department"
            name="department"
            maxlength="100"
            value="<?= escapeHtml($department) ?>"
            required
        >
        <br><br>

        <label for="semester">Semester:</label>
        <input
            type="number"
            id="semester"
            name="semester"
            min="1"
            max="12"
            value="<?= escapeHtml($semesterInput ?? '') ?>"
            required
        >
        <br><br>

        <button type="submit">Save Student</button>

    </form>

    <p><a href="index.php">Back to Student List</a></p>

</body>
</html>
```

### Understanding the form-handling code

**`$_SERVER['REQUEST_METHOD'] === 'POST'`**

Checks whether the browser submitted the form using the POST method.

- `GET` is commonly used to request a page.
- `POST` is commonly used to submit form data.

The comparison operator `===` checks both value and type.

**`$_POST['full_name'] ?? ''`**

The `??` operator is called the null coalescing operator. It returns the left-hand value if it exists and is not `null`; otherwise, it returns the right-hand value.

This prevents an undefined-array-key warning when the form field is absent.

**`trim()`**

Removes whitespace from the start and end of a string.

**`filter_var()`**

Can validate data according to a filter. We use `FILTER_VALIDATE_EMAIL` for email format and `FILTER_VALIDATE_INT` for the semester.

Client-side HTML attributes such as `required`, `min`, and `max` improve usability but do not replace server-side validation.

**`$pdo->prepare($sql)`**

Prepares a SQL statement containing named placeholders.

For example, `:email` is a placeholder, not a PHP variable.

**`$stmt->execute([...])`**

Supplies the values separately from the SQL statement. This helps prevent SQL injection because the supplied values are not interpreted as SQL syntax.

**`try` and `catch`**

A `try` block contains code that might throw an exception. A matching `catch` block handles the exception.

Here, a duplicate email may cause the database insert to fail because the column is unique.

**`header('Location: ...')`**

Sends a redirect response. Call it before any HTML or other output is sent.

**`exit`**

Stops execution after the redirect.

**Why redirect after POST?**

If the browser refreshes the result page, the application should not immediately resubmit the same form. Redirecting to the listing page helps avoid accidental duplicate submissions.

---

## 9. Step 6 — Edit an existing student

Create the file:

```bash
sudo nano /var/www/dev/public/students/edit.php
```

Add:

```php
<?php

require_once dirname(__DIR__, 2) . '/includes/auth.php';
requireLogin();

require_once dirname(__DIR__, 2) . '/config/database.php';

function escapeHtml($value)
{
    return htmlspecialchars((string) $value, ENT_QUOTES, 'UTF-8');
}

$error = '';

// Read the record ID from the URL.
// Validate it before using it in a database query.
$id = filter_input(INPUT_GET, 'id', FILTER_VALIDATE_INT);

if (!$id || $id < 1) {
    http_response_code(400);
    exit('Invalid student ID.');
}

// Retrieve the selected student using a prepared statement.
$stmt = $pdo->prepare(
    'SELECT id, full_name, email, department, semester
     FROM students
     WHERE id = :id'
);

$stmt->execute(['id' => $id]);

$student = $stmt->fetch();

if (!$student) {
    http_response_code(404);
    exit('Student record not found.');
}

// Initialize form values from the database record.
$fullName = $student['full_name'];
$email = $student['email'];
$department = $student['department'];
$semesterInput = (string) $student['semester'];

if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    // Use the validated ID obtained from the URL, not a trusted ID from the form.
    $fullName = trim($_POST['full_name'] ?? '');
    $email = trim($_POST['email'] ?? '');
    $department = trim($_POST['department'] ?? '');
    $semesterInput = trim($_POST['semester'] ?? '');

    if ($fullName === '' || $department === '') {
        $error = 'Name and department are required.';

    } elseif (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        $error = 'Please enter a valid email address.';

    } elseif (
        $semesterInput === '' ||
        filter_var($semesterInput, FILTER_VALIDATE_INT) === false ||
        (int) $semesterInput < 1 ||
        (int) $semesterInput > 12
    ) {
        $error = 'Semester must be an integer between 1 and 12.';

    } else {

        try {

            $sql = 'UPDATE students
                    SET full_name = :full_name,
                        email = :email,
                        department = :department,
                        semester = :semester
                    WHERE id = :id';

            $stmt = $pdo->prepare($sql);

            $stmt->execute([
                'full_name' => $fullName,
                'email' => $email,
                'department' => $department,
                'semester' => (int) $semesterInput,
                'id' => $id
            ]);

            header('Location: index.php?message=' . urlencode('Student updated successfully.'));
            exit;

        } catch (PDOException $exception) {
            $error = 'Could not update the student. Check that the email is unique.';
        }
    }
}
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Edit Student</title>
</head>
<body>

    <h1>Edit Student</h1>

    <?php if ($error !== ''): ?>
        <p><?= escapeHtml($error) ?></p>
    <?php endif; ?>

    <form method="POST" action="edit.php?id=<?= escapeHtml($id) ?>">

        <label for="full_name">Full Name:</label>
        <input
            type="text"
            id="full_name"
            name="full_name"
            maxlength="100"
            value="<?= escapeHtml($fullName) ?>"
            required
        >
        <br><br>

        <label for="email">Email:</label>
        <input
            type="email"
            id="email"
            name="email"
            maxlength="150"
            value="<?= escapeHtml($email) ?>"
            required
        >
        <br><br>

        <label for="department">Department:</label>
        <input
            type="text"
            id="department"
            name="department"
            maxlength="100"
            value="<?= escapeHtml($department) ?>"
            required
        >
        <br><br>

        <label for="semester">Semester:</label>
        <input
            type="number"
            id="semester"
            name="semester"
            min="1"
            max="12"
            value="<?= escapeHtml($semesterInput) ?>"
            required
        >
        <br><br>

        <button type="submit">Update Student</button>

    </form>

    <p><a href="index.php">Cancel</a></p>

</body>
</html>
```

### Understanding UPDATE operations

The SQL statement:

```sql
UPDATE students
SET full_name = :full_name
WHERE id = :id
```

changes the name of the row matching the specified ID.

**Never forget the `WHERE` clause in an update operation.** Without it, the database may update every row in the table.

### Why validate the ID?

The URL might contain values such as:

```text
edit.php?id=abc
edit.php?id=-5
edit.php?id=999999
```

`FILTER_VALIDATE_INT` checks whether the ID is an integer. We then reject invalid or non-positive IDs and check whether a corresponding database record exists.

A prepared statement prevents the ID from being interpreted as SQL code, but it does not replace input validation.

---

## 10. Step 7 — Delete a student safely

Deleting a record changes data permanently, so we will require a POST request and a confirmation step.

Create the file:

```bash
sudo nano /var/www/dev/public/students/delete.php
```

Add:

```php
<?php

require_once dirname(__DIR__, 2) . '/includes/auth.php';
requireLogin();

require_once dirname(__DIR__, 2) . '/config/database.php';

function escapeHtml($value)
{
    return htmlspecialchars((string) $value, ENT_QUOTES, 'UTF-8');
}

// Validate the record ID supplied in the URL.
$id = filter_input(INPUT_GET, 'id', FILTER_VALIDATE_INT);

if (!$id || $id < 1) {
    http_response_code(400);
    exit('Invalid student ID.');
}

// Retrieve the student so the confirmation page can show the name.
$stmt = $pdo->prepare(
    'SELECT id, full_name
     FROM students
     WHERE id = :id'
);

$stmt->execute(['id' => $id]);

$student = $stmt->fetch();

if (!$student) {
    http_response_code(404);
    exit('Student record not found.');
}

if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    // A POST request is used for the destructive operation.
    // A production application should also validate a CSRF token here.
    $stmt = $pdo->prepare(
        'DELETE FROM students
         WHERE id = :id'
    );

    $stmt->execute(['id' => $id]);

    header('Location: index.php?message=' . urlencode('Student deleted successfully.'));
    exit;
}
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Delete Student</title>
</head>
<body>

    <h1>Delete Student</h1>

    <p>
        Are you sure you want to delete
        <strong><?= escapeHtml($student['full_name']) ?></strong>?
    </p>

    <form method="POST" action="delete.php?id=<?= escapeHtml($id) ?>">
        <button type="submit">Yes, Delete Student</button>
    </form>

    <p><a href="index.php">Cancel</a></p>

</body>
</html>
```

### Why not delete through a GET request?

A GET request should retrieve information rather than perform a destructive operation. Links may be followed unintentionally by a user, browser preview, crawler, or other system.

This example uses a confirmation page and submits the deletion through POST.

**Important security improvement:** POST alone does not prevent cross-site request forgery (CSRF). A production application should generate a CSRF token, store it in the session, include it in the form, and validate it before performing the deletion. The same protection should be applied to create and update forms.

### Understanding the DELETE query

```sql
DELETE FROM students WHERE id = :id
```

This removes the matching record.

The `WHERE` clause limits the operation to the selected student. Without it, the statement could delete every row in the table.

---

## 11. Step 8 — Connect the feature to the dashboard

Open your existing dashboard:

```bash
sudo nano /var/www/dev/public/dashboard.php
```

Add a link to the student listing somewhere inside the page body:

```php
<p>
    <a href="students/index.php">Manage Students</a>
</p>
```

This is an ordinary HTML link. It does not itself enforce access control.

Each student management PHP page calls `requireLogin()`, so users who are not logged in should be redirected or denied access according to the authentication helper from the previous lesson.

### What about administrators?

If only administrators should manage student records, use your existing role helper:

```php
requireRole('admin');
```

Replace `requireLogin()` with `requireRole('admin')` in each protected management page, assuming your previous helper also checks that the user is logged in.

Do not rely on hiding the dashboard link as the security mechanism. Access must be checked on the server for every protected request.

For a more advanced system, you might allow teachers to create and update records while restricting deletion to administrators.

---

## 12. Step 9 — Test the complete application

Open the browser and log in through your existing login page.

Visit:

`http://dev.local/students/index.php`

### Test A — Read

1. Open the student list.
2. Confirm that it displays a message when there are no records.
3. Add a test record through MySQL or the form.
4. Reload the list.
5. Confirm that the record appears.

### Test B — Create

1. Select **Add New Student**.
2. Enter a name, email, department, and semester.
3. Submit the form.
4. Confirm that the record appears in the listing.
5. Try submitting an invalid email.
6. Try entering a semester outside the permitted range.
7. Try adding another record with an existing email.

Expected result: invalid inputs should be rejected, and the database should enforce unique email addresses.

### Test C — Update

1. Select **Edit** beside a record.
2. Change its department or semester.
3. Submit the form.
4. Return to the listing.
5. Confirm that the existing record has changed rather than a new record being created.

### Test D — Delete

1. Select **Delete** beside a test record.
2. Confirm that the correct student name is shown.
3. Cancel and verify that the record remains.
4. Repeat the operation and confirm deletion.
5. Return to the listing and verify that the record no longer appears.

### Test E — Access control

1. Log out.
2. Attempt to open the student listing directly.
3. Confirm that access is denied or the browser is redirected to login.
4. Log in again and verify that the pages work.

### Test F — Invalid record IDs

Try these URLs:

```text
http://dev.local/students/edit.php?id=abc
http://dev.local/students/edit.php?id=-1
http://dev.local/students/edit.php?id=999999
```

The application should reject invalid IDs and report that nonexistent records cannot be edited.

---

## 13. Common problems and troubleshooting

### Problem 1: `Unknown column` or `Table doesn't exist`

**Possible cause:** The table was not created in the expected database.

Check the selected database:

```sql
SELECT DATABASE();
SHOW TABLES;
DESCRIBE students;
```

Confirm that your PHP connection uses the same database.

### Problem 2: `Undefined variable $pdo`

**Possible cause:** The database connection file may use a different variable name, or it may not be included correctly.

Check the existing `config/database.php` file and use its actual PDO variable. Do not create a second connection just to resolve a variable-name mismatch.

### Problem 3: `Call to undefined function requireLogin()`

**Possible cause:** The authentication file uses a different function name or is not included.

Open the existing authentication helper and verify the function names. Reuse the names from your previous lesson.

### Problem 4: Duplicate email error

**Possible cause:** A record already uses the submitted email address.

This is expected behavior because the database column has a `UNIQUE` constraint. Try a different email address.

For a more polished application, check for an existing email before inserting or updating, while retaining the database constraint as the final safeguard against concurrent requests.

### Problem 5: `headers already sent`

**Possible cause:** PHP attempted to redirect after output had already been sent.

Ensure that redirects occur before HTML output. Check for accidental whitespace or output before the opening `<?php` tag in included files.

### Problem 6: HTTP 403 or 500

- A `403` usually indicates forbidden access.
- A `500` indicates a server-side failure.

Check Apache's error log:

```bash
sudo tail -n 50 /var/log/apache2/error.log
```

During local development, inspect errors carefully. Do not expose detailed PHP exceptions, SQL errors, or credentials to ordinary users.

---

## 14. Security lessons

This practical introduces several important security principles.

### Use prepared statements

Do not construct SQL by concatenating untrusted input into the query.

Unsafe pattern:

```php
$sql = "SELECT * FROM students WHERE email = '$email'";
```

Safer pattern:

```php
$stmt = $pdo->prepare(
    'SELECT * FROM students WHERE email = :email'
);

$stmt->execute(['email' => $email]);
```

### Validate input on the server

Browser validation is useful for usability but can be bypassed. Validate all submitted data in PHP before using it.

### Escape output

Use `htmlspecialchars()` when inserting user-controlled values into HTML text or quoted HTML attributes.

### Protect every page

A user can enter a URL directly. Each protected page must perform its own authorization check.

### Use the HTTP method appropriately

- GET: retrieve or display information.
- POST: submit data or perform a state-changing operation.

For production applications, use CSRF tokens for state-changing requests, including deletion.

### Use least privilege

A database account used by a web application should have only the permissions it needs. For a production deployment, do not use a database administrator account.

### Do not expose internal errors

Log technical details for developers and show a generic message to users. Keep credentials and sensitive configuration outside publicly accessible directories.

---

## 15. Student exercises

Complete these tasks independently after the main practical works.

### Exercise 1 — Add a phone number

1. Add a `phone` column to the students table.
2. Add a phone field to the create form.
3. Store the phone number when inserting a record.
4. Display it in the listing.
5. Allow it to be updated.

### Exercise 2 — Search students

Add a search form that allows a user to search by student name or email.

Requirements:

- Use a GET form for the search.
- Validate and trim the search input.
- Use a prepared statement.
- Display the results safely.
- Provide a way to clear the search.

### Exercise 3 — Restrict deletion

Allow administrators to delete student records, but allow ordinary authenticated users only to view them.

Requirements:

- Use the role helper from the previous lesson.
- Check authorization on the server.
- Do not rely only on hiding the Delete link.

### Exercise 4 — Add pagination

Modify the listing to display a limited number of records per page.

Requirements:

- Display no more than 10 records on each page.
- Add Next and Previous navigation.
- Validate the page number.
- Use prepared statements for any user-supplied query values.
- Explain why pagination is useful for large databases.

### Exercise 5 — Add CSRF protection

Extend the create, edit, and delete forms to use CSRF tokens.

Requirements:

- Generate a token using a cryptographically secure PHP function.
- Store it in the session.
- Include it in each form.
- Verify it before changing database records.
- Reject missing or invalid tokens.

---

## 16. Submission requirements

Submit your working Student Management System with:

1. SQL used to create the table.
2. PHP source files for listing, creating, editing, and deleting records.
3. Screenshots showing successful CRUD operations.
4. Evidence that invalid email addresses and semester values are rejected.
5. Evidence that duplicate email addresses are not accepted.
6. Evidence that unauthenticated users cannot access protected pages.
7. A short explanation of prepared statements, validation, output escaping, and access control.

Use test data only. Do not include real passwords or private student information in screenshots or source files.

---

## 17. Summary and next lesson

In this lesson, we connected HTML forms, PHP, PDO, and MySQL to implement all four CRUD operations.

We learned that:

- **Create** inserts new records.
- **Read** retrieves records for display.
- **Update** changes an existing record.
- **Delete** removes a selected record.
- Prepared statements separate SQL from supplied values.
- Server-side validation checks submitted data.
- Output escaping helps protect HTML pages.
- Authentication and authorization checks must run on the server.
- State-changing operations need additional protections such as CSRF tokens.

**Suggested next lesson:** Search, filtering, pagination, and secure file uploads in PHP. These features will help students extend the basic CRUD application into a more practical web application.
