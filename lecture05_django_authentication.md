# Lecture 05: Authentication & Authorization

> **Platform:** Ubuntu
> **Project:** Continue with `student_management` from Lecture 04

## Learning Outcomes

By the end of this lecture you will be able to:

1. Explain the difference between **authentication** and **authorization**.
2. Describe how sessions and cookies keep a user logged in.
3. Use Django's built-in `User` model and explain how passwords are stored.
4. Add login and logout using Django's built-in views.
5. Build a registration (sign-up) page.
6. Show different content to logged-in and anonymous users.
7. Protect views with `@login_required`.
8. Control who can do what using staff status, permissions and groups.
9. Link records to the user who created them.

---

## 01. Where We Are

After Lecture 04, **anyone** who opens our site can add, edit and delete students. A real system must know **who** the user is and **what** they are allowed to do.

| Term | Question it answers | Example |
| :--- | :--- | :--- |
| **Authentication** | *Who are you?* | Logging in with a username and password |
| **Authorization** | *What are you allowed to do?* | Only staff can delete a student |

Authentication always comes first: you cannot decide what someone may do until you know who they are.

---

## 02. How Login Works: Sessions and Cookies

HTTP is **stateless**: every request is independent, and the server does not remember the previous one. So how does the site know you are still logged in on the next page?

```text
 1. Browser ── POST /accounts/login/ (username, password) ──> Server
                                                               checks the password
                                                               creates a SESSION in the database
 2. Browser <── response + cookie: sessionid=abc123 ────────── Server

 3. Browser ── GET /students/  + cookie: sessionid=abc123 ───> Server
                                                               finds session abc123 → this is "ali"
                                                               sets request.user = ali
```

* A **session** is a record stored on the **server** (table `django_session`).
* A **cookie** is a small value stored in the **browser** and sent automatically with every request. It holds only the session ID, never the password.
* **Logging out** deletes the session, so the cookie becomes useless.

Django does all of this for you. In every view and template you simply get **`request.user`**.

> See it yourself: after logging in, open the browser's Developer Tools → **Application/Storage → Cookies** and find `sessionid`.

---

## 03. Django's Built-in Authentication System

It is already installed. Look at `config/settings.py`:

```python
INSTALLED_APPS = [
    'django.contrib.auth',           # users, passwords, permissions, groups
    'django.contrib.contenttypes',
    'django.contrib.sessions',       # sessions
    ...
]

MIDDLEWARE = [
    ...
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',   # sets request.user
    ...
]
```

The tables (`auth_user`, `auth_group`, `auth_permission`, `django_session`) were created by the very first `migrate` in Lecture 01. The superuser you created in Lecture 03 is a row in `auth_user`.

### The `User` model

| Field | Meaning |
| :--- | :--- |
| `username` | Unique login name. |
| `password` | The **hashed** password (never the real one). |
| `email`, `first_name`, `last_name` | Optional details. |
| `is_active` | `False` = account disabled, cannot log in. |
| `is_staff` | May log in to the **admin panel**. |
| `is_superuser` | Has **all** permissions automatically. |
| `date_joined`, `last_login` | Set by Django. |

### Passwords are hashed

```bash
python manage.py shell
```

```python
>>> from django.contrib.auth.models import User

>>> user = User.objects.create_user(username='ali', password='Test@12345')
>>> user.password
'pbkdf2_sha256$1000000$Xy7...$k9d...'        # not the real password

>>> user.check_password('Test@12345')
True
>>> user.check_password('wrong')
False
```

* A **hash** is a one-way conversion: the password cannot be recovered from it. Even the administrator and the database owner cannot see users' passwords.
* Always use **`create_user()`**, never `User.objects.create(password='...')`. The second one stores the password as plain text, and the user will not be able to log in.
* To change a password in code: `user.set_password('new')` then `user.save()`.

---

## 04. Login and Logout with Built-in Views

Django ships ready-made views for login, logout and password change. We only supply the URLs and templates.

### Step 4.1: Add the URLs

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('django.contrib.auth.urls')),
    path('students/', include('students.urls')),
]
```

This one line adds:

| URL | Name | Purpose |
| :--- | :--- | :--- |
| `/accounts/login/` | `login` | Login page |
| `/accounts/logout/` | `logout` | Logout (POST only) |
| `/accounts/password_change/` | `password_change` | Change password |
| `/accounts/password_reset/` | `password_reset` | "Forgot password" by email |

> These URL names have **no namespace**: write `{% url 'login' %}`, not `{% url 'accounts:login' %}`.

### Step 4.2: Create the login template

The built-in login view looks for a template named **`registration/login.html`**. Put it in the project-level templates folder:

```bash
mkdir -p templates/registration
```

`templates/registration/login.html`:

```html
{% extends 'base.html' %}

{% block title %}Login{% endblock %}

{% block content %}
<h1>Login</h1>

<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <input type="hidden" name="next" value="{{ next }}">
    <button type="submit">Login</button>
</form>
{% endblock %}
```

The view provides `form` (username and password fields) automatically. A wrong username or password shows an error and the form again.

### Step 4.3: Tell Django where to go after login and logout

```python
# config/settings.py (at the bottom)
LOGIN_URL = 'login'                          # where to send users who must log in
LOGIN_REDIRECT_URL = 'students:student_list' # where to go after a successful login
LOGOUT_REDIRECT_URL = 'login'                # where to go after logout
```

Without `LOGIN_REDIRECT_URL`, Django redirects to `/accounts/profile/`, which does not exist (404).

### Step 4.4: Logout must be a POST form

Logging out changes state, so it follows the same rule as Lecture 04: **POST, not a link**. A plain `<a href="/accounts/logout/">` gives **405 Method Not Allowed**.

```html
<form method="post" action="{% url 'logout' %}" style="display:inline">
    {% csrf_token %}
    <button type="submit">Logout</button>
</form>
```

---

## 05. The `user` Object in Templates

Every template automatically receives **`user`** (the same object as `request.user`).

| Expression | Meaning |
| :--- | :--- |
| `user.is_authenticated` | `True` if logged in. |
| `user.username` | Login name. |
| `user.is_staff` / `user.is_superuser` | Staff / superuser flags. |

If nobody is logged in, `user` is an **`AnonymousUser`**, and `is_authenticated` is `False`.

### Update the navbar

`templates/partials/navbar.html`:

```html
<nav>
    <a href="{% url 'students:student_list' %}">Students</a>

    {% if user.is_authenticated %}
        <span>Hello, {{ user.username }}</span>
        <form method="post" action="{% url 'logout' %}" style="display:inline">
            {% csrf_token %}
            <button type="submit">Logout</button>
        </form>
    {% else %}
        <a href="{% url 'login' %}">Login</a>
        <a href="{% url 'accounts:signup' %}">Sign up</a>
    {% endif %}
</nav>
```

### Hide buttons from anonymous users

```html
{% if user.is_authenticated %}
    <a href="{% url 'students:student_add' %}">+ Add Student</a>
{% endif %}
```

> ⚠️ **Hiding a link is not security.** Anyone can still type `/students/add/` in the address bar. The **view** itself must be protected (Section 07).

---

## 06. Registration (Sign-up)

Django provides the form (`UserCreationForm`) but not the view, so we write it. We will keep user-related code in its own app.

### Step 6.1: Create the `accounts` app

```bash
python manage.py startapp accounts
```

Add `'accounts'` to `INSTALLED_APPS`.

### Step 6.2: The view

```python
# accounts/views.py
from django.contrib import messages
from django.contrib.auth import login
from django.contrib.auth.forms import UserCreationForm
from django.shortcuts import render, redirect


def signup(request):
    if request.method == 'POST':
        form = UserCreationForm(request.POST)
        if form.is_valid():
            user = form.save()                 # creates the user, password hashed
            login(request, user)               # log the new user in immediately
            messages.success(request, f"Welcome, {user.username}!")
            return redirect('students:student_list')
    else:
        form = UserCreationForm()

    return render(request, 'accounts/signup.html', {'form': form})
```

This is exactly the form-view pattern from Lecture 04. `UserCreationForm` is a `ModelForm` for `User` with three fields: **username**, **password** and **password confirmation**. It checks that:

* the username is not already taken,
* the two passwords match,
* the password is strong enough (`AUTH_PASSWORD_VALIDATORS` in `settings.py`: minimum 8 characters, not entirely numeric, not too common, not similar to the username).

**`login(request, user)`** creates the session for that user.

### Step 6.3: Template and URLs

`accounts/templates/accounts/signup.html`:

```html
{% extends 'base.html' %}

{% block title %}Sign up{% endblock %}

{% block content %}
<h1>Create an Account</h1>

<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Sign up</button>
</form>

<p>Already have an account? <a href="{% url 'login' %}">Login</a></p>
{% endblock %}
```

```python
# accounts/urls.py
from django.urls import path
from . import views

app_name = 'accounts'

urlpatterns = [
    path('signup/', views.signup, name='signup'),
]
```

```python
# config/urls.py
urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('accounts.urls')),               # our signup
    path('accounts/', include('django.contrib.auth.urls')),    # built-in login/logout
    path('students/', include('students.urls')),
]
```

Two `include()` lines may share the same prefix. Django tries them in order.

---

## 07. Protecting Views: `@login_required`

```python
# students/views.py
from django.contrib.auth.decorators import login_required


@login_required
def student_add(request):
    ...


@login_required
def student_edit(request, student_id):
    ...


@login_required
def student_delete(request, student_id):
    ...
```

A **decorator** (`@...`) wraps a function with extra behaviour. `@login_required` checks the user **before** the view runs:

```text
 Request ──> @login_required ──┬── logged in?  Yes ──> run the view
                               └── No ──> redirect to /accounts/login/?next=/students/add/
```

### The `next` parameter

When an anonymous user opens `/students/add/`, Django redirects to:

```text
/accounts/login/?next=/students/add/
```

After a successful login, the user is sent back to the page they originally wanted. That is why our login template contains:

```html
<input type="hidden" name="next" value="{{ next }}">
```

`student_list` and `student_detail` stay public, so we do not decorate them.

---

## 08. Authorization: Who Can Do What

Being logged in is not enough. A newly registered user should not be able to delete students.

### 8.1 Checking in the view

```python
from django.core.exceptions import PermissionDenied


@login_required
def student_delete(request, student_id):
    if not request.user.is_staff:
        raise PermissionDenied          # 403 Forbidden

    student = get_object_or_404(Student, id=student_id)
    ...
```

| Status code | Meaning |
| :--- | :--- |
| **401 / redirect to login** | We don't know who you are. |
| **403 Forbidden** | We know who you are, and you are not allowed. |
| **404 Not Found** | The thing does not exist. |

### 8.2 Permissions

For every model, Django automatically creates four permissions:

| Permission codename | Allows |
| :--- | :--- |
| `students.add_student` | Creating students |
| `students.change_student` | Editing students |
| `students.delete_student` | Deleting students |
| `students.view_student` | Viewing students |

Format: **`<app_label>.<action>_<modelname>`** (model name in lowercase).

Protect a view with a permission:

```python
from django.contrib.auth.decorators import login_required, permission_required


@login_required
@permission_required('students.delete_student', raise_exception=True)
def student_delete(request, student_id):
    ...
```

* `raise_exception=True` shows **403** to a logged-in user without the permission (instead of sending them to the login page again).
* Decorators run from **top to bottom**: first login is checked, then the permission.

Check in code or templates:

```python
request.user.has_perm('students.delete_student')     # True / False
```

```html
{% if perms.students.change_student %}
    <a href="{% url 'students:student_edit' s.id %}">Edit</a>
{% endif %}
{% if perms.students.delete_student %}
    <a href="{% url 'students:student_delete' s.id %}">Delete</a>
{% endif %}
```

`perms` is available in every template, like `user`.

### 8.3 Groups

Giving permissions to users one by one does not scale. A **group** is a named set of permissions (a role). Users placed in the group receive all of its permissions.

In the **admin panel** (`/admin/` → Groups):

| Group | Permissions |
| :--- | :--- |
| `Teachers` | add, change, view student |
| `Coordinators` | add, change, delete, view student |

Then open a user in the admin and add them to a group.

```text
 User "ali" ──member of──> Group "Teachers" ──has──> add_student, change_student, view_student
```

> **Superusers** pass every permission check automatically, so test permissions with a **normal** user.

---

## 09. Linking Records to Users

Who added this student? Store the user on the record with a `ForeignKey`.

### Step 9.1: The model

```python
# students/models.py
from django.conf import settings
from django.db import models


class Student(models.Model):
    ...
    created_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='students_created',
    )
```

`settings.AUTH_USER_MODEL` is the recommended way to refer to the user model in a `ForeignKey`.

```bash
python manage.py makemigrations
python manage.py migrate
```

### Step 9.2: Set it in the view, not in the form

`created_by` is **not** in the form's `fields`. If it were, a user could choose somebody else's name. The view sets it from `request.user`:

```python
@login_required
def student_add(request):
    if request.method == 'POST':
        form = StudentForm(request.POST)
        if form.is_valid():
            student = form.save(commit=False)     # build the object, don't save yet
            student.created_by = request.user     # fill the missing field
            student.save()                        # now save
            messages.success(request, f"Student {student.name} added successfully.")
            return redirect('students:student_list')
    else:
        form = StudentForm()

    return render(request, 'students/student_form.html', {'form': form})
```

**`form.save(commit=False)`** returns the object **without** writing it to the database, so you can set extra fields first.

### Step 9.3: Use it

```html
<p>Added by: {{ student.created_by.username|default:"Unknown" }}</p>
```

```python
# "My students" page
@login_required
def my_students(request):
    students = Student.objects.filter(created_by=request.user)
    return render(request, 'students/student_list.html', {'students': students})
```

### Step 9.4: Only the owner (or staff) may edit

```python
@login_required
def student_edit(request, student_id):
    student = get_object_or_404(Student, id=student_id)

    if student.created_by != request.user and not request.user.is_staff:
        raise PermissionDenied

    ...
```

This is **object-level** authorization: the rule depends on **which** record is being changed, not only on the user's role.

---

## 10. Summary of Protection Levels

| Level | Tool | Example |
| :--- | :--- | :--- |
| Public | (nothing) | Student list, detail |
| Logged-in users only | `@login_required` | Add a student |
| Specific role | `@permission_required`, groups, `is_staff` | Delete a student |
| Specific record | Check in the view + `PermissionDenied` | Edit only your own entries |
| Template | `user.is_authenticated`, `perms` | Hide buttons (convenience only) |

---

## 🚨 Troubleshooting

| Error | Cause | Fix |
| :--- | :--- | :--- |
| `TemplateDoesNotExist: registration/login.html` | Login template missing or in the wrong folder. | Create `templates/registration/login.html` and check `DIRS` in settings. |
| 404 at `/accounts/profile/` after login | `LOGIN_REDIRECT_URL` not set. | Set it in `settings.py`. |
| `405 Method Not Allowed` on logout | Logout requested with a link (`GET`). | Use a `POST` form with `{% csrf_token %}`. |
| Logout shows the admin's "Logged out" page | `LOGOUT_REDIRECT_URL` not set. | Set it in `settings.py`. |
| `NoReverseMatch: 'accounts' is not a registered namespace` | Used `accounts:login`; built-in URLs have no namespace. | Use `{% url 'login' %}` and `{% url 'logout' %}`. |
| User created in the shell cannot log in | Created with `User.objects.create(...)`, password not hashed. | Use `create_user()` or `set_password()`. |
| Redirect loop back to the login page | `@permission_required` without `raise_exception=True` for a logged-in user. | Add `raise_exception=True`. |
| Permission checks always pass | Testing with a superuser. | Test with a normal user. |
| `NOT NULL constraint failed: students_student.created_by_id` | `created_by` not set before saving. | Use `form.save(commit=False)`, set the user, then `save()`. |
| Not redirected back after login | Hidden `next` input missing from the login form. | Add it to `login.html`. |

---

## 📋 Quick Reference

```text
URLS (config/urls.py)
  path('accounts/', include('django.contrib.auth.urls'))     → login, logout, password_change

SETTINGS
  LOGIN_URL = 'login'
  LOGIN_REDIRECT_URL = 'students:student_list'
  LOGOUT_REDIRECT_URL = 'login'

TEMPLATES
  templates/registration/login.html
  {% if user.is_authenticated %} {{ user.username }} {% endif %}
  {% if perms.students.delete_student %} ... {% endif %}
  <form method="post" action="{% url 'logout' %}">{% csrf_token %}<button>Logout</button></form>

VIEWS
  from django.contrib.auth.decorators import login_required, permission_required
  from django.core.exceptions import PermissionDenied

  @login_required
  @permission_required('students.delete_student', raise_exception=True)
  def student_delete(request, student_id): ...

  request.user            request.user.is_staff          request.user.has_perm('app.codename')
  raise PermissionDenied  → 403

SIGN-UP
  form = UserCreationForm(request.POST) → user = form.save() → login(request, user)

OWNERSHIP
  created_by = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.SET_NULL, null=True)
  obj = form.save(commit=False); obj.created_by = request.user; obj.save()

SHELL
  User.objects.create_user(username='ali', password='...')
  user.set_password('new'); user.save()
  python manage.py changepassword <username>
```

---

## 🔜 Next Lecture

**Class-Based Views & Pagination:** writing list, detail, create, update and delete pages with Django's generic views, and splitting long lists into pages.
