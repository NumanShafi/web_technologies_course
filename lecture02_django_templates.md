# Lecture 02: Django Templates, Template Inheritance & Static Files

> **Platform:** Ubuntu
> **Project:** Continue with `student_management` from Lecture 01

## Learning Outcomes

By the end of this lecture you will be able to:

1. Explain why HTML should not be written inside views.
2. Create templates and render them with `render()`.
3. Send data from a view to a template using a **context**.
4. Use the Django Template Language: variables, filters, `if` and `for`.
5. Avoid repeating HTML using **template inheritance** and `include`.
6. Build links with the `{% url %}` tag instead of hard-coded paths.
7. Add CSS and images using **static files**.

---

## 01. The Problem with `HttpResponse` + HTML Strings

In Lecture 01 we wrote HTML directly inside Python:

```python
def student_list(request):
    return HttpResponse("<h1>Student Management System</h1><p>Welcome...</p>")
```

This becomes a problem quickly:

* A full page with a navbar, footer and CSS becomes hundreds of lines inside Python.
* Every page repeats the same navbar and footer.
* Designers can't edit the HTML without touching Python code.
* No syntax highlighting or HTML help from the editor.

**Solution:** Keep HTML in separate **template** files. The view prepares the data; the template displays it.

```text
           View (views.py)                    Template (.html)
   ------------------------------      ------------------------------
   Business logic, prepares data  -->  Presentation: how data looks
              context = {'name': 'Ali'}       <h1>{{ name }}</h1>
```

This is the **T** in **MVT**.

---

## 02. Setting Up the Templates Folder

### Step 2.1: Check `TEMPLATES` in `settings.py`

Open `config/settings.py`. The `TEMPLATES` setting already exists:

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [ ... ],
        },
    },
]
```

| Key | Meaning |
| :--- | :--- |
| `BACKEND` | The template engine (Django Template Language). |
| `DIRS` | Extra folders to search for templates (project-level templates). |
| `APP_DIRS: True` | Django also searches a folder named `templates/` **inside every installed app**. |

### Step 2.2: Create the app's template folder

```bash
mkdir -p students/templates/students
```

```text
students/
├── templates/
│   └── students/          # ← same name as the app (namespace)
│       ├── student_list.html
│       └── student_detail.html
├── views.py
└── urls.py
```

#### Why `templates/students/` and not just `templates/`?

Django combines the `templates/` folders of **all** apps into one search list. If both `students` and `courses` have a `list.html`, Django uses whichever it finds first, which may be the wrong one.
Adding a subfolder with the app's name makes the path unique: `students/student_list.html` vs `courses/list.html`.

### Step 2.3: Project-level templates folder (for shared files)

Files used by **all** apps, such as the base layout, go in a project-level folder:

```bash
mkdir templates
```

Tell Django about it in `config/settings.py`:

```python
'DIRS': [BASE_DIR / 'templates'],
```

```text
student_management/
├── config/
├── students/
│   └── templates/students/   # App-specific templates
├── templates/                # Shared templates (base.html, navbar.html)
└── manage.py
```

---

## 03. Rendering a Template with `render()`

### Step 3.1: Create the template

`students/templates/students/student_list.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Student Directory</title>
</head>
<body>
    <h1>Student Management System</h1>
    <p>Welcome to the Student Portal.</p>
</body>
</html>
```

### Step 3.2: Update the view

```python
# students/views.py
from django.shortcuts import render


def student_list(request):
    return render(request, 'students/student_list.html')
```

**`render(request, template_name, context=None)`**

| Argument | Meaning |
| :--- | :--- |
| `request` | The incoming request object. Always pass it. |
| `template_name` | Path of the template **relative to a `templates/` folder**. |
| `context` | A dictionary of data to send to the template (optional). |

`render()` loads the template, fills in the data, and returns an `HttpResponse`.

> **Note:** You don't write `students/templates/students/student_list.html`. Django already knows to look inside `templates/`, so you write only `students/student_list.html`.

---

## 04. Passing Data: The Context

A **context** is a Python dictionary. Each **key** becomes a variable name inside the template.

```python
# students/views.py
from django.shortcuts import render


def student_list(request):
    context = {
        'title': 'Student Directory',
        'total': 3,
        'students': ['Ali Raza', 'Ayesha Khan', 'Bilal Ahmed'],
    }
    return render(request, 'students/student_list.html', context)
```

```html
<h1>{{ title }}</h1>
<p>Total students: {{ total }}</p>
```

Output:

```text
Student Directory
Total students: 3
```

> Until we learn **Models** (database) in a later lecture, we'll store data in Python lists and dictionaries inside `views.py`.

---

## 05. Django Template Language (DTL)

Templates are HTML plus three special kinds of syntax:

| Syntax | Name | Purpose | Example |
| :--- | :--- | :--- | :--- |
| `{{ ... }}` | Variable | Prints a value | `{{ title }}` |
| `{% ... %}` | Tag | Logic: loops, conditions, loading | `{% if %}`, `{% for %}` |
| `{# ... #}` | Comment | Not shown in the output | `{# TODO #}` |

### 5.1 Variables and the Dot (`.`)

The dot works for dictionary keys, object attributes and list indexes:

```python
context = {
    'student': {'name': 'Ali Raza', 'roll_no': 'BSCS-01', 'cgpa': 3.45},
    'courses': ['Web Tech', 'Databases', 'OS'],
}
```

```html
<p>Name: {{ student.name }}</p>          {# dictionary key  → Ali Raza #}
<p>Roll No: {{ student.roll_no }}</p>    {# dictionary key  → BSCS-01 #}
<p>First course: {{ courses.0 }}</p>     {# list index      → Web Tech #}
```

> In Python you'd write `student['name']` and `courses[0]`. In templates, always use a **dot**.
> If a variable doesn't exist, Django prints **nothing**. It doesn't raise an error, so check spelling carefully.

### 5.2 Filters — Formatting Values

A **filter** changes how a value is displayed. Syntax: `{{ variable|filter }}` or `{{ variable|filter:"argument" }}`

| Filter | Example | Output |
| :--- | :--- | :--- |
| `upper` | `{{ "ali"\|upper }}` | `ALI` |
| `lower` | `{{ "ALI"\|lower }}` | `ali` |
| `title` | `{{ "ali raza"\|title }}` | `Ali Raza` |
| `length` | `{{ students\|length }}` | `3` |
| `default` | `{{ phone\|default:"N/A" }}` | `N/A` if empty |
| `floatformat` | `{{ 3.4567\|floatformat:2 }}` | `3.46` |
| `truncatewords` | `{{ bio\|truncatewords:5 }}` | First 5 words + `…` |
| `date` | `{{ today\|date:"d M Y" }}` | `11 Nov 2026` |
| `join` | `{{ courses\|join:", " }}` | `Web Tech, Databases, OS` |
| `pluralize` | `student{{ total\|pluralize }}` | `student` / `students` |

Filters can be chained, left to right:

```html
{{ student.name|lower|title }}
```

### 5.3 The `if` Tag

```html
{% if student.cgpa >= 3.5 %}
    <p>Dean's Honour List</p>
{% elif student.cgpa >= 2.0 %}
    <p>Good Standing</p>
{% else %}
    <p>Probation</p>
{% endif %}
```

* Supports `==`, `!=`, `<`, `>`, `<=`, `>=`, `and`, `or`, `not`, `in`.
* Every `{% if %}` **must** close with `{% endif %}`.

Checking whether a list is empty:

```html
{% if students %}
    <p>We have {{ students|length }} students.</p>
{% else %}
    <p>No students yet.</p>
{% endif %}
```

### 5.4 The `for` Tag

```html
<ul>
    {% for name in students %}
        <li>{{ name }}</li>
    {% empty %}
        <li>No students found.</li>
    {% endfor %}
</ul>
```

* `{% empty %}` runs when the list has no items (optional).
* Every `{% for %}` **must** close with `{% endfor %}`.

#### The `forloop` variable

Inside a loop, Django provides `forloop`:

| Variable | Value |
| :--- | :--- |
| `forloop.counter` | 1, 2, 3, … |
| `forloop.counter0` | 0, 1, 2, … |
| `forloop.first` | `True` on the first item |
| `forloop.last` | `True` on the last item |

### 5.5 Example: Students Table

```python
# students/views.py
STUDENTS = [
    {'id': 1, 'name': 'Ali Raza',    'roll_no': 'BSCS-01', 'cgpa': 3.65},
    {'id': 2, 'name': 'Ayesha Khan', 'roll_no': 'BSCS-02', 'cgpa': 3.10},
    {'id': 3, 'name': 'Bilal Ahmed', 'roll_no': 'BSCS-03', 'cgpa': 1.85},
]


def student_list(request):
    context = {'students': STUDENTS}
    return render(request, 'students/student_list.html', context)
```

```html
<h1>Students ({{ students|length }})</h1>

<table border="1">
    <tr>
        <th>#</th><th>Name</th><th>Roll No</th><th>CGPA</th><th>Status</th>
    </tr>
    {% for s in students %}
    <tr>
        <td>{{ forloop.counter }}</td>
        <td>{{ s.name }}</td>
        <td>{{ s.roll_no }}</td>
        <td>{{ s.cgpa|floatformat:2 }}</td>
        <td>
            {% if s.cgpa >= 2.0 %}Good Standing{% else %}Probation{% endif %}
        </td>
    </tr>
    {% empty %}
    <tr><td colspan="5">No students found.</td></tr>
    {% endfor %}
</table>
```

> **Logic belongs in the view.** Templates are for simple display decisions only (show/hide, loop). Calculations such as fines, totals and sorting should be done in the view and passed in the context.

---

## 06. Template Inheritance (`extends` and `block`)

Every page needs the same `<head>`, navbar and footer. Copy-pasting them into every template means that when the navbar changes, you edit 20 files.

**Solution:** Write the common layout **once** in a base template, and let other pages **extend** it and fill in only the parts that differ.

```text
            base.html  (layout: head, navbar, footer)
            ┌──────────────────────────────┐
            │  Navbar                      │
            │  {% block content %}         │ ← "hole" to be filled
            │  {% endblock %}              │
            │  Footer                      │
            └──────────────────────────────┘
                 ▲                    ▲
        extends  │                    │  extends
  student_list.html            student_detail.html
  fills "content" block        fills "content" block
```

### Step 6.1: Create the base template

`templates/base.html` (project-level folder):

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Student Portal{% endblock %}</title>
</head>
<body>
    <nav>
        <a href="/students/">Students</a> |
        <a href="/students/about/">About</a>
    </nav>
    <hr>

    <main>
        {% block content %}
        {% endblock %}
    </main>

    <hr>
    <footer>&copy; 2026 Student Management System</footer>
</body>
</html>
```

* **`{% block name %}...{% endblock %}`** defines a named area that child templates can replace.
* Text inside a block in the base (e.g. `Student Portal`) is the **default**, used if the child doesn't override it.

### Step 6.2: Extend it in a child template

`students/templates/students/student_list.html`:

```html
{% extends 'base.html' %}

{% block title %}Students{% endblock %}

{% block content %}
    <h1>Students ({{ students|length }})</h1>
    <ul>
        {% for s in students %}
            <li>{{ s.name }} — {{ s.roll_no }}</li>
        {% endfor %}
    </ul>
{% endblock %}
```

**Rules:**

1. `{% extends %}` must be the **very first** tag in the file.
2. Anything in the child **outside** a `{% block %}` is ignored.
3. Block names in the child must match block names in the base.
4. Use `{{ block.super }}` inside a block to keep the parent's content and add to it.

---

## 07. Reusable Pieces with `{% include %}`

`include` inserts another template file at that point. It's useful for small parts such as a navbar, a card or a message box.

`templates/partials/navbar.html`:

```html
<nav>
    <a href="/students/">Students</a> |
    <a href="/students/about/">About</a>
</nav>
```

In `base.html`, replace the `<nav>` block with:

```html
{% include 'partials/navbar.html' %}
```

| `extends` | `include` |
| :--- | :--- |
| "This page **is built on** that layout" | "Put **that piece** here" |
| One per template, first line | Any number, anywhere |

---

## 08. Named URLs and the `{% url %}` Tag

Hard-coded links like `<a href="/students/">` break when you change a URL in `urls.py`, and then you have to search every template.
In Lecture 01 we gave every path a `name`. Now we use it.

### Step 8.1: Add an `app_name` (URL namespace)

```python
# students/urls.py
from django.urls import path
from . import views

app_name = 'students'

urlpatterns = [
    path('', views.student_list, name='student_list'),
    path('<int:student_id>/', views.student_detail, name='student_detail'),
    path('about/', views.about, name='about'),
]
```

`app_name` prevents name clashes. Two apps can both have a URL named `list`, referred to as `students:list` and `courses:list`.

### Step 8.2: Use `{% url %}` in templates

```html
<a href="{% url 'students:student_list' %}">All Students</a>
<a href="{% url 'students:about' %}">About</a>

{# URL with a parameter #}
<a href="{% url 'students:student_detail' s.id %}">{{ s.name }}</a>
```

| Template code | Generated link |
| :--- | :--- |
| `{% url 'students:student_list' %}` | `/students/` |
| `{% url 'students:student_detail' 2 %}` | `/students/2/` |

Now, if you change `path('students/', ...)` to `path('learners/', ...)` in `config/urls.py`, **every link updates automatically**.

---

## 09. Detail Page & Handling "Not Found"

```python
# students/views.py
from django.http import Http404
from django.shortcuts import render


def student_detail(request, student_id):
    student = None
    for s in STUDENTS:
        if s['id'] == student_id:
            student = s
            break

    if student is None:
        raise Http404("Student not found")

    return render(request, 'students/student_detail.html', {'student': student})
```

`students/templates/students/student_detail.html`:

```html
{% extends 'base.html' %}

{% block title %}{{ student.name }}{% endblock %}

{% block content %}
    <h1>{{ student.name }}</h1>
    <p><strong>Roll No:</strong> {{ student.roll_no }}</p>
    <p><strong>CGPA:</strong> {{ student.cgpa|floatformat:2 }}</p>
    <a href="{% url 'students:student_list' %}">← Back to list</a>
{% endblock %}
```

* **`raise Http404(...)`** returns a proper **404 Not Found** page instead of an empty page or a crash.
* `/students/2/` shows Ayesha's details; `/students/99/` shows 404.

---

## 10. Static Files (CSS, JavaScript, Images)

**Static files** are files sent to the browser **as they are**, without any processing: CSS, JavaScript, images and fonts.

### Step 10.1: Check `settings.py`

```python
INSTALLED_APPS = [
    ...
    'django.contrib.staticfiles',   # already present
]

STATIC_URL = 'static/'              # URL prefix for static files
```

### Step 10.2: Create the folder (same namespacing idea as templates)

```bash
mkdir -p students/static/students/css
mkdir -p students/static/students/images
```

```text
students/
├── static/
│   └── students/
│       ├── css/
│       │   └── style.css
│       └── images/
│           └── logo.png
└── templates/
```

`students/static/students/css/style.css`:

```css
body {
    font-family: Arial, sans-serif;
    margin: 0;
    background: #f4f6f8;
}
nav {
    background: #2c3e50;
    padding: 12px;
}
nav a {
    color: white;
    margin-right: 15px;
    text-decoration: none;
}
main {
    padding: 20px;
}
table {
    border-collapse: collapse;
    width: 100%;
    background: white;
}
th, td {
    padding: 8px;
    border: 1px solid #ddd;
}
```

### Step 10.3: Load static files in the template

`templates/base.html`:

```html
{% load static %}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Student Portal{% endblock %}</title>
    <link rel="stylesheet" href="{% static 'students/css/style.css' %}">
</head>
<body>
    {% include 'partials/navbar.html' %}
    <main>
        <img src="{% static 'students/images/logo.png' %}" alt="Logo" width="80">
        {% block content %}{% endblock %}
    </main>
    <footer>&copy; 2026 Student Management System</footer>
</body>
</html>
```

* **`{% load static %}`** must appear in **every template** that uses `{% static %}`. It is **not** inherited from the base.
* **`{% static 'path' %}`** builds the URL, e.g. `/static/students/css/style.css`.

> ⚠️ **Restart the server** after creating a new `static/` folder. Django doesn't detect new static folders while it's running.
> The development server serves static files only when `DEBUG = True`. In production a web server (e.g. Nginx) does it. We'll cover that during deployment.

---

## 11. Final Structure After This Lecture

```text
student_management/
├── config/
│   ├── settings.py          # DIRS = [BASE_DIR / 'templates']
│   └── urls.py
├── students/
│   ├── static/students/
│   │   ├── css/style.css
│   │   └── images/logo.png
│   ├── templates/students/
│   │   ├── student_list.html
│   │   └── student_detail.html
│   ├── urls.py               # app_name = 'students'
│   └── views.py              # render(), context, Http404
├── templates/
│   ├── base.html
│   └── partials/navbar.html
├── db.sqlite3
├── manage.py
└── requirements.txt
```

---

## 🚨 Troubleshooting

| Error | Cause | Fix |
| :--- | :--- | :--- |
| `TemplateDoesNotExist` | Wrong path, wrong folder name (`template` instead of `templates`), or app not in `INSTALLED_APPS`. | The error page lists every folder Django searched (**Template-loader postmortem**); compare it with your actual path. |
| `TemplateSyntaxError: Invalid block tag ... 'endfor'` | Missing or misspelled closing tag. | Every `{% if %}` / `{% for %}` / `{% block %}` needs its `{% end... %}`. |
| `Invalid block tag: 'static'` | `{% load static %}` missing in that template. | Add `{% load static %}` at the top of **that** file. |
| Variable shows nothing | Misspelled variable or key not in context. | Check the key spelling in the view's context dictionary. |
| `NoReverseMatch` | Wrong URL name, missing `app_name` namespace, or missing parameter in `{% url %}`. | Check `app_name`, `name=`, and that you passed the required argument. |
| CSS not applied (404 in browser) | Wrong static path, or server not restarted. | Open `http://127.0.0.1:8000/static/students/css/style.css` directly. If 404, check the folder path and restart. |
| Old CSS still showing | Browser cache. | Hard refresh: `Ctrl + Shift + R`. |
| Child page shows only base layout | Content written **outside** a `{% block %}`, or block names don't match. | Put content inside `{% block content %}...{% endblock %}`. |

---

## 📋 Quick Reference

```text
VIEW
  from django.shortcuts import render
  return render(request, 'app/page.html', {'key': value})

TEMPLATE SYNTAX
  {{ var }}   {{ dict.key }}   {{ list.0 }}   {{ var|filter:"arg" }}
  {% if x %} ... {% elif y %} ... {% else %} ... {% endif %}
  {% for item in items %} ... {% empty %} ... {% endfor %}
  {# comment #}

INHERITANCE
  base:  {% block content %}{% endblock %}
  child: {% extends 'base.html' %}
         {% block content %} ... {% endblock %}
  piece: {% include 'partials/navbar.html' %}

URLS
  urls.py:  app_name = 'students'   path(..., name='student_detail')
  template: {% url 'students:student_detail' s.id %}

STATIC
  {% load static %}
  <link rel="stylesheet" href="{% static 'students/css/style.css' %}">
```

---

## 🔜 Next Lecture

**Models & the Django ORM:** replacing the Python `STUDENTS` list with a real database table, migrations, and the Django Admin panel.
