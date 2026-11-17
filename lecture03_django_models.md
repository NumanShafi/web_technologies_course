# Lecture 03: Models, Migrations, the Django ORM & Admin Panel

> **Platform:** Ubuntu
> **Project:** Continue with `student_management` from Lecture 02

## Learning Outcomes

By the end of this lecture you will be able to:

1. Explain what a Model and an ORM are, and why we use them instead of raw SQL.
2. Define a model with suitable field types and options.
3. Create and apply migrations, and read the SQL they generate.
4. Create, read, update and delete records using the ORM in the Django shell.
5. Filter, sort and count records with QuerySets and field lookups.
6. Manage data through the Django Admin panel and customise it.
7. Show database records in templates through views.
8. Link two models with a `ForeignKey`.

---

## 01. Why Do We Need a Database?

In Lecture 02 our data lived in a Python list inside `views.py`:

```python
STUDENTS = [
    {'id': 1, 'name': 'Ali Raza', 'roll_no': 'BSCS-01', 'cgpa': 3.65},
    ...
]
```

Problems:

* Data is **lost or reset** every time the server restarts.
* Adding a student means **editing code**.
* No searching, sorting or relationships without writing everything by hand.
* Cannot handle thousands of records or many users at once.

**Solution:** Store data in a **database**, and describe it with **Models**. This is the **M** in **MVT**.

---

## 02. Models and the ORM

A **Model** is a Python class that describes one database table.

| Python (Model) | Database |
| :--- | :--- |
| Class `Student` | Table `students_student` |
| Attribute `name = CharField(...)` | Column `name VARCHAR(...)` |
| Object `Student(name='Ali')` | One row |

The **ORM (Object-Relational Mapper)** translates Python code into SQL for you:

```text
 Python (you write)                          SQL (Django generates)
 ----------------------------------          --------------------------------------------
 Student.objects.all()                 -->   SELECT * FROM students_student;
 Student.objects.filter(cgpa__gte=3)   -->   SELECT * FROM students_student WHERE cgpa >= 3;
 Student.objects.create(name='Ali')    -->   INSERT INTO students_student (name) VALUES ('Ali');
```

**Why use the ORM?**

* Write Python, not SQL.
* The same code works on SQLite, PostgreSQL and MySQL.
* Protection against SQL injection is built in.
* Table changes are tracked by migrations.

### The database we are using

`config/settings.py` already has:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

**SQLite** stores the whole database in one file (`db.sqlite3`). No installation or server is needed, which makes it ideal for learning and development.

---

## 03. Defining a Model

Open `students/models.py`:

```python
from django.db import models


class Student(models.Model):
    name = models.CharField(max_length=100)
    roll_no = models.CharField(max_length=20, unique=True)
    email = models.EmailField(blank=True)
    cgpa = models.DecimalField(max_digits=3, decimal_places=2, default=0)
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"{self.name} ({self.roll_no})"
```

* Every model **inherits** from `models.Model`.
* Each class attribute becomes a **column**.
* Django adds an **`id`** column automatically (primary key, auto-increment). You don't write it.
* **`__str__`** decides how an object is shown in the shell and the admin panel. Without it you see `Student object (1)`.

### Common field types

| Field | Stores | Example |
| :--- | :--- | :--- |
| `CharField(max_length=...)` | Short text (`max_length` required) | Name, roll number |
| `TextField()` | Long text | Address, bio |
| `IntegerField()` | Whole numbers | Semester, credit hours |
| `DecimalField(max_digits, decimal_places)` | Exact decimal numbers | CGPA, fee |
| `BooleanField()` | `True` / `False` | Is active |
| `DateField()` | Date | Date of birth |
| `DateTimeField()` | Date and time | Created at |
| `EmailField()` | Text, validated as an email | Email |
| `ForeignKey(...)` | Link to another model | Department |

### Common field options

| Option | Meaning |
| :--- | :--- |
| `max_length=100` | Maximum number of characters. |
| `unique=True` | No two rows may have the same value. |
| `default=0` | Value used when none is given. |
| `null=True` | The **database** may store `NULL` (empty) in this column. |
| `blank=True` | **Forms and admin** may leave this field empty. |
| `choices=[...]` | Restrict values to a fixed list. |
| `auto_now_add=True` | Set to the current time when the row is **created**. |
| `auto_now=True` | Set to the current time on **every save**. |

> **`null` vs `blank`:** `null` is about the database, `blank` is about validation. For text fields use only `blank=True`; for numbers, dates and foreign keys that may be empty, use `null=True, blank=True`.

---

## 04. Migrations

Writing a model does **not** create the table. Migrations do.

A **migration** is a file that records one change to your models (create a table, add a column, ...). Migrations are the version history of your database structure.

```text
  models.py  ──(1) makemigrations──>  migrations/0001_initial.py  ──(2) migrate──>  Database
  (you edit)                          (Django writes the plan)                      (tables change)
```

### Step 4.1: Create the migration

```bash
python manage.py makemigrations
```

```text
Migrations for 'students':
  students/migrations/0001_initial.py
    + Create model Student
```

Open `students/migrations/0001_initial.py` and look at it. It is plain Python describing the table.

### Step 4.2: Apply the migration

```bash
python manage.py migrate
```

```text
Applying students.0001_initial... OK
```

The table `students_student` (`<app>_<model>`) now exists in `db.sqlite3`.

### Step 4.3: Inspect

| Command | Shows |
| :--- | :--- |
| `python manage.py showmigrations` | All migrations; `[X]` = applied, `[ ]` = not applied. |
| `python manage.py sqlmigrate students 0001` | The exact SQL (`CREATE TABLE ...`) for that migration. |

### The golden rule

> **Every time you change `models.py`:** run `makemigrations`, then `migrate`.
> Never delete or hand-edit migration files, and never delete `db.sqlite3` to "fix" an error in a real project.

---

## 05. The Django Shell

The Django shell is a Python prompt with your project loaded. It is the best place to practise the ORM.

```bash
python manage.py shell
```

```python
>>> from students.models import Student
```

> Django 5.2 imports your models automatically when the shell starts, but writing the import is a good habit and is required in scripts and views.

Exit with `exit()` or `Ctrl + D`.

---

## 06. CRUD with the ORM

Every model has a manager called **`objects`**. All database queries start from it.

### 6.1 Create

```python
# Way 1: create() builds and saves in one step
>>> Student.objects.create(name='Ali Raza', roll_no='BSCS-01', cgpa=3.65)

# Way 2: build the object, then save()
>>> s = Student(name='Ayesha Khan', roll_no='BSCS-02', cgpa=3.10)
>>> s.save()

>>> Student.objects.create(name='Bilal Ahmed', roll_no='BSCS-03', cgpa=1.85)
>>> Student.objects.create(name='Sara Malik', roll_no='BSCS-04', cgpa=3.90, email='sara@example.com')
```

Nothing is written to the database until `save()` (or `create()`) is called.

### 6.2 Read

```python
>>> Student.objects.all()                     # all rows
<QuerySet [<Student: Ali Raza (BSCS-01)>, <Student: Ayesha Khan (BSCS-02)>, ...]>

>>> Student.objects.count()                   # number of rows
4

>>> s = Student.objects.get(roll_no='BSCS-01')   # exactly ONE object
>>> s.name
'Ali Raza'
>>> s.id
1

>>> Student.objects.first()                   # first row
>>> Student.objects.last()                    # last row
```

#### `get()` vs `filter()`

| | `get()` | `filter()` |
| :--- | :--- | :--- |
| Returns | **One object** | A **QuerySet** (list-like, 0 or more objects) |
| Nothing found | Raises `DoesNotExist` | Empty QuerySet |
| More than one found | Raises `MultipleObjectsReturned` | Returns all of them |
| Use when | Looking up by `id` or a unique field | Searching |

### 6.3 Update

```python
# One object
>>> s = Student.objects.get(roll_no='BSCS-03')
>>> s.cgpa = 2.40
>>> s.save()

# Many rows at once
>>> Student.objects.filter(cgpa__lt=2.0).update(is_active=False)
```

### 6.4 Delete

```python
>>> s = Student.objects.get(roll_no='BSCS-04')
>>> s.delete()

>>> Student.objects.filter(is_active=False).delete()    # many rows
```

> ⚠️ `Student.objects.all().delete()` empties the table. There is no undo.

---

## 07. Filtering, Lookups and Ordering

### 7.1 `filter()` and `exclude()`

```python
>>> Student.objects.filter(is_active=True)          # WHERE is_active = true
>>> Student.objects.exclude(cgpa=0)                 # WHERE NOT cgpa = 0
>>> Student.objects.filter(is_active=True, cgpa__gte=3.0)   # AND
```

### 7.2 Field lookups

A **lookup** is written as `field__lookup=value` (two underscores).

| Lookup | Meaning | Example |
| :--- | :--- | :--- |
| `exact` (default) | Equal to | `filter(name='Ali Raza')` |
| `iexact` | Equal, ignoring case | `filter(name__iexact='ali raza')` |
| `contains` | Contains text | `filter(name__contains='Khan')` |
| `icontains` | Contains, ignoring case | `filter(name__icontains='khan')` |
| `startswith` / `endswith` | Starts / ends with | `filter(roll_no__startswith='BSCS')` |
| `gt` / `gte` | Greater than / or equal | `filter(cgpa__gte=3.5)` |
| `lt` / `lte` | Less than / or equal | `filter(cgpa__lt=2.0)` |
| `in` | In a list | `filter(id__in=[1, 3])` |
| `range` | Between two values | `filter(cgpa__range=(2.0, 3.0))` |

### 7.3 Ordering and slicing

```python
>>> Student.objects.order_by('name')        # A → Z
>>> Student.objects.order_by('-cgpa')       # highest CGPA first ("-" = descending)
>>> Student.objects.order_by('-cgpa')[:3]   # top 3 (SQL LIMIT 3)
```

### 7.4 Chaining

Each method returns a new QuerySet, so calls can be chained:

```python
>>> Student.objects.filter(is_active=True).exclude(cgpa__lt=2.0).order_by('-cgpa')
```

### 7.5 Other useful methods

| Method | Returns |
| :--- | :--- |
| `.count()` | Number of matching rows. |
| `.exists()` | `True` if at least one row matches. |
| `.values('name', 'cgpa')` | Dictionaries with only those columns. |

### 7.6 QuerySets are lazy

A QuerySet does not touch the database until you actually **use** it (loop over it, print it, call `count()`, ...). Building and chaining filters is free.

To see the SQL Django will run:

```python
>>> print(Student.objects.filter(cgpa__gte=3.0).query)
```

### 7.7 Default ordering with `Meta`

```python
class Student(models.Model):
    ...

    class Meta:
        ordering = ['roll_no']

    def __str__(self):
        return f"{self.name} ({self.roll_no})"
```

`Meta` holds settings about the model itself. `ordering` sets the default order for every query. Run `makemigrations` and `migrate` after adding it.

---

## 08. The Django Admin Panel

Django builds a complete data-management website from your models, with no extra code.

### Step 8.1: Create an admin user

```bash
python manage.py createsuperuser
```

Enter a username, email (optional) and password. The password is not shown while typing.

### Step 8.2: Register the model

```python
# students/admin.py
from django.contrib import admin
from .models import Student

admin.site.register(Student)
```

### Step 8.3: Log in

```bash
python manage.py runserver
```

Open **http://127.0.0.1:8000/admin/** and log in. You can now add, edit, delete and search students from the browser.

### Step 8.4: Customise the admin

Replace the simple `register` line with a `ModelAdmin` class:

```python
# students/admin.py
from django.contrib import admin
from .models import Student


@admin.register(Student)
class StudentAdmin(admin.ModelAdmin):
    list_display = ('roll_no', 'name', 'cgpa', 'is_active')
    search_fields = ('name', 'roll_no')
    list_filter = ('is_active',)
    ordering = ('roll_no',)
    list_per_page = 20
```

| Option | Effect |
| :--- | :--- |
| `list_display` | Columns shown in the list page. |
| `search_fields` | Adds a search box that searches these fields. |
| `list_filter` | Adds a filter sidebar. |
| `ordering` | Default sort order in the admin. |
| `list_per_page` | Rows per page. |

> The admin panel is for **staff and administrators**, not for the public users of your site. Public pages are still built with views and templates.

---

## 09. Using Models in Views

Now we replace the `STUDENTS` list from Lecture 02 with real database queries. **Delete the `STUDENTS` list** from `views.py`.

### 9.1 List view

```python
# students/views.py
from django.shortcuts import render, get_object_or_404
from .models import Student


def student_list(request):
    students = Student.objects.all()
    return render(request, 'students/student_list.html', {'students': students})
```

The template from Lecture 02 keeps working without changes, because `{{ s.name }}`, `{{ s.roll_no }}` and `{{ s.cgpa }}` work the same way on model objects as on dictionaries:

```html
{% for s in students %}
    <tr>
        <td>{{ forloop.counter }}</td>
        <td><a href="{% url 'students:student_detail' s.id %}">{{ s.name }}</a></td>
        <td>{{ s.roll_no }}</td>
        <td>{{ s.cgpa }}</td>
    </tr>
{% empty %}
    <tr><td colspan="4">No students found.</td></tr>
{% endfor %}
```

### 9.2 Detail view with `get_object_or_404`

In Lecture 02 we searched the list with a loop and raised `Http404` ourselves. With models this becomes one line:

```python
def student_detail(request, student_id):
    student = get_object_or_404(Student, id=student_id)
    return render(request, 'students/student_detail.html', {'student': student})
```

**`get_object_or_404(Model, condition)`** returns the object if it exists, otherwise it shows a **404** page. Without it, `Student.objects.get(id=99)` would crash with a `DoesNotExist` error (a 500 page).

### 9.3 Filtering from the URL

```python
def student_list(request):
    students = Student.objects.all()

    query = request.GET.get('q')
    if query:
        students = students.filter(name__icontains=query)

    context = {
        'students': students,
        'total': students.count(),
        'query': query,
    }
    return render(request, 'students/student_list.html', context)
```

`/students/?q=khan` now shows only students whose name contains "khan".

### The full MVT cycle

```text
 Browser ── GET /students/2/ ──> urls.py ──> views.student_detail
                                                   │
                                  Student (models.py) <── ORM query ──> db.sqlite3
                                                   │
                                  student_detail.html (template)
                                                   │
 Browser <────────────── HTML response ────────────┘
```

---

## 10. Relationships: `ForeignKey`

Real data is connected: every student belongs to a department. A **`ForeignKey`** creates a **many-to-one** relationship: many students, one department.

### Step 10.1: Add the models

```python
# students/models.py
from django.db import models


class Department(models.Model):
    name = models.CharField(max_length=100, unique=True)
    code = models.CharField(max_length=10, unique=True)

    def __str__(self):
        return self.name


class Student(models.Model):
    name = models.CharField(max_length=100)
    roll_no = models.CharField(max_length=20, unique=True)
    email = models.EmailField(blank=True)
    cgpa = models.DecimalField(max_digits=3, decimal_places=2, default=0)
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    department = models.ForeignKey(
        Department,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='students',
    )

    class Meta:
        ordering = ['roll_no']

    def __str__(self):
        return f"{self.name} ({self.roll_no})"
```

> `Department` must be written **above** `Student` because `Student` refers to it.

**`on_delete`** (required) decides what happens to students when their department is deleted:

| Option | Effect |
| :--- | :--- |
| `models.CASCADE` | Delete the students too. |
| `models.PROTECT` | Refuse to delete a department that still has students. |
| `models.SET_NULL` | Keep the students, set their department to empty (needs `null=True`). |

**`related_name='students'`** is the name used to go **backwards**, from a department to its students.

### Step 10.2: Migrate

```bash
python manage.py makemigrations
python manage.py migrate
```

In the database the column is named `department_id` and stores the `id` of the department.

### Step 10.3: Use the relationship (shell)

```python
>>> from students.models import Student, Department

>>> cs = Department.objects.create(name='Computer Science', code='CS')
>>> it = Department.objects.create(name='Information Technology', code='IT')

>>> s = Student.objects.get(roll_no='BSCS-01')
>>> s.department = cs
>>> s.save()

# Forward: student → department
>>> s.department.name
'Computer Science'

# Backward: department → students (uses related_name)
>>> cs.students.all()
>>> cs.students.count()

# Filter across the relationship (double underscore)
>>> Student.objects.filter(department__code='CS')
```

### Step 10.4: Admin and template

```python
# students/admin.py
from .models import Student, Department

admin.site.register(Department)
```

Add `'department'` to `list_display` and `list_filter` in `StudentAdmin`. The student form in the admin now shows a department **dropdown**.

```html
<p><strong>Department:</strong> {{ student.department.name|default:"Not assigned" }}</p>
```

### Other relationship types (for reference)

| Field | Relationship | Example |
| :--- | :--- | :--- |
| `ForeignKey` | Many-to-one | Many students → one department |
| `ManyToManyField` | Many-to-many | Students ↔ courses |
| `OneToOneField` | One-to-one | Student ↔ profile |

---

## 🚨 Troubleshooting

| Error | Cause | Fix |
| :--- | :--- | :--- |
| `OperationalError: no such table: students_student` | Migrations not created or not applied. | `python manage.py makemigrations` then `migrate`. |
| `No changes detected` | App not in `INSTALLED_APPS`, or `models.py` not saved. | Check `INSTALLED_APPS`; save the file. |
| `It is impossible to add a non-nullable field ... without specifying a default` | New required field added, but old rows have no value for it. | Choose option 1 and type a default, or add `default=...` / `null=True` to the field. |
| `IntegrityError: UNIQUE constraint failed` | Duplicate value in a `unique=True` field. | Use a different value. |
| `DoesNotExist: Student matching query does not exist` | `get()` found nothing. | Use `get_object_or_404()` in views, or `filter()`. |
| `MultipleObjectsReturned` | `get()` found more than one row. | Use `filter()`, or look up by a unique field. |
| Model not visible in admin | Not registered in `admin.py`. | Register it and reload. |
| Admin shows `Student object (1)` | No `__str__` method. | Add `__str__` to the model. |
| `TypeError: ForeignKey.__init__() missing ... 'on_delete'` | `on_delete` not given. | Add `on_delete=models.CASCADE` (or another option). |
| Forgot the admin password | — | `python manage.py changepassword <username>` |

---

## 📋 Quick Reference

```text
WORKFLOW
  edit models.py  →  python manage.py makemigrations  →  python manage.py migrate

COMMANDS
  python manage.py showmigrations
  python manage.py sqlmigrate students 0001
  python manage.py shell
  python manage.py createsuperuser

CRUD
  Student.objects.create(name='Ali', roll_no='BSCS-01')
  Student.objects.all()   .get(id=1)   .filter(...)   .exclude(...)
  obj.field = value; obj.save()          qs.update(field=value)
  obj.delete()                           qs.delete()

QUERYSETS
  .filter(cgpa__gte=3.0)   .filter(name__icontains='ali')
  .order_by('-cgpa')       .count()     .exists()     [:5]

VIEWS
  from django.shortcuts import render, get_object_or_404
  student = get_object_or_404(Student, id=student_id)

RELATIONSHIPS
  department = models.ForeignKey(Department, on_delete=models.CASCADE, related_name='students')
  student.department.name        department.students.all()
  Student.objects.filter(department__code='CS')

ADMIN
  @admin.register(Student)
  class StudentAdmin(admin.ModelAdmin):
      list_display = (...); search_fields = (...); list_filter = (...)
```

---

## 🔜 Next Lecture

**Forms & CRUD from the browser:** HTML forms, `POST` requests, CSRF protection, Django `ModelForm`, and adding, editing and deleting students from your own pages.
