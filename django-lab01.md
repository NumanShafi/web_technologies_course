# Lab 01: Introduction to Django, Environment Setup, Views & URLs

## 01. Introduction to Django

### What is Django?
Django is a high-level, open-source Python web framework designed to enable rapid application development and clean design. It handles much of the boilerplate work of web development, allowing you to focus on writing your application's unique features.

### Framework Architecture (MVT Pattern)
Django uses the **MVT (Model-View-Template)** architectural pattern:

```text
+-------------------------------------------------------------------------------+
|                             HTTP REQUEST FLOW                                 |
|                                                                               |
|  [ User Browser ] ---> ( Request ) ---> [ Project URLs (urls.py) ]            |
|                                                     |                         |
|                                                     v                         |
|  [ User Browser ] <-- ( Response ) <--- [ App View (views.py) ]               |
|                                           (Returns HttpResponse)              |
+-------------------------------------------------------------------------------+
```

* **Model:** Defines the data structures and database interaction rules.
* **View:** Contains the business logic, handles incoming HTTP requests, and returns HTTP responses.
* **Template:** Defines the presentation layer (HTML layout and formatting).
* **URL Router:** Maps incoming request URLs (e.g., `/students/`) to specific View functions.

---

## 02. Getting Started & System Prerequisites

Before installing Django, ensure Python and terminal tools are ready.

### Terminal Commands & Options

#### 1. Check Installed Python Version
```bash
python3 --version
# On Windows Command Prompt:
python --version
```
* **Explanation:** Verifies the system Python version. Django requires **Python 3.10** or higher.

#### 2. Check Package Manager (`pip`)
```bash
python3 -m pip --version
```
* **`-m pip`:** Invokes `pip` directly through Python, ensuring you run the package manager associated with that specific Python installation.

---

## 03. Setting Up a Virtual Environment (`venv`)

A **Virtual Environment** is an isolated workspace containing its own Python interpreter and package directory. This ensures project dependencies do not conflict with system packages or other Python projects on your machine.

### Step-by-Step Commands & Flag Explanations

#### Step 3.1: Create Project Root Directory
```bash
mkdir -p ~/projects/student_management
cd ~/projects/student_management
```
* **`mkdir -p`:** Creates parent directories as needed without throwing an error if they already exist.
* **`cd`:** Changes working directory into `~/projects/student_management`.

#### Step 3.2: Create Virtual Environment
```bash
# On Linux / macOS:
python3 -m venv venv

# On Windows:
py -m venv venv
```

##### Breakdown of `python3 -m venv venv`:
* **`python3`:** The system Python interpreter.
* **`-m` (Module Flag):** Runs a built-in library module as a script.
* **`venv` (First):** The Python standard library module responsible for building isolated environments.
* **`venv` (Second):** The directory name where the virtual environment files will be stored.

#### Optional Creation Flags for `venv`:
| Flag | Description | Example Usage |
| :--- | :--- | :--- |
| `--clear` | Deletes existing contents of the environment folder before creating. | `python3 -m venv --clear venv` |
| `--system-site-packages` | Gives the environment access to global system packages. | `python3 -m venv --system-site-packages venv` |
| `--copies` | Copies binaries instead of symlinking them. | `python3 -m venv --copies venv` |

#### Step 3.3: Activate Virtual Environment
Activation redirects your shell to use the virtual environment's Python and `pip` binaries.

```bash
# On Linux / macOS (Bash/Zsh):
source venv/bin/activate

# On Windows (Command Prompt - cmd.exe):
venv\Scripts\activate.bat

# On Windows (PowerShell):
venv\Scripts\Activate.ps1
```
* **`source`:** Executes script commands within the current shell context.


#### Step 3.4: Deactivating Environment
```bash
deactivate
```
* **Explanation:** Returns your terminal shell back to the system's global Python environment.

---

## 04. Installing Django

Install Django inside the active `venv` environment.

### Commands, Options, and Examples

#### 1. Upgrade `pip` Executable Inside Environment
```bash
python -m pip install --upgrade pip
```
* **`install`:** Command to download and install packages from PyPI (Python Package Index).
* **`--upgrade` (or `-U`):** Upgrades the specified package to its latest available version.

#### 2. Install Django
```bash
pip install django
```

##### Useful `pip install` Options:
| Command Option | Purpose | Example |
| :--- | :--- | :--- |
| `django==5.1.2` | Installs an exact version. | `pip install django==5.1.2` |
| `django>=5.0,<6.0` | Installs a version within a specified version range. | `pip install "django>=5.0,<6.0"` |
| `--no-cache-dir` | Forces a fresh download without using cached files. | `pip install --no-cache-dir django` |

#### 3. Verify Django Installation
```bash
python -m django --version
```
* **Output:** `5.1.x` (confirms Django is correctly imported inside `venv`).

---

## 05. Creating a Django Project (`django-admin startproject`)

In Django, a **Project** is a collection of configuration settings, routes, and WSGI/ASGI server definitions for an application site.

### Command Execution
```bash
django-admin startproject config .
```

### Command & Arguments Breakdown:
* **`django-admin`:** Django's command-line utility for administrative tasks.
* **`startproject`:** Subcommand that generates standard project boilerplate files.
* **`config`:** The directory name for your core configuration package.
* **`.` (Trailing Dot):** Tells Django to place files directly in the **current working directory** rather than nesting them inside an extra subfolder layer.

---

### File-by-File Breakdown of Generated Project

After running `startproject`, your directory layout looks like this:

```text
student_management/
│
├── config/               # Core Project Configuration Package
│   ├── __init__.py       # Empty file marking folder as a Python package
│   ├── settings.py       # Master project configuration settings
│   ├── urls.py           # Top-level URL dispatcher table
│   ├── asgi.py           # Entry point for ASGI async web servers
│   └── wsgi.py           # Entry point for WSGI sync web servers
│
├── manage.py             # Command-line driver for project tasks
└── venv/                 # Virtual environment directory
```

#### 1. `manage.py`
A CLI utility configured specifically for your project. You execute this file for local development tasks.

```python
# manage.py (Simplified)
import os
import sys

def main():
    """Run administrative tasks."""
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError("Couldn't import Django...") from exc
    execute_from_command_line(sys.argv)

if __name__ == '__main__':
    main()
```
* **Function:** Directs Python to load `config.settings` and parses CLI inputs (such as `runserver`, `startapp`).

#### 2. `config/__init__.py`
An empty file that informs Python to treat `config/` as an importable package directory.

#### 3. `config/settings.py`
Contains configuration parameters for your Django instance.

```python
# Key sections in config/settings.py:

# 1. Base directory path of project
BASE_DIR = Path(__file__).resolve().parent.parent

# 2. Secret security key (keep hidden in production!)
SECRET_KEY = 'django-insecure-example-key...'

# 3. Debug mode (True for development, MUST be False in production)
DEBUG = True

# 4. Allowed hostnames/IP addresses when DEBUG=False
ALLOWED_HOSTS = []

# 5. Registered application packages
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```

#### 4. `config/urls.py`
The master routing entry point for incoming HTTP requests.

```python
# config/urls.py (Initial Boilerplate)
from django.contrib import admin
from django.urls import path

urlpatterns = [
    path('admin/', admin.site.urls),
]
```

#### 5. `config/wsgi.py`
*(Web Server Gateway Interface)* Entry point used by standard synchronous production servers (Gunicorn, uWSGI, Apache).

#### 6. `config/asgi.py`
*(Asynchronous Server Gateway Interface)* Entry point used by asynchronous servers (Daphne, Uvicorn) to handle protocols like WebSockets.

---

## 06. Running the Development Server & Options

Django includes a built-in development web server for local testing. **Never use this development server in production environments.**

### Basic Server Execution
```bash
python manage.py runserver
```

* **Default Behavior:** Starts the server on `http://127.0.0.1:8000/` (Localhost, Port 8000).

---

### Command-Line Options for `runserver`

| Command / Option | Description & Use Case | Example |
| :--- | :--- | :--- |
| `python manage.py runserver <port>` | Runs server on a custom port if 8000 is occupied. | `python manage.py runserver 8080` |
| `python manage.py runserver <ip>:<port>` | Binds to an explicit IP address and custom port. | `python manage.py runserver 127.0.0.1:9000` |
| `python manage.py runserver 0.0.0.0:8000` | Binds server to all network interfaces (accessible by other devices on local Wi-Fi). | `python manage.py runserver 0.0.0.0:8000` |
| `--noreload` | Disables the auto-reloader (prevents server from auto-restarting on code changes). | `python manage.py runserver --noreload` |
| `--nothreading` | Disables multithreading in server execution. | `python manage.py runserver --nothreading` |
| `--settings=<path>` | Explicitly specifies an alternative settings file. | `python manage.py runserver --settings=config.settings_dev` |

#### Stopping the Server:
Press `CTRL + C` in your active terminal.

---

## 07. Creating a Django App (`python manage.py startapp`)

A **Project** contains global site configuration, while an **App** is a modular package that handles a specific set of features (e.g., `students`, `courses`, `grades`). A single project can house multiple apps.

### Command Execution
```bash
python manage.py startapp students
```
* **`startapp`:** Generates an app package folder named `students/`.

---

### Registering the App in `settings.py`
Open `config/settings.py` and add `'students'` to `INSTALLED_APPS`:

```python
# config/settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Local Apps
    'students',
]
```

---

### File-by-File Breakdown of Generated App

Running `startapp students` creates the following files inside `students/`:

```text
students/
├── __init__.py       # Marks folder as a Python package
├── admin.py          # Admin interface registrations
├── apps.py           # App configuration settings
├── migrations/       # Database migration tracking files
│   └── __init__.py
├── models.py         # Database schema model definitions
├── tests.py          # Unit testing logic
└── views.py          # HTTP request handlers & view functions
```

#### 1. `students/apps.py`
Contains configuration metadata for the `students` application.

```python
# students/apps.py
from django.apps import AppConfig

class StudentsConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'students'
```
* **Function:** Configures app metadata (such as primary key default fields and internal app name).

#### 2. `students/views.py`
Houses your view logic functions. Views process incoming HTTP requests and return appropriate HTTP responses.

#### 3. `students/admin.py`
Used to register models to make them manageable within Django's built-in administrative interface.

#### 4. `students/models.py`
Defines your application database structures using Python classes.

#### 5. `students/tests.py`
Houses automated unit testing functions.

#### 6. `students/migrations/`
Directory that stores schema version history files generated by Django.

---

## 08. Django Views (`HttpResponse`)

A **View** function accepts an HTTP request object (`request`), runs business logic, and returns an HTTP response object.

### Writing a View
Open `students/views.py` and replace its contents:

```python
# students/views.py

from django.http import HttpResponse

def student_list(request):
    """
    Sample View Function.
    Param: request -> HttpRequest object containing metadata (headers, request method)
    Returns: HttpResponse -> HTML payload returned to the browser.
    """
    html_content = """
    <!DOCTYPE html>
    <html>
    <head>
        <title>Student Directory</title>
    </head>
    <body>
        <h1 style="color: #2c3e50;">Student Management System</h1>
        <p>Welcome to the Student Portal Directory.</p>
    </body>
    </html>
    """
    return HttpResponse(html_content)
```

---

## 09. Django URLs (Two-Tier Routing)

To make a view accessible via the browser, map the view function to a URL pattern using a **two-tier URL architecture**:
1. App-level routes defined in `students/urls.py`.
2. Project-level routes defined in `config/urls.py`.

```text
+-------------------------------------------------------------------------------+
|                           TWO-TIER ROUTING FLOW                               |
|                                                                               |
| [ Browser ] -> Request "http://127.0.0.1:8000/students/"                      |
|                      |                                                        |
|                      v                                                        |
|        [ config/urls.py ]     -- path('students/', include('students.urls'))  |
|                      |                                                        |
|                      v                                                        |
|        [ students/urls.py ]   -- path('', views.student_list)                 |
|                      |                                                        |
|                      v                                                        |
|        [ views.student_list() ] --> Returns HttpResponse                      |
+-------------------------------------------------------------------------------+
```

### Step 9.1: Create App-Level `urls.py`
Create a new file `students/urls.py`:

```bash
touch students/urls.py
```

Add the following code to `students/urls.py`:

```python
# students/urls.py

from django.urls import path
from . import views

urlpatterns = [
    # Route: /students/
    path('', views.student_list, name='student_list'),
]
```

#### Line-by-Line Explanation of `path()`:
* **`path(route, view, kwargs=None, name=None)`**
  * `route`: String matching the URL pattern (`''` matches root relative to the app mount point).
  * `view`: Target view function to execute (`views.student_list`).
  * `name`: Optional string name allowing reverse URL lookups in templates and code.

### Step 9.2: Include App Routes in `config/urls.py`
Open `config/urls.py` and connect `students.urls` using `include`:

```python
# config/urls.py

from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('students/', include('students.urls')),  # Mounts students/urls.py at /students/
]
```

#### Line-by-Line Explanation of `include()`:
* **`include('students.urls')`:** Truncates the URL prefix matched so far (`students/`) and forwards the remainder of the URL path to `students.urls` for matching.

---

### Step 9.3: Run Server & Verify Setup
Launch the development server:

```bash
python manage.py runserver 8080
```

Open your browser and navigate to:
```text
http://127.0.0.1:8080/students/
```

*Expected Browser Output:*
> **Student Management System**  
> Welcome to the Student Portal Directory.

---

## 🚨 Troubleshooting & Common Errors

| Error Message | Cause | Resolution |
| :--- | :--- | :--- |
| `That port is already in use.` | Another process is using port `8000`. | Specify a different port: `python manage.py runserver 8085`. |
| `ModuleNotFoundError: No module named 'students'` | App is not listed in `INSTALLED_APPS` or folder name has a typo. | Verify app folder name and list `'students'` in `config/settings.py`. |
| `Page not found (404)` | Accessing `http://127.0.0.1:8000/` instead of `/students/`. | Navigate to the full path: `http://127.0.0.1:8000/students/`. |
| `django-admin: command not found` | Terminal executing outside an active virtual environment. | Reactivate environment using `source venv/bin/activate` or `venv\Scripts\activate`. |