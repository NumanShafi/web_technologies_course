# Lab 01: Introduction to Django, Environment Setup, Views & URLs

> **Platform:** Linux only (Ubuntu / Debian commands shown, Fedora alternatives where different)
> **Project built in this lab:** `student_management`

## Learning Outcomes

By the end of this lab you will be able to:

1. Explain what Django is and how a request travels through it (MVT).
2. Install Python, `pip` and `venv` on Linux.
3. Create and use a virtual environment.
4. Install Django, create a project and an app.
5. Run the development server.
6. Write views and connect them to URLs, including URLs that take parameters.

---

## 01. Introduction to Django

### What is Django?

Django is a free, open-source **Python web framework** for building websites and web applications quickly. It ships with most of what a web app needs out of the box ("batteries included"):

* URL routing
* Database access through an ORM (write Python, not SQL)
* HTML templates
* User authentication (login, logout, passwords)
* An automatic admin panel
* Protection against common attacks (CSRF, XSS, SQL injection)

### The MVT Pattern

Django follows the **MVT (Model – View – Template)** pattern.

| Part | File | Responsibility |
| :--- | :--- | :--- |
| **Model** | `models.py` | Defines data and talks to the database. |
| **View** | `views.py` | Receives a request, runs logic, returns a response. |
| **Template** | `templates/*.html` | The HTML the user sees. |
| **URLs** | `urls.py` | Decides which view handles which URL. |

### Request–Response Cycle

```text
 Browser                       Django
 -------                       ------
   |  GET /students/             |
   |---------------------------->|  1. urls.py   finds a matching path
   |                             |  2. views.py  runs the view function
   |                             |  3. models.py (optional) reads/writes DB
   |                             |  4. template  (optional) builds HTML
   |      HttpResponse (HTML)    |
   |<----------------------------|
```

> In this lab we use only **URLs** and **Views**. Models and Templates come in later labs.

---

## 02. Installing Python on Linux

### Step 2.1: Check if Python is already installed

```bash
python3 --version
```

On Linux the command is **`python3`**, not `python`. Most distributions already include Python 3.

### Which Python version do I need?

| Django version | Supported Python |
| :--- | :--- |
| **5.2 (LTS)** — used in this course | 3.10, 3.11, 3.12, 3.13, 3.14 |
| 6.0 | 3.12, 3.13, 3.14 |

Ubuntu 22.04 ships Python 3.10 and Ubuntu 24.04 ships Python 3.12, so Django 5.2 works on both. That is why this course uses Django 5.2 LTS (Long-Term Support).

### Step 2.2: Install Python, pip and venv

**Ubuntu / Debian / Linux Mint:**

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv
```

**Fedora / RHEL / Rocky:**

```bash
sudo dnf install -y python3 python3-pip
```

| Package | Purpose |
| :--- | :--- |
| `python3` | The Python interpreter. |
| `python3-pip` | `pip`, the package installer (downloads packages from PyPI). |
| `python3-venv` | The `venv` module for creating virtual environments. On Ubuntu/Debian it is a separate package; without it `python3 -m venv` fails. |

* **`sudo`** runs the command as administrator.
* **`apt update`** refreshes the list of available packages.
* **`-y`** answers "yes" to the install prompt automatically.

### Step 2.3: Verify

```bash
python3 --version          # e.g. Python 3.12.3
python3 -m pip --version   # e.g. pip 24.0 from ... (python 3.12)
which python3              # e.g. /usr/bin/python3
```

* **`python3 -m pip`** runs the `pip` that belongs to that exact Python, which avoids version mix-ups.
* **`which`** shows the full path of the program the shell will run.

> ⚠️ **Never run `sudo pip install ...`.** It installs packages into the system Python that Linux itself depends on and can break system tools. Recent Ubuntu/Debian block this with an `externally-managed-environment` error. Always install packages **inside a virtual environment** (next section).

---

## 03. Virtual Environments (`venv`)

A **virtual environment** is a private folder holding its own Python interpreter and its own packages. Each project gets its own environment, so:

* Project A can use Django 5.2 while Project B uses Django 6.0.
* You never touch the system Python.
* You can delete the folder and rebuild it at any time.

### Step 3.1: Create the project folder

```bash
mkdir -p ~/projects/student_management
cd ~/projects/student_management
```

* **`~`** is your home directory (`/home/<username>`).
* **`mkdir -p`** creates the folder and any missing parent folders, with no error if they already exist.

### Step 3.2: Create the virtual environment

```bash
python3 -m venv venv
```

| Part | Meaning |
| :--- | :--- |
| `python3` | Python interpreter used to build the environment. |
| `-m venv` | Run the built-in `venv` module. |
| `venv` (last word) | Name of the folder to create. `venv` or `.venv` is the usual convention. |

### Step 3.3: Activate it

```bash
source venv/bin/activate
```

Your prompt changes to show the environment name:

```text
(venv) student@pc:~/projects/student_management$
```

Confirm you are using the environment's Python:

```bash
which python
# /home/student/projects/student_management/venv/bin/python
```

> Inside an active venv, `python` and `pip` point to the venv. You can use them instead of `python3` / `pip3`.

### Step 3.4: Deactivate it

```bash
deactivate
```

> **Remember:** Every time you open a new terminal, `cd` into the project and run `source venv/bin/activate` again before working.

---

## 04. Installing Django

Make sure the prompt shows `(venv)` first.

### Step 4.1: Upgrade pip

```bash
python -m pip install --upgrade pip
```

### Step 4.2: Install Django 5.2 LTS

```bash
pip install "django>=5.2,<5.3"
```

* The quotes stop the shell from treating `<` and `>` as redirection.
* `>=5.2,<5.3` installs the newest 5.2.x release (security fixes included).

### Step 4.3: Verify

```bash
python -m django --version
# 5.2.x
```

### Step 4.4: Save your dependencies

```bash
pip freeze > requirements.txt
```

`requirements.txt` lists every installed package with its exact version. Anyone (or you, on another machine) can recreate the same environment with:

```bash
pip install -r requirements.txt
```

> Share `requirements.txt`, **not** the `venv/` folder. If you use Git, add `venv/` to `.gitignore`.

---

## 05. Creating a Django Project

A **project** is the whole website: its settings and its top-level URLs.

```bash
django-admin startproject config .
```

| Part | Meaning |
| :--- | :--- |
| `django-admin` | Django's command-line tool (installed with Django). |
| `startproject` | Generates the project skeleton. |
| `config` | Name of the settings package. |
| `.` | Create files in the **current folder** instead of an extra nested folder. Don't forget it. |

### Project Structure

```text
student_management/
├── config/
│   ├── __init__.py     # Marks the folder as a Python package
│   ├── settings.py     # All project settings
│   ├── urls.py         # Top-level URL table
│   ├── asgi.py         # Entry point for async servers (deployment)
│   └── wsgi.py         # Entry point for sync servers (deployment)
├── manage.py           # Run project commands with this
├── requirements.txt
└── venv/
```

### Files you need to know now

**`manage.py`** — the command runner for your project. You never edit it; you run it:

```bash
python manage.py <command>
```

**`config/settings.py`** — key settings:

```python
DEBUG = True            # Shows detailed error pages. MUST be False in production.
ALLOWED_HOSTS = []      # Domain names/IPs allowed to serve the site when DEBUG=False.
INSTALLED_APPS = [...]  # Apps active in this project (built-in + yours).
DATABASES = {...}       # Default: SQLite file db.sqlite3 (no setup needed).
TIME_ZONE = 'UTC'       # Change to 'Asia/Karachi' for local time.
SECRET_KEY = '...'      # Used for security signing. Never share or commit in real projects.
```

**`config/urls.py`** — the first place Django looks when a request arrives:

```python
from django.contrib import admin
from django.urls import path

urlpatterns = [
    path('admin/', admin.site.urls),
]
```

---

## 06. Running the Development Server

### Step 6.1: Apply initial migrations

Django's built-in apps (admin, auth, sessions) need database tables. Create them once:

```bash
python manage.py migrate
```

This creates `db.sqlite3` in your project folder. Without this step the server shows a red *"You have 18 unapplied migration(s)"* warning.

### Step 6.2: Start the server

```bash
python manage.py runserver
```

Open **http://127.0.0.1:8000/** in your browser. You should see the Django welcome page with a rocket.

* The server **auto-reloads** when you save a Python file.
* Press **`Ctrl + C`** to stop it.
* This server is for **development only**, never for production.

### Useful options

| Command | Use |
| :--- | :--- |
| `python manage.py runserver 8080` | Use port 8080 if 8000 is busy. |
| `python manage.py runserver 0.0.0.0:8000` | Make the site reachable from other devices on your network (open `http://<your-ip>:8000`; also add your IP to `ALLOWED_HOSTS`). |

Find your machine's IP with `hostname -I`.

---

## 07. Creating a Django App

A **project** is the whole site; an **app** is one feature of it (e.g. `students`, `courses`, `fees`). One project contains many apps.

### Step 7.1: Create the app

```bash
python manage.py startapp students
```

```text
students/
├── __init__.py
├── admin.py        # Register models for the admin panel
├── apps.py         # App configuration
├── migrations/     # Database change history (auto-generated)
│   └── __init__.py
├── models.py       # Database tables as Python classes
├── tests.py        # Automated tests
└── views.py        # View functions
```

### Step 7.2: Register the app

Django ignores an app until you add it to `INSTALLED_APPS` in `config/settings.py`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Local apps
    'students',
]
```

---

## 08. Views

A **view** is a Python function that:

1. receives a `request` object (an `HttpRequest`), and
2. returns a response object (an `HttpResponse`).

### Step 8.1: Write your first views

Replace the contents of `students/views.py`:

```python
from django.http import HttpResponse


def student_list(request):
    return HttpResponse("""
        <h1>Student Management System</h1>
        <p>Welcome to the Student Portal.</p>
    """)


def student_detail(request, student_id):
    return HttpResponse(f"<h1>Details of student #{student_id}</h1>")


def about(request):
    return HttpResponse(f"<p>You sent a {request.method} request to {request.path}</p>")
```

### What's inside `request`?

| Attribute | Example value | Meaning |
| :--- | :--- | :--- |
| `request.method` | `'GET'`, `'POST'` | HTTP method used. |
| `request.path` | `'/students/about/'` | URL path requested. |
| `request.GET` | `{'q': 'ali'}` | Query-string values (`?q=ali`). |
| `request.POST` | form data | Data submitted by a form. |

> Writing HTML inside Python strings is fine for learning, but it gets messy quickly. In the next lab we move HTML into **templates**.

---

## 09. URLs (Two-Level Routing)

Each app keeps its own `urls.py`, and the project's `urls.py` includes it. This keeps apps independent and reusable.

```text
Request: /students/5/
          │
          ▼
config/urls.py     path('students/', include('students.urls'))
          │         matches "students/" → passes on the rest: "5/"
          ▼
students/urls.py   path('<int:student_id>/', views.student_detail)
          │         matches "5/" → student_id = 5
          ▼
views.student_detail(request, student_id=5)
```

### Step 9.1: Create the app's `urls.py`

```bash
touch students/urls.py
```

```python
# students/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('', views.student_list, name='student_list'),                 # /students/
    path('about/', views.about, name='about'),                         # /students/about/
    path('<int:student_id>/', views.student_detail, name='student_detail'),  # /students/5/
]
```

**`path(route, view, name=...)`**

* **`route`** — the URL pattern as a string. `''` means "the app's root".
* **`view`** — the function to call (no parentheses: `views.student_list`, not `views.student_list()`).
* **`name`** — a label for the URL so you can refer to it later without hard-coding the path.

### Path converters

`<int:student_id>` captures part of the URL and passes it to the view as an argument.

| Converter | Matches | Example |
| :--- | :--- | :--- |
| `<int:x>` | Whole numbers | `/students/5/` |
| `<str:x>` | Any text without `/` (default) | `/students/ali/` |
| `<slug:x>` | Letters, numbers, `-`, `_` | `/posts/my-first-post/` |

> The name inside `< >` must match the view's parameter name exactly.

### Step 9.2: Include the app in the project

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('students/', include('students.urls')),
]
```

**`include('students.urls')`** removes the matched part (`students/`) and sends the remainder to `students/urls.py`.

### Step 9.3: Test

```bash
python manage.py runserver
```

| URL | Expected output |
| :--- | :--- |
| http://127.0.0.1:8000/students/ | Student Management System |
| http://127.0.0.1:8000/students/5/ | Details of student #5 |
| http://127.0.0.1:8000/students/about/ | You sent a GET request to /students/about/ |
| http://127.0.0.1:8000/students/abc/ | 404, because `abc` is not an `int` |

> **Trailing slashes:** Django routes conventionally end with `/`. If you visit `/students/about` (no slash), Django redirects to `/students/about/` automatically.

---

## 🚨 Troubleshooting

| Error | Cause | Fix |
| :--- | :--- | :--- |
| `The virtual environment was not created successfully because ensurepip is not available` | `python3-venv` not installed. | `sudo apt install python3-venv`, then delete `venv/` and create it again. |
| `error: externally-managed-environment` | Running `pip install` outside a venv. | Activate the venv: `source venv/bin/activate`. |
| `django-admin: command not found` / `No module named django` | venv not active, or Django not installed in it. | Activate the venv; check with `which python` and `pip list`. |
| `Command 'python' not found` | Outside a venv, Linux uses `python3`. | Use `python3`, or activate the venv. |
| `Error: That port is already in use.` | Another server is running on 8000. | Stop it with `Ctrl + C` in its terminal, or use `runserver 8080`. |
| `ModuleNotFoundError: No module named 'students'` | Typo in folder name or `INSTALLED_APPS`. | Names must match exactly. |
| `Page not found (404)` at `/` | No route defined for the site root. | Visit `/students/`. |
| `TypeError: student_detail() got an unexpected keyword argument` | Converter name ≠ view parameter name. | Make `<int:student_id>` and `def student_detail(request, student_id)` match. |
| `DisallowedHost` | Accessing by IP/domain not in `ALLOWED_HOSTS`. | Add it: `ALLOWED_HOSTS = ['192.168.1.10']`. |

---

## 📋 Command Cheat Sheet

```bash
# One-time setup
sudo apt install -y python3 python3-pip python3-venv
mkdir -p ~/projects/student_management && cd ~/projects/student_management
python3 -m venv venv
source venv/bin/activate
pip install "django>=5.2,<5.3"
pip freeze > requirements.txt
django-admin startproject config .
python manage.py migrate
python manage.py startapp students

# Every session
cd ~/projects/student_management
source venv/bin/activate
python manage.py runserver
```

---

## 📝 Lab Tasks

1. Add a view `home` to `students/views.py` that returns "Welcome to the Student Portal" and map it to `/students/home/`.
2. Add a URL `/students/course/<str:course_code>/` that displays `Course: <course_code>`.
3. Add a URL `/students/<int:student_id>/marks/<int:marks>/` that shows **Pass** if `marks >= 50` and **Fail** otherwise.
4. Create a second app called `courses`, register it, and serve a page at `/courses/` that lists three course names.
5. Make the site root `http://127.0.0.1:8000/` show a welcome page (hint: route `''` in `config/urls.py`).
6. Visit `/students/xyz/` and explain in one line why you get a 404.

**Submission:** Zip your project **without** the `venv/` folder, and include `requirements.txt`.
