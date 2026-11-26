# Lecture 06: Class-Based Views & Pagination

> **Platform:** Ubuntu
> **Project:** Continue with `student_management` from Lecture 05

## Learning Outcomes

By the end of this lecture you will be able to:

1. Explain what class-based views (CBVs) are and why Django provides them.
2. Use `TemplateView`, `ListView` and `DetailView` to display data.
3. Use `CreateView`, `UpdateView` and `DeleteView` to change data.
4. Customise generic views with `get_queryset()`, `get_context_data()` and `form_valid()`.
5. Protect class-based views with mixins.
6. Split long lists into pages with pagination.
7. Trace, for any URL, which method of a class-based view runs and what the browser shows.

## How to Use This Lecture

Every step follows the same pattern:

```text
 📖 What it is  → short explanation: what this view does and when to use it
 📝 Code        → what to write
 ▶  Try it      → which URL to open
 🖥️ Output      → what you see in the browser and in the terminal
```

Type the code, open the URL, and check that you see the **same output**. If you don't, stop and fix it before moving on.

---

## 00. Before You Start

### 0.1 Sample data

So that everyone sees the same output, reset the students to this data. Open the shell:

```bash
python manage.py shell
```

```python
from students.models import Student, Department

Student.objects.all().delete()
cs, _ = Department.objects.get_or_create(name='Computer Science', code='CS')
it, _ = Department.objects.get_or_create(name='Information Technology', code='IT')

Student.objects.create(name='Ali Raza',    roll_no='BSCS-01', cgpa=3.65, department=cs)
Student.objects.create(name='Ayesha Khan', roll_no='BSCS-02', cgpa=3.10, department=cs)
Student.objects.create(name='Bilal Ahmed', roll_no='BSCS-03', cgpa=2.40, department=it)
Student.objects.create(name='Sara Malik',  roll_no='BSCS-04', cgpa=3.90, department=cs)
Student.objects.create(name='Usman Khan',  roll_no='BSCS-05', cgpa=1.80, department=it, is_active=False)
Student.objects.create(name='Hina Iqbal',  roll_no='BSCS-06', cgpa=3.25, department=it)

for s in Student.objects.all():
    print(s.id, s.name, s.is_active)
```

Note the **ids** printed. This lecture assumes Ayesha's id is **2** and Hina's id is **6**. If yours differ, use your own ids in the URLs.

| id | Name | Roll No | CGPA | Department | Active |
| :---: | :--- | :--- | :---: | :---: | :---: |
| 1 | Ali Raza | BSCS-01 | 3.65 | CS | ✅ |
| 2 | Ayesha Khan | BSCS-02 | 3.10 | CS | ✅ |
| 3 | Bilal Ahmed | BSCS-03 | 2.40 | IT | ✅ |
| 4 | Sara Malik | BSCS-04 | 3.90 | CS | ✅ |
| 5 | Usman Khan | BSCS-05 | 1.80 | IT | ❌ |
| 6 | Hina Iqbal | BSCS-06 | 3.25 | IT | ✅ |

### 0.2 Users

You need the two users from earlier lectures:

| Username | Type | Created in |
| :--- | :--- | :--- |
| `admin` | Superuser | Lecture 03 (`createsuperuser`) |
| `ali` | Normal user, no permissions | Lecture 05 (sign-up page) |

### 0.3 Keep the terminal visible

Run the server and keep its terminal next to the browser. Every request prints a line like this:

```text
"GET /students/ HTTP/1.1" 200 2154
  │       │                │    └── size of the response in bytes (ignore)
  │       │                └─────── status code: 200 OK, 302 redirect, 403 forbidden, 404 not found
  │       └──────────────────────── URL
  └──────────────────────────────── HTTP method
```

These lines show you the **flow** of each request. We will read them in every step.

### 0.4 The plan

We will **replace the function-based views of Lectures 04–05 one by one** with class-based views. After each replacement, the page must work exactly as before.

---

## 01. Why Class-Based Views?

Our function-based views from Lectures 03–05:

| View | What it does |
| :--- | :--- |
| `student_list` | Query objects → render the list template |
| `student_detail` | Get one object or 404 → render the detail template |
| `student_add` | GET: empty form. POST: validate → save → redirect |
| `student_edit` | Same as add, with `instance=` |
| `student_delete` | GET: confirm page. POST: delete → redirect |

For `Department`, `Course`, `Teacher` and `Fee` we would write the same five views again, changing only the model, form and template names.

**Class-based views** contain these patterns already. You tell Django **which** model, form and template; Django already knows **how** to do the rest.

```python
# Function-based (Lecture 03)
def student_list(request):
    students = Student.objects.all()
    return render(request, 'students/student_list.html', {'students': students})

# Class-based (this lecture)
class StudentListView(ListView):
    model = Student
    context_object_name = 'students'
```

Both produce the same page. Step 3 proves it.

---

## Step 1 — The Simplest CBV: `View`

### 📖 What it is

`View` is the parent class of **every** class-based view. It does only one job: it looks at the HTTP method of the request (`GET`, `POST`, ...) and calls the method of the class with the same name.

| Function-based view | Class-based view |
| :--- | :--- |
| One function for all methods | One **method** per HTTP method: `get()`, `post()`, ... |
| `if request.method == 'POST':` | Not needed; Django chooses `get()` or `post()` |
| Unsupported method → you handle it | Unsupported method → **405** automatically |
| `request` is a parameter | Also available everywhere as `self.request` |

Inside any method of a CBV you can use:

| Attribute | Contains | Example |
| :--- | :--- | :--- |
| `self.request` | The request object | `self.request.user`, `self.request.GET.get('q')` |
| `self.kwargs` | Values captured from the URL | `self.kwargs['pk']` for `/students/2/` → `2` |

**Use it when:** you need full control and no generic view fits. Most of the time you'll use the generic views in the next steps, which are built on top of `View`.

### 📝 Code

```python
# students/views.py  (add at the top with the other imports)
from django.http import HttpResponse
from django.views import View


class HelloView(View):
    def get(self, request):
        return HttpResponse("Hello from a GET request")
```

```python
# students/urls.py  (add this line to urlpatterns)
path('hello/', views.HelloView.as_view(), name='hello'),
```

* Instead of `if request.method == 'GET':`, we write a **method named `get`**. For POST requests we would write a method named `post`.
* `path()` needs a function. **`.as_view()`** turns the class into one.

### ▶ Try it

Open **http://127.0.0.1:8000/students/hello/**

### 🖥️ Output

Browser:

```text
Hello from a GET request
```

Terminal:

```text
"GET /students/hello/ HTTP/1.1" 200 24
```

### ▶ Try it: a method the view doesn't have

Rename the method from `get` to `post`:

```python
class HelloView(View):
    def post(self, request):
        return HttpResponse("Hello from a POST request")
```

Reload **http://127.0.0.1:8000/students/hello/**. The browser always sends `GET` when you open a URL.

### 🖥️ Output

Browser: an empty page.

Terminal:

```text
Method Not Allowed (GET): /students/hello/
"GET /students/hello/ HTTP/1.1" 405 0
```

`HelloView` has no `get()` method, so `dispatch()` answers **405** by itself. We didn't write any code for this. Rename the method back to `get`.

### ▶ Try it: forget `.as_view()`

Change the URL to `path('hello/', views.HelloView, name='hello')` and reload.

### 🖥️ Output

```text
TypeError at /students/hello/
HelloView.__init__() takes 1 positional argument but 2 were given
```

This is the most common CBV mistake. Put `.as_view()` back.

---

## Step 2 — `TemplateView`: A Page With No Model

### 📖 What it is

`TemplateView` shows a template. That's all. It has no model and runs no query.

**Use it for:** pages whose content is mostly fixed, such as *About*, *Contact*, *Help*, or a home page.

| Attribute / method | Purpose |
| :--- | :--- |
| `template_name` | **Required.** Which template to show. |
| `get_context_data()` | Override it to send variables to the template. |

Without `get_context_data()`, two lines are enough:

```python
class AboutView(TemplateView):
    template_name = 'students/about.html'
```

To send data (here, the number of students), override `get_context_data()`:

* Start with `context = super().get_context_data(**kwargs)` to keep what Django already put in the dictionary.
* Add your own keys.
* **Return** the dictionary. If you forget either `super()` or `return`, the template loses data.

### 📝 Code

```python
# students/views.py
from django.views.generic import TemplateView
from .models import Student


class AboutView(TemplateView):
    template_name = 'students/about.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['total_students'] = Student.objects.count()
        context['university'] = 'University of the Punjab'
        return context
```

`students/templates/students/about.html`:

```html
{% extends 'base.html' %}

{% block content %}
    <h1>About</h1>
    <p>Student Management System of {{ university }}.</p>
    <p>Total students in the system: {{ total_students }}</p>
{% endblock %}
```

```python
# students/urls.py  (replace the old about line from Lecture 01)
path('about/', views.AboutView.as_view(), name='about'),
```

### ▶ Try it

Open **http://127.0.0.1:8000/students/about/**

### 🖥️ Output

```text
🌐 http://127.0.0.1:8000/students/about/
─────────────────────────────────────────
About
Student Management System of University of the Punjab.
Total students in the system: 6
```

```text
"GET /students/about/ HTTP/1.1" 200 1032
```

### ▶ Try it: forget the `return`

Delete the line `return context` and reload.

### 🖥️ Output

```text
About
Student Management System of .
Total students in the system:
```

No error, but the values are **missing**. Without `return`, the method returns `None`, so the template receives no data, and missing variables print nothing (Lecture 02). Put `return context` back.

> **Key idea:** In a CBV you don't write the whole view. You **override only the method you want to change**. Here we changed only "what data goes to the template".

---

## Step 3 — `ListView`: The Student List

### 📖 What it is

`ListView` shows a **list of objects** of one model. It replaces views like our `student_list` function.

With only `model = Student`, Django will:

1. run `Student.objects.all()`,
2. render the template **`students/student_list.html`** (`<app>/<model>_list.html`),
3. send the results to the template as **`object_list`** and as **`student_list`**.

| Attribute / method | Purpose |
| :--- | :--- |
| `model` | Which model to list. |
| `context_object_name` | Name of the list in the template (instead of `object_list`). |
| `template_name` | Use a different template than the default. |
| `ordering` | Sort order, e.g. `['-cgpa']`. |
| `paginate_by` | Number of objects per page (Step 10). |
| `get_queryset()` | Override it to filter or search (Step 4). |

**Use it for:** any page that lists records: students, books, courses, orders.

### 📝 Code

Comment out the old `student_list` function in `views.py` and add:

```python
# students/views.py
from django.views.generic import TemplateView, ListView


class StudentListView(ListView):
    model = Student
```

```python
# students/urls.py
path('', views.StudentListView.as_view(), name='student_list'),
```

Your template from Lectures 02–03 (`students/templates/students/student_list.html`) should look like this. Update it if needed:

```html
{% extends 'base.html' %}

{% block title %}Students{% endblock %}

{% block content %}
<h1>Students ({{ students|length }})</h1>

<table>
    <tr><th>#</th><th>Name</th><th>Roll No</th><th>CGPA</th></tr>
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
</table>
{% endblock %}
```

### ▶ Try it

Open **http://127.0.0.1:8000/students/**

### 🖥️ Output

```text
🌐 http://127.0.0.1:8000/students/
─────────────────────────────────────────
Students (0)
 #   Name   Roll No   CGPA
 No students found.
```

**Six students are in the database, but the page shows none.** No error either. Why?

`ListView` sends the list under the names **`object_list`** and **`student_list`**. Our template uses **`students`**. A missing variable in a template prints nothing (Lecture 02), so the loop runs zero times.

### 📝 Fix

```python
class StudentListView(ListView):
    model = Student
    context_object_name = 'students'
```

`context_object_name` tells `ListView` which name to use for the list.

### 🖥️ Output (after the fix)

```text
🌐 http://127.0.0.1:8000/students/
─────────────────────────────────────────
Students (6)
 #   Name          Roll No   CGPA
 1   Ali Raza      BSCS-01   3.65
 2   Ayesha Khan   BSCS-02   3.10
 3   Bilal Ahmed   BSCS-03   2.40
 4   Sara Malik    BSCS-04   3.90
 5   Usman Khan    BSCS-05   1.80
 6   Hina Iqbal    BSCS-06   3.25
```

```text
"GET /students/ HTTP/1.1" 200 3410
```

Sorted by roll number because of `Meta.ordering = ['roll_no']` (Lecture 03).

### How did `ListView` know which template to use?

We never wrote `template_name`. Generic views build it from the app and model name:

| Generic view | Default template | Context variables |
| :--- | :--- | :--- |
| `ListView` | `students/student_list.html` | `object_list`, `student_list` |
| `DetailView` | `students/student_detail.html` | `object`, `student` |
| `CreateView` | `students/student_form.html` | `form` |
| `UpdateView` | `students/student_form.html` | `form`, `object`, `student` |
| `DeleteView` | `students/student_confirm_delete.html` | `object`, `student` |

These are exactly the template names we have used since Lecture 02. That's why our old templates keep working.

> ▶ **Try it yourself:** Rename `student_list.html` to `students.html` and reload. You get `TemplateDoesNotExist: students/student_list.html`. Rename it back.

---

## Step 4 — `get_queryset()` and `get_context_data()`: Search

These two methods are the ones you'll override most. They do **different jobs**:

| Method | Answers the question | Runs |
| :--- | :--- | :--- |
| `get_queryset()` | **Which records** should the page show? | First |
| `get_context_data()` | **What extra variables** does the template need? | Second, and receives the records from `get_queryset()` |

### 4.1 Show only active students

### 📝 Code

```python
class StudentListView(ListView):
    model = Student
    context_object_name = 'students'

    def get_queryset(self):
        return Student.objects.filter(is_active=True)
```

### ▶ Try it

Open **http://127.0.0.1:8000/students/**

### 🖥️ Output

```text
Students (5)
 #   Name          Roll No   CGPA
 1   Ali Raza      BSCS-01   3.65
 2   Ayesha Khan   BSCS-02   3.10
 3   Bilal Ahmed   BSCS-03   2.40
 4   Sara Malik    BSCS-04   3.90
 5   Hina Iqbal    BSCS-06   3.25
```

**Usman Khan** (inactive) is gone.

### 4.2 Add search

### 📝 Code

```python
class StudentListView(ListView):
    model = Student
    context_object_name = 'students'

    def get_queryset(self):
        queryset = Student.objects.filter(is_active=True)

        query = self.request.GET.get('q')
        if query:
            queryset = queryset.filter(name__icontains=query)

        return queryset
```

* In a CBV the request is **`self.request`** (not a parameter as in function views).

Add a search form above the table in `student_list.html`:

```html
<form method="get">
    <input type="text" name="q" placeholder="Search by name">
    <button type="submit">Search</button>
</form>
```

### ▶ Try it

Type **`khan`** in the box and press **Search**. The URL becomes **http://127.0.0.1:8000/students/?q=khan**

### 🖥️ Output

```text
🌐 http://127.0.0.1:8000/students/?q=khan
─────────────────────────────────────────
[                 ] [Search]          ← the box is EMPTY again
Students (1)
 #   Name          Roll No   CGPA
 1   Ayesha Khan   BSCS-02   3.10
```

```text
"GET /students/?q=khan HTTP/1.1" 200 2287
```

The filtering works, but there's a problem: **the search box is empty**, and the page doesn't say what was searched. The template doesn't know the word `khan`. It only received the **students**.

### 4.3 Send the search word to the template: `get_context_data()`

### 📝 Code

```python
class StudentListView(ListView):
    model = Student
    context_object_name = 'students'

    def get_queryset(self):
        queryset = Student.objects.filter(is_active=True)
        query = self.request.GET.get('q')
        if query:
            queryset = queryset.filter(name__icontains=query)
        return queryset

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)    # contains 'students' already
        context['query'] = self.request.GET.get('q', '')
        return context
```

Update the template:

```html
<form method="get">
    <input type="text" name="q" value="{{ query }}" placeholder="Search by name">
    <button type="submit">Search</button>
</form>

{% if query %}
    <p>{{ students|length }} result(s) for "{{ query }}" — <a href="{% url 'students:student_list' %}">Clear</a></p>
{% endif %}
```

### ▶ Try it

Open **http://127.0.0.1:8000/students/?q=khan** again.

### 🖥️ Output

```text
🌐 http://127.0.0.1:8000/students/?q=khan
─────────────────────────────────────────
[ khan            ] [Search]          ← keeps the search word
1 result(s) for "khan" — Clear
Students (1)
 #   Name          Roll No   CGPA
 1   Ayesha Khan   BSCS-02   3.10
```

Now try **http://127.0.0.1:8000/students/** (no `q`): `query` is `''`, so the "results" line is hidden and all 5 active students are shown.

> **Why not filter inside `get_context_data()` too?** You could write `context['students'] = Student.objects.filter(...)` there, but pagination (Step 10) and the detail/edit/delete views all use `get_queryset()`. Filtering anywhere else is skipped by them. **Rule: filter in `get_queryset()`, add extra variables in `get_context_data()`.**

---

## Step 5 — `DetailView`: One Student

### 📖 What it is

`DetailView` shows **one object**. It replaces our `student_detail` function.

It reads the object's primary key from the URL, fetches that object, and shows **404** if it doesn't exist. In Lecture 03 we wrote `get_object_or_404()` ourselves; `DetailView` does it in its **`get_object()`** method.

| Attribute / method | Purpose |
| :--- | :--- |
| `model` | Which model. |
| `pk_url_kwarg` | Name of the URL parameter, if it isn't `pk`. |
| `context_object_name` | Name of the object in the template (default: model name in lowercase, `student`). |
| `get_object()` | Override it to change how the object is found. |

Default template: **`students/student_detail.html`**.

**Use it for:** a page about one record: a student's profile, one book, one order.

### 📝 Code

```python
from django.views.generic import TemplateView, ListView, DetailView


class StudentDetailView(DetailView):
    model = Student
```

```python
# students/urls.py  (replace the old detail line)
path('<int:pk>/', views.StudentDetailView.as_view(), name='student_detail'),
```

> We changed `<int:student_id>` to **`<int:pk>`**. `DetailView` looks for a URL value named **`pk`** (primary key). The URL **name** `student_detail` is unchanged, so `{% url 'students:student_detail' s.id %}` in the list still works.

Your `student_detail.html` from earlier lectures:

```html
{% extends 'base.html' %}

{% block title %}{{ student.name }}{% endblock %}

{% block content %}
    <h1>{{ student.name }}</h1>
    <p><strong>Roll No:</strong> {{ student.roll_no }}</p>
    <p><strong>CGPA:</strong> {{ student.cgpa }}</p>
    <p><strong>Department:</strong> {{ student.department.name|default:"Not assigned" }}</p>
    <p><strong>Added by:</strong> {{ student.created_by.username|default:"Unknown" }}</p>
    <a href="{% url 'students:student_list' %}">← Back to list</a>
{% endblock %}
```

### ▶ Try it

On the list page, click **Ayesha Khan**. The URL is **http://127.0.0.1:8000/students/2/**

### 🖥️ Output

```text
🌐 http://127.0.0.1:8000/students/2/
─────────────────────────────────────────
Ayesha Khan
Roll No: BSCS-02
CGPA: 3.10
Department: Computer Science
Added by: Unknown
← Back to list
```

```text
"GET /students/2/ HTTP/1.1" 200 1544
```

### ▶ Try it: a student that doesn't exist

Open **http://127.0.0.1:8000/students/99/**

```text
Page not found (404)
No student found matching the query
```

```text
"GET /students/99/ HTTP/1.1" 404 ...
```

`get_object()` did `get_object_or_404()` for us. In Lecture 03 we wrote that line ourselves.

### ▶ Try it: wrong parameter name

Change the URL back to `'<int:student_id>/'` and reload `/students/2/`.

```text
AttributeError at /students/2/
Generic detail view StudentDetailView must be called with either an object pk or a slug in the URLconf.
```

`get_object()` looked for `self.kwargs['pk']` and found only `student_id`. Two ways to fix it: use `<int:pk>` in the URL (what we do), **or** keep `student_id` and add `pk_url_kwarg = 'student_id'` to the view.

---

## Step 6 — `CreateView`: Add a Student

### 📖 What it is

`CreateView` shows a form for a **new** object and saves it. It contains the whole form pattern from Lecture 04: GET shows an empty form; POST validates, then saves and redirects, or shows the form again with errors.

| Attribute / method | Purpose |
| :--- | :--- |
| `model` | Which model to create. |
| `form_class` | Your `ModelForm` (keeps validation such as `clean_cgpa`). |
| `fields = [...]` | Instead of `form_class`: let Django build a simple form from these fields. Use **one** of the two. |
| `success_url` | Where to redirect after saving. |
| `form_valid()` | Override it to do something extra when the form is valid (Step 9.2). |

Default template: **`students/student_form.html`**, with the form available as `form`.

**Use it for:** any "Add new ..." page.

### 📝 Code

```python
from django.urls import reverse_lazy
from django.views.generic import TemplateView, ListView, DetailView, CreateView
from .forms import StudentForm


class StudentCreateView(CreateView):
    model = Student
    form_class = StudentForm                              # our form from Lecture 04
    success_url = reverse_lazy('students:student_list')   # where to go after saving
```

```python
# students/urls.py  (replace the old add line)
path('add/', views.StudentCreateView.as_view(), name='student_add'),
```

Your `student_form.html` from Lecture 04:

```html
{% extends 'base.html' %}

{% block content %}
<h1>{% if student %}Edit {{ student.name }}{% else %}Add Student{% endif %}</h1>

<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Save</button>
    <a href="{% url 'students:student_list' %}">Cancel</a>
</form>
{% endblock %}
```

> **`reverse_lazy` instead of `reverse`:** `success_url` is a class attribute. Python reads it when `views.py` is first loaded, **before** Django has loaded the URLs, so `reverse()` would fail. `reverse_lazy()` waits until the value is used. Rule: **`reverse_lazy()` in class attributes, `reverse()` / `redirect()` inside methods.**

> For now this page is **not protected** (anyone can open it). We add login protection in Step 9.

A `CreateView` handles **three** different situations. Try each one.

### ▶ Try it (A): open the form

Open **http://127.0.0.1:8000/students/add/**

### 🖥️ Output (A)

```text
🌐 http://127.0.0.1:8000/students/add/
─────────────────────────────────────────
Add Student
Name:        [               ]
Roll Number: [               ]
Email:       [               ]
CGPA:        [               ]
Department:  [---------     v]
Currently enrolled: [✓]
[Save]  Cancel
```

```text
"GET /students/add/ HTTP/1.1" 200 4120
```

### ▶ Try it (B): submit invalid data

Fill in: Name `Test`, Roll Number `BSCS-50`, CGPA **`5`**. Click **Save**.

### 🖥️ Output (B)

```text
Add Student
Name:        [Test           ]       ← your data is kept
Roll Number: [BSCS-50        ]
• CGPA must be between 0.00 and 4.00.
CGPA:        [5              ]
...
```

```text
"POST /students/add/ HTTP/1.1" 200 4380
```

Status **200**, not 302: the page was shown again, nothing was saved.

### ▶ Try it (C): submit valid data

Change CGPA to **`3.40`** and set the name to **`Zara Ahmed`**, roll number **`BSCS-07`**, department **Computer Science**. Click **Save**.

### 🖥️ Output (C)

Terminal — **two** lines for one click:

```text
"POST /students/add/ HTTP/1.1" 302 0          ← saved, "go to /students/"
"GET /students/ HTTP/1.1" 200 3620            ← browser follows the redirect
```

Browser:

```text
🌐 http://127.0.0.1:8000/students/
─────────────────────────────────────────
Students (6)
 #   Name          Roll No   CGPA
 1   Ali Raza      BSCS-01   3.65
 2   Ayesha Khan   BSCS-02   3.10
 3   Bilal Ahmed   BSCS-03   2.40
 4   Sara Malik    BSCS-04   3.90
 5   Hina Iqbal    BSCS-06   3.25
 6   Zara Ahmed    BSCS-07   3.40
```

This is the **Post/Redirect/Get** pattern from Lecture 04, done for us.

### Where did our Lecture 04 code go?

| Lecture 04 function-based view | Done by `CreateView` in |
| :--- | :--- |
| `if request.method == 'POST':` | `dispatch()` → `get()` or `post()` |
| `form = StudentForm()` / `StudentForm(request.POST)` | `get_form()` |
| `if form.is_valid():` | `post()` |
| `form.save()` + `return redirect(...)` | `form_valid()` |
| `return render(request, ..., {'form': form})` | `get()` / `form_invalid()` |

You wrote all of this by hand in Lecture 04. `CreateView` contains exactly the same logic.

---

## Step 7 — `UpdateView`: Edit a Student

### 📖 What it is

`UpdateView` is `CreateView` plus `DetailView`: it finds an **existing** object by `pk` (like `DetailView`), shows a form **pre-filled** with its data, and saves the changes.

It passes the object to the form as `instance=` itself, so saving always **updates** that row.

| Attribute | Purpose |
| :--- | :--- |
| `model`, `form_class` / `fields` | Same as `CreateView`. |
| `success_url` | Where to go after saving. If you leave it out, Django uses the model's **`get_absolute_url()`**. |

Default template: **`students/student_form.html`** (the same as `CreateView`), with `form` and `student` in the context.

**`get_absolute_url()`** is a model method that returns the URL of the object's own page. Write it once in the model, and then `UpdateView` and `CreateView` can redirect to it, and templates can use `{{ student.get_absolute_url }}`.

**Use it for:** any "Edit ..." page.

### 📝 Code

First, give the model a "home URL", so Django knows where a student's page is:

```python
# students/models.py
from django.urls import reverse


class Student(models.Model):
    ...

    def get_absolute_url(self):
        return reverse('students:student_detail', args=[self.pk])
```

```python
# students/views.py
from django.views.generic import (
    TemplateView, ListView, DetailView, CreateView, UpdateView,
)


class StudentUpdateView(UpdateView):
    model = Student
    form_class = StudentForm
    # no success_url → Django redirects to student.get_absolute_url()
```

```python
# students/urls.py  (replace the old edit line)
path('<int:pk>/edit/', views.StudentUpdateView.as_view(), name='student_edit'),
```

No new template: `UpdateView` uses the same `student_form.html`.

### ▶ Try it (A): open the edit form

Open **http://127.0.0.1:8000/students/2/edit/**

### 🖥️ Output (A)

```text
🌐 http://127.0.0.1:8000/students/2/edit/
─────────────────────────────────────────
Edit Ayesha Khan                       ← {% if student %} is true now
Name:        [Ayesha Khan    ]
Roll Number: [BSCS-02        ]
CGPA:        [3.10           ]
Department:  [Computer Science v]
...
```

### ▶ Try it (B): save a change

Change CGPA to **`3.20`** and click **Save**.

### 🖥️ Output (B)

```text
"POST /students/2/edit/ HTTP/1.1" 302 0
"GET /students/2/ HTTP/1.1" 200 1544
```

```text
🌐 http://127.0.0.1:8000/students/2/
─────────────────────────────────────────
Ayesha Khan
Roll No: BSCS-02
CGPA: 3.20                ← changed
...
```

> Remember the Lecture 04 bug where forgetting `instance=` made every "edit" create a new student? `UpdateView` always passes `instance=`, so that bug can't happen.

---

## Step 8 — `DeleteView`: Delete a Student

### 📖 What it is

`DeleteView` deletes one object, with a confirmation step:

* **GET** shows a confirmation page. Nothing is deleted.
* **POST** deletes the object and redirects.

This is the same safe pattern we built by hand in Lecture 04.

| Attribute | Purpose |
| :--- | :--- |
| `model` | Which model. |
| `success_url` | **Required.** Where to go after deleting (the object's own page no longer exists). |

Default template: **`students/student_confirm_delete.html`**, with `student` in the context.

**Use it for:** any "Delete ..." action.

### 📝 Code

```python
from django.views.generic import (
    TemplateView, ListView, DetailView, CreateView, UpdateView, DeleteView,
)


class StudentDeleteView(DeleteView):
    model = Student
    success_url = reverse_lazy('students:student_list')
```

```python
# students/urls.py  (replace the old delete line)
path('<int:pk>/delete/', views.StudentDeleteView.as_view(), name='student_delete'),
```

Your `student_confirm_delete.html` from Lecture 04 works unchanged.

### ▶ Try it (A): open the delete page

Open **http://127.0.0.1:8000/students/6/delete/**

### 🖥️ Output (A)

```text
🌐 http://127.0.0.1:8000/students/6/delete/
─────────────────────────────────────────
Delete Student
Are you sure you want to delete Hina Iqbal (BSCS-06)?
[Yes, delete]  Cancel
```

Click **Cancel** first: you go back, and Hina is still in the list.

### ▶ Try it (B): confirm

Open the delete page again and click **Yes, delete**.

### 🖥️ Output (B)

```text
"POST /students/6/delete/ HTTP/1.1" 302 0
"GET /students/ HTTP/1.1" 200 3301
```

```text
Students (5)
 #   Name          Roll No   CGPA
 1   Ali Raza      BSCS-01   3.65
 2   Ayesha Khan   BSCS-02   3.20
 3   Bilal Ahmed   BSCS-03   2.40
 4   Sara Malik    BSCS-04   3.90
 5   Zara Ahmed    BSCS-07   3.40
```

**Hina Iqbal** is gone. Opening **/students/6/** now gives **404**.

### Summary so far: which method runs when?

| View | GET request | POST request |
| :--- | :--- | :--- |
| `ListView` | `get_queryset()` → `get_context_data()` → render | — |
| `DetailView` | `get_object()` → `get_context_data()` → render | — |
| `CreateView` | `get_form()` (empty) → render | `get_form()` → `is_valid()` → `form_valid()` (save + redirect) **or** `form_invalid()` (render with errors) |
| `UpdateView` | `get_object()` → `get_form()` (pre-filled) → render | `get_object()` → `get_form()` → `is_valid()` → `form_valid()` **or** `form_invalid()` |
| `DeleteView` | `get_object()` → render confirm page | `get_object()` → `form_valid()` (delete + redirect) |

To change one part of the behaviour, **override that one method**. Steps 4 and 9 show how.

### Methods you will override most

| Method | Override it to ... | Available in |
| :--- | :--- | :--- |
| `get_queryset()` | Filter or sort the objects | List, Detail, Update, Delete |
| `get_context_data()` | Add extra variables to the template | All |
| `get_object()` | Change how the single object is found | Detail, Update, Delete |
| `form_valid(form)` | Do something extra when the form is valid | Create, Update, Delete |
| `get_success_url()` | Decide the redirect address in code | Create, Update, Delete |

---

## Step 9 — Protecting Views and Doing Extra Work

Right now, anyone (even without logging in) can add, edit and delete. In Lecture 05 we used decorators such as `@login_required`. Decorators are for functions; classes use **mixins**.

A **mixin** is a small class that adds one feature when you inherit from it.

### 9.1 `LoginRequiredMixin`

**What it does:** if the user isn't logged in, it redirects to the login page (with `?next=`), exactly like `@login_required` from Lecture 05. If the user is logged in, the view runs normally.

### 📝 Code

```python
from django.contrib.auth.mixins import LoginRequiredMixin


class StudentCreateView(LoginRequiredMixin, CreateView):
    model = Student
    form_class = StudentForm
    success_url = reverse_lazy('students:student_list')
```

### ▶ Try it

**Log out**, then open **http://127.0.0.1:8000/students/add/**

### 🖥️ Output

```text
"GET /students/add/ HTTP/1.1" 302 0
"GET /accounts/login/?next=/students/add/ HTTP/1.1" 200 2950
         ... you type username and password ...
"POST /accounts/login/ HTTP/1.1" 302 0
"GET /students/add/ HTTP/1.1" 200 4120
```

```text
🌐 http://127.0.0.1:8000/accounts/login/?next=/students/add/
─────────────────────────────────────────
Login
Username: [          ]
Password: [          ]
[Login]
```

After logging in, you land on the **Add Student** form.

### ▶ Try it: wrong order

Change the class line to `class StudentCreateView(CreateView, LoginRequiredMixin):`, log out and open `/students/add/` again.

```text
"GET /students/add/ HTTP/1.1" 200 4120     ← the form opens WITHOUT login!
```

Python looks for `dispatch()` **from left to right**. `CreateView`'s own `dispatch()` is found first, so the login check never runs. **Mixins always go on the left.** Fix the order.

### 9.2 `form_valid()`: set `created_by` and show a message

**What it does:** `form_valid(form)` is the method that runs when the submitted form is valid. Normally it saves the object and redirects. We override it to **add one step before saving**, and then call the original with `super()`.

**`SuccessMessageMixin`** adds a success message (Lecture 04's messages framework) after a successful save. You only write the text in `success_message`.

In Lecture 05 we set `created_by` with `form.save(commit=False)`. In a CBV, we override **`form_valid()`**, the method that runs only when the form is valid (Step 6, flow C).

### 📝 Code

```python
from django.contrib.messages.views import SuccessMessageMixin


class StudentCreateView(LoginRequiredMixin, SuccessMessageMixin, CreateView):
    model = Student
    form_class = StudentForm
    success_url = reverse_lazy('students:student_list')
    success_message = "Student %(name)s added successfully."

    def form_valid(self, form):
        form.instance.created_by = self.request.user    # OUR extra step
        return super().form_valid(form)                 # normal step: save + redirect
```

* **`form.instance`** is the student object the form is about to save.
* **`super().form_valid(form)`** runs `CreateView`'s normal `form_valid()`: save and redirect. Always call it **and return** it.
* **`%(name)s`** in `success_message` is replaced by the value of the form's `name` field.

### ▶ Try it

Logged in as **ali**, add a student: Name **`Omar Farooq`**, Roll Number **`BSCS-08`**, CGPA **`2.90`**, Department **IT**. Click **Save**.

### 🖥️ Output

```text
🌐 http://127.0.0.1:8000/students/
─────────────────────────────────────────
┌───────────────────────────────────────────┐
│ Student Omar Farooq added successfully.   │   ← green message
└───────────────────────────────────────────┘
Students (6)
 ...
 6   Omar Farooq   BSCS-08   2.90
```

Click **Omar Farooq**:

```text
Omar Farooq
...
Added by: ali
```

Reload the list page: the green message **disappears**. Messages are shown only once.

### 9.3 `UserPassesTestMixin`: only the owner (or staff) may edit

**What it does:** before the view runs, it calls your method **`test_func()`**. If it returns `True`, the view runs; if `False`, the user gets **403 Forbidden**. Use it for rules that depend on the specific record, such as "only the person who added this student".

### 📝 Code

```python
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin


class StudentUpdateView(LoginRequiredMixin, UserPassesTestMixin, SuccessMessageMixin, UpdateView):
    model = Student
    form_class = StudentForm
    success_message = "Student %(name)s updated."

    def test_func(self):
        student = self.get_object()
        return student.created_by == self.request.user or self.request.user.is_staff
```

`test_func()` must return `True` (allowed) or `False` (forbidden).

### ▶ Try it

Logged in as **ali**:

| URL | Student | `created_by` | Result |
| :--- | :--- | :--- | :--- |
| `/students/2/edit/` | Ayesha Khan | `None` (shell) | **403** |
| `/students/<Omar's id>/edit/` | Omar Farooq | `ali` | Edit form opens |

### 🖥️ Output

```text
403 Forbidden
```

```text
"GET /students/2/edit/ HTTP/1.1" 403 135
```

Ayesha was added in the shell, so her `created_by` is empty, and ali is not staff: `test_func()` returns `False`, and `UpdateView` never runs.

Log in as **admin** and open `/students/2/edit/`: `is_staff` is `True`, so the form opens.

### 9.4 `PermissionRequiredMixin`: only users with the delete permission

**What it does:** checks that the user has the permission named in `permission_required` (Lecture 05: `<app>.<action>_<model>`). Without it, the user gets **403 Forbidden**. Superusers have every permission.

### 📝 Code

```python
from django.contrib.auth.mixins import (
    LoginRequiredMixin, PermissionRequiredMixin, UserPassesTestMixin,
)


class StudentDeleteView(LoginRequiredMixin, PermissionRequiredMixin, SuccessMessageMixin, DeleteView):
    model = Student
    success_url = reverse_lazy('students:student_list')
    permission_required = 'students.delete_student'
    success_message = "Student deleted."
```

### ▶ Try it

| Logged in as | Open | Result |
| :--- | :--- | :--- |
| ali (no permissions) | `/students/<Omar's id>/delete/` | **403 Forbidden**, even though ali created Omar |
| admin (superuser) | `/students/<Omar's id>/delete/` | Confirmation page |

Then give ali the permission in the admin panel (**Users → ali → User permissions → `students | student | Can delete student`**), and try again as ali: the confirmation page opens. Click **Cancel**: we need Omar in Step 10.

### Decorators vs mixins

| Function-based view (Lecture 05) | Class-based view |
| :--- | :--- |
| `@login_required` | `LoginRequiredMixin` |
| `@permission_required('students.delete_student', raise_exception=True)` | `PermissionRequiredMixin` + `permission_required = '...'` |
| `if ...: raise PermissionDenied` | `UserPassesTestMixin` + `test_func()` |
| `messages.success(request, '...')` | `SuccessMessageMixin` + `success_message = '...'` |

---

## Step 10 — Pagination

### 📖 What it is

Django's **`Paginator`** takes a list of objects and a page size, and gives you **one page** at a time. You can try it in the shell:

```python
>>> from django.core.paginator import Paginator
>>> from students.models import Student

>>> paginator = Paginator(Student.objects.all(), 2)     # 2 students per page
>>> paginator.count            # total number of students
>>> paginator.num_pages        # total number of pages

>>> page = paginator.page(1)
>>> page.object_list           # the 2 students on page 1
>>> page.has_next()
True
>>> page.next_page_number()
2
```

In a `ListView` you don't use `Paginator` yourself. You write **`paginate_by = 2`** and `ListView`:

1. reads the page number from `?page=` in the URL,
2. puts only that page's objects in the template (as `students` / `object_list`),
3. adds `page_obj`, `paginator` and `is_paginated` to the context for the page links.

With 6 students one page is fine. With 5,000, the page is slow and unusable. **Pagination** splits the list into pages: `?page=1`, `?page=2`, ...

We'll use **2 students per page**, so we can see several pages with our small data. In real projects use 10–25.

### 10.1 Turn it on

### 📝 Code

```python
class StudentListView(ListView):
    model = Student
    context_object_name = 'students'
    paginate_by = 2                         # ← the only new line

    def get_queryset(self):
        ...                                  # unchanged from Step 4

    def get_context_data(self, **kwargs):
        ...                                  # unchanged from Step 4
```

### ▶ Try it

Open **http://127.0.0.1:8000/students/**

### 🖥️ Output

```text
🌐 http://127.0.0.1:8000/students/
─────────────────────────────────────────
Students (2)
 #   Name          Roll No   CGPA
 1   Ali Raza      BSCS-01   3.65
 2   Ayesha Khan   BSCS-02   3.20
```

Only 2 students, and no way to reach the others yet. Also note **"Students (2)"**: `students|length` now counts only the current page.

### ▶ Try it: change the page by hand

| URL | Shows |
| :--- | :--- |
| `/students/?page=1` | Ali Raza, Ayesha Khan |
| `/students/?page=2` | Bilal Ahmed, Sara Malik |
| `/students/?page=3` | Zara Ahmed (BSCS-07), Omar Farooq (BSCS-08) |
| `/students/?page=99` | **404**: `Invalid page (99): That page contains no results` |
| `/students/?page=abc` | **404** |
| `/students/?page=last` | Page 3 |

The order follows the roll numbers (`Meta.ordering`), so Zara (BSCS-07) comes before Omar (BSCS-08).

### 10.2 Page links

Create `templates/partials/pagination.html`:

```html
{% if is_paginated %}
<div class="pagination">
    {% if page_obj.has_previous %}
        <a href="?page=1">&laquo; First</a>
        <a href="?page={{ page_obj.previous_page_number }}">Previous</a>
    {% endif %}

    <span>Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}</span>

    {% if page_obj.has_next %}
        <a href="?page={{ page_obj.next_page_number }}">Next</a>
        <a href="?page={{ page_obj.paginator.num_pages }}">Last &raquo;</a>
    {% endif %}
</div>
{% endif %}
```

In `student_list.html`, change the heading to use the **total** count and include the links below the table:

```html
<h1>Students ({{ page_obj.paginator.count }})</h1>
<p>Showing {{ page_obj.start_index }}–{{ page_obj.end_index }} of {{ page_obj.paginator.count }}</p>

<table> ... </table>

{% include 'partials/pagination.html' %}
```

### 🖥️ Output: page 1

```text
🌐 http://127.0.0.1:8000/students/
─────────────────────────────────────────
Students (6)
Showing 1–2 of 6
 #   Name          Roll No   CGPA
 1   Ali Raza      BSCS-01   3.65
 2   Ayesha Khan   BSCS-02   3.20

Page 1 of 3   Next   Last »
```

Click **Next**:

```text
🌐 http://127.0.0.1:8000/students/?page=2
─────────────────────────────────────────
Students (6)
Showing 3–4 of 6
 #   Name          Roll No   CGPA
 1   Bilal Ahmed   BSCS-03   2.40
 2   Sara Malik    BSCS-04   3.90

« First   Previous   Page 2 of 3   Next   Last »
```

The `#` column restarts at 1 on every page because `forloop.counter` counts only the current page. To number continuously, use `{{ page_obj.start_index|add:forloop.counter0 }}`.

| Template value | Page 2 value |
| :--- | :--- |
| `page_obj.number` | 2 |
| `page_obj.paginator.num_pages` | 3 |
| `page_obj.paginator.count` | 6 |
| `page_obj.has_previous` / `has_next` | True / True |
| `page_obj.previous_page_number` / `next_page_number` | 1 / 3 |
| `page_obj.start_index` / `end_index` | 3 / 4 |

### 10.3 Pagination together with search: a bug to fix

### ▶ Try it

Search for **`r`**. The URL is **/students/?q=r**

### 🖥️ Output

```text
🌐 http://127.0.0.1:8000/students/?q=r
─────────────────────────────────────────
[ r          ] [Search]
2 result(s) for "r"        ← wrong: students|length counts only this page (fixed below)
Students (4)
 1   Ali Raza      BSCS-01   3.65
 2   Sara Malik    BSCS-04   3.90

Page 1 of 2   Next   Last »
```

Now click **Next**:

```text
🌐 http://127.0.0.1:8000/students/?page=2          ← q=r has disappeared!
─────────────────────────────────────────
[            ] [Search]
Students (6)
 1   Bilal Ahmed   BSCS-03   2.40               ← doesn't contain 'r'
 2   Sara Malik    BSCS-04   3.90

Page 2 of 3
```

The link was `?page=2`, which **drops `q=r`**. So `get_queryset()` received no search word and returned all students.

### 📝 Fix

Every page link must carry the search word as well. In `pagination.html`, add `{% if query %}&q={{ query|urlencode }}{% endif %}` to each link:

```html
{% if is_paginated %}
<div class="pagination">
    {% if page_obj.has_previous %}
        <a href="?page=1{% if query %}&q={{ query|urlencode }}{% endif %}">&laquo; First</a>
        <a href="?page={{ page_obj.previous_page_number }}{% if query %}&q={{ query|urlencode }}{% endif %}">Previous</a>
    {% endif %}

    <span>Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}</span>

    {% if page_obj.has_next %}
        <a href="?page={{ page_obj.next_page_number }}{% if query %}&q={{ query|urlencode }}{% endif %}">Next</a>
        <a href="?page={{ page_obj.paginator.num_pages }}{% if query %}&q={{ query|urlencode }}{% endif %}">Last &raquo;</a>
    {% endif %}
</div>
{% endif %}
```

And in `student_list.html`, use the total count in the results line:

```html
<p>{{ page_obj.paginator.count }} result(s) for "{{ query }}"</p>
```

This works because `query` was added in `get_context_data()` (Step 4.3). Here is a second reason we needed it.

### 🖥️ Output: after the fix

Search **`r`** again and click **Next**:

```text
🌐 http://127.0.0.1:8000/students/?page=2&q=r
─────────────────────────────────────────
[ r          ] [Search]
4 result(s) for "r"
 1   Zara Ahmed    BSCS-07   3.40
 2   Omar Farooq   BSCS-08   2.90

« First   Previous   Page 2 of 2
```

Both students on page 2 contain `r`, and the search box keeps `r`.

### 10.4 Pagination in a function-based view (for reference)

If you write a list page as a function, use `Paginator` yourself:

```python
from django.core.paginator import Paginator


def student_list(request):
    students = Student.objects.filter(is_active=True)
    paginator = Paginator(students, 2)
    page_obj = paginator.get_page(request.GET.get('page'))
    return render(request, 'students/student_list.html', {'page_obj': page_obj, 'students': page_obj})
```

One difference: **`get_page()`** never gives 404. `?page=99` shows the last page and `?page=abc` shows the first page. `ListView` gives 404 for those.

> Always paginate an **ordered** queryset. Our `Student` model has `Meta.ordering`, so this is already done. Without an order, Django shows `UnorderedObjectListWarning` and a student may appear on two pages.

---

## 11. The Complete Code

### `students/views.py`

```python
from django.contrib.auth.mixins import (
    LoginRequiredMixin, PermissionRequiredMixin, UserPassesTestMixin,
)
from django.contrib.messages.views import SuccessMessageMixin
from django.urls import reverse_lazy
from django.views.generic import (
    TemplateView, ListView, DetailView, CreateView, UpdateView, DeleteView,
)

from .forms import StudentForm
from .models import Student


class AboutView(TemplateView):
    template_name = 'students/about.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['total_students'] = Student.objects.count()
        context['university'] = 'University of the Punjab'
        return context


class StudentListView(ListView):
    model = Student
    context_object_name = 'students'
    paginate_by = 10

    def get_queryset(self):
        queryset = Student.objects.filter(is_active=True)
        query = self.request.GET.get('q')
        if query:
            queryset = queryset.filter(name__icontains=query)
        return queryset

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['query'] = self.request.GET.get('q', '')
        return context


class StudentDetailView(DetailView):
    model = Student


class StudentCreateView(LoginRequiredMixin, SuccessMessageMixin, CreateView):
    model = Student
    form_class = StudentForm
    success_url = reverse_lazy('students:student_list')
    success_message = "Student %(name)s added successfully."

    def form_valid(self, form):
        form.instance.created_by = self.request.user
        return super().form_valid(form)


class StudentUpdateView(LoginRequiredMixin, UserPassesTestMixin, SuccessMessageMixin, UpdateView):
    model = Student
    form_class = StudentForm
    success_message = "Student %(name)s updated."

    def test_func(self):
        student = self.get_object()
        return student.created_by == self.request.user or self.request.user.is_staff


class StudentDeleteView(LoginRequiredMixin, PermissionRequiredMixin, SuccessMessageMixin, DeleteView):
    model = Student
    success_url = reverse_lazy('students:student_list')
    permission_required = 'students.delete_student'
    success_message = "Student deleted."
```

### `students/urls.py`

```python
from django.urls import path
from . import views

app_name = 'students'

urlpatterns = [
    path('', views.StudentListView.as_view(), name='student_list'),
    path('about/', views.AboutView.as_view(), name='about'),
    path('add/', views.StudentCreateView.as_view(), name='student_add'),
    path('<int:pk>/', views.StudentDetailView.as_view(), name='student_detail'),
    path('<int:pk>/edit/', views.StudentUpdateView.as_view(), name='student_edit'),
    path('<int:pk>/delete/', views.StudentDeleteView.as_view(), name='student_delete'),
]
```

### Every URL at a glance

| You open | View | Method(s) that run | Result |
| :--- | :--- | :--- | :--- |
| `GET /students/` | `StudentListView` | `get_queryset` → paginate → `get_context_data` | Page 1 of active students |
| `GET /students/?q=r&page=2` | `StudentListView` | same, with `q` and `page` | Page 2 of matching students |
| `GET /students/about/` | `AboutView` | `get_context_data` | About page with total |
| `GET /students/2/` | `StudentDetailView` | `get_object` → `get_context_data` | Ayesha's details, or 404 |
| `GET /students/add/` | `StudentCreateView` | login check → `get_form` | Empty form, or redirect to login |
| `POST /students/add/` | `StudentCreateView` | login check → `is_valid` → `form_valid` / `form_invalid` | 302 to list + message, or form with errors |
| `GET /students/2/edit/` | `StudentUpdateView` | login → `test_func` → `get_object` → `get_form` | Pre-filled form, or 403 |
| `POST /students/2/edit/` | `StudentUpdateView` | login → `test_func` → `form_valid` | 302 to `/students/2/` |
| `GET /students/2/delete/` | `StudentDeleteView` | login → permission → `get_object` | Confirmation page, or 403 |
| `POST /students/2/delete/` | `StudentDeleteView` | login → permission → `form_valid` | Deleted, 302 to list |

---

## 12. FBV or CBV?

| Choose a **function-based view** when ... | Choose a **class-based view** when ... |
| :--- | :--- |
| The logic is unusual or doesn't fit list / detail / create / update / delete | The page is a standard list, detail or form page |
| You want every step visible in one place | The same pattern repeats for many models |
| You handle several forms on one page | You want to reuse behaviour through mixins |

> The difficulty with CBVs is that most of the code that runs is **inside Django**, not in your file. When you don't know which method to override, look up the view on **https://ccbv.co.uk/** (Classy Class-Based Views). It lists every attribute and method of every generic view.

---

## 🚨 Troubleshooting

| Error / Symptom | Cause | Fix |
| :--- | :--- | :--- |
| `TypeError: ...__init__() takes 1 positional argument but 2 were given` | `.as_view()` missing in `urls.py`. | `views.StudentListView.as_view()` |
| List page shows "No students found" but data exists | Template uses `students`, view sends `object_list`. | `context_object_name = 'students'` |
| `AttributeError: Generic detail view must be called with either an object pk or a slug` | URL parameter isn't named `pk`. | Use `<int:pk>` or `pk_url_kwarg = 'student_id'`. |
| `TemplateDoesNotExist: students/student_list.html` | Template name doesn't follow the convention. | Rename the file or set `template_name`. |
| `ImproperlyConfigured: Using ModelFormMixin without the 'fields' attribute is prohibited` | Neither `fields` nor `form_class` given. | Add one of them. |
| `ImproperlyConfigured: No URL to redirect to` | No `success_url` and no `get_absolute_url()` on the model. | Add either. |
| `NoReverseMatch` when the server starts | `reverse()` used in a class attribute. | Use `reverse_lazy()`. |
| Page opens without login | Mixin written after the generic view. | `class X(LoginRequiredMixin, CreateView)` |
| Template variable from `get_context_data()` is empty | Method doesn't call `super()` or doesn't `return context`. | Both are required. |
| Page 2 of search results shows all students | `q` not in the pagination links. | Add `&q={{ query\|urlencode }}`. |
| "Students (2)" instead of the total | `students\|length` counts only the current page. | Use `page_obj.paginator.count`. |
| `?page=99` gives 404 | `ListView` rejects invalid page numbers. | Expected behaviour. |

---

## 📋 Quick Reference

```text
URL
  path('', views.StudentListView.as_view(), name='student_list')
  path('<int:pk>/', views.StudentDetailView.as_view(), name='student_detail')

GENERIC VIEWS
  TemplateView   template_name
  ListView       model, context_object_name, get_queryset(), paginate_by
  DetailView     model                     (URL needs <int:pk>, or pk_url_kwarg)
  CreateView     model, form_class, success_url
  UpdateView     model, form_class         (success_url or model.get_absolute_url)
  DeleteView     model, success_url

WHICH METHOD TO OVERRIDE
  Which records?               get_queryset()
  Extra template variables?    get_context_data()     → super() first, return context
  Do something on valid save?  form_valid()           → return super().form_valid(form)

INSIDE A CBV
  self.request   self.request.user   self.request.GET.get('q')   self.kwargs['pk']   self.get_object()

MIXINS (always on the LEFT)
  LoginRequiredMixin
  PermissionRequiredMixin     permission_required = 'students.delete_student'
  UserPassesTestMixin         def test_func(self): return True / False
  SuccessMessageMixin         success_message = "Saved %(name)s"

CLASS ATTRIBUTES THAT ARE URLS
  success_url = reverse_lazy('students:student_list')

PAGINATION
  View:     paginate_by = 10
  Template: page_obj.number   page_obj.paginator.num_pages   page_obj.paginator.count
            page_obj.has_next   page_obj.next_page_number   is_paginated
  Links:    ?page=N{% if query %}&q={{ query|urlencode }}{% endif %}
```

---

## 🔜 Next Lecture

**File Uploads & Media Files:** uploading student photos and documents with `ImageField` / `FileField`, `MEDIA_ROOT` and `MEDIA_URL`, and handling `request.FILES` in forms.
