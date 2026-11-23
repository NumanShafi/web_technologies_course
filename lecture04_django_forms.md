# Lecture 04: Forms, Validation & CRUD from the Browser

> **Platform:** Ubuntu
> **Project:** Continue with `student_management` from Lecture 03

## Learning Outcomes

By the end of this lecture you will be able to:

1. Explain the difference between `GET` and `POST` requests and when to use each.
2. Build an HTML form and read its data in a view.
3. Explain what CSRF is and why `{% csrf_token %}` is required.
4. Apply the Post/Redirect/Get pattern using `redirect()`.
5. Create forms from models with `ModelForm`.
6. Validate user input and display errors.
7. Build complete **C**reate, **R**ead, **U**pdate and **D**elete pages for a model.
8. Show success messages with the messages framework.

---

## 01. Where We Are

In Lecture 03 we stored students in the database, but the only ways to add or change them were the **shell** and the **admin panel**. Real users have neither. They need pages of our own site.

| Operation | Lecture 03 | This lecture |
| :--- | :--- | :--- |
| **C**reate | Shell / admin | `/students/add/` |
| **R**ead | `/students/`, `/students/2/` | (already done) |
| **U**pdate | Shell / admin | `/students/2/edit/` |
| **D**elete | Shell / admin | `/students/2/delete/` |

---

## 02. GET vs POST

A browser can send data to the server in two main ways.

| | `GET` | `POST` |
| :--- | :--- | :--- |
| Purpose | **Ask for** data | **Send** data that changes something |
| Where the data travels | In the URL: `/students/?q=ali` | In the request body (not in the URL) |
| Visible in address bar / history | Yes | No |
| Can be bookmarked | Yes | No |
| Use for | Search, filters, opening pages | Create, update, delete, login |
| Read in Django with | `request.GET` | `request.POST` |

> **Rule:** If the request **changes data** on the server, it must be `POST`. Never `GET`.

---

## 03. A Form by Hand (to understand what Django does for us)

### Step 3.1: The template

`students/templates/students/student_add_manual.html`:

```html
{% extends 'base.html' %}

{% block content %}
<h1>Add Student</h1>

<form method="post" action="">
    {% csrf_token %}

    <p>Name: <input type="text" name="name"></p>
    <p>Roll No: <input type="text" name="roll_no"></p>
    <p>CGPA: <input type="number" name="cgpa" step="0.01"></p>

    <button type="submit">Save</button>
</form>
{% endblock %}
```

| Part | Meaning |
| :--- | :--- |
| `method="post"` | Send the data as a `POST` request. |
| `action=""` | Send it to the **same URL** that displayed the form. |
| `name="roll_no"` | The key used to read this value in the view. An input without `name` is not sent. |
| `{% csrf_token %}` | Security token. Required in every `POST` form (explained below). |

### Step 3.2: The view

One view handles **both** requests: `GET` shows the empty form, `POST` saves the data.

```python
# students/views.py
from django.shortcuts import render, redirect
from .models import Student


def student_add_manual(request):
    if request.method == 'POST':
        name = request.POST.get('name')
        roll_no = request.POST.get('roll_no')
        cgpa = request.POST.get('cgpa')

        Student.objects.create(name=name, roll_no=roll_no, cgpa=cgpa)
        return redirect('students:student_list')

    return render(request, 'students/student_add_manual.html')
```

```text
 GET  /students/add/   ──>  show empty form
 POST /students/add/   ──>  read data → save → redirect to the list
```

### Step 3.3: What is wrong with this?

Try submitting it with an empty name, a CGPA of `50`, or a roll number that already exists:

* Empty values are saved, or the page **crashes** with an error.
* A duplicate roll number crashes with `IntegrityError`.
* The user sees no helpful message, and everything typed is lost.
* We wrote every `<input>` by hand, and must do it all again for the edit page.

Django's **forms** solve all of these. But first, two ideas that apply to every form.

---

## 04. CSRF Protection

**CSRF (Cross-Site Request Forgery)** is an attack in which another website secretly submits a form to **your** site using the logged-in user's browser, for example a hidden form that deletes a record.

Django's defence:

1. `{% csrf_token %}` puts a secret, random value in a hidden input of the form.
2. When the form is submitted, Django checks that the value is present and correct.
3. Another website cannot know this value, so its forged request is rejected.

View the page source in the browser to see it:

```html
<input type="hidden" name="csrfmiddlewaretoken" value="k3Jx...">
```

> Forgetting `{% csrf_token %}` in a `POST` form gives **403 Forbidden: CSRF verification failed**.
> `GET` forms (such as search) don't need it, because they don't change data.

---

## 05. Post/Redirect/Get (PRG) and `redirect()`

After a successful `POST`, **never** return a page with `render()`. If the user presses **refresh (F5)**, the browser sends the same `POST` again and the record is saved **twice**.

Instead, answer with a **redirect**. The browser then loads the next page with a normal `GET`, which is safe to refresh.

```text
 Browser                              Server
   │  POST /students/add/  (data)       │
   │ ─────────────────────────────────> │  save to database
   │  302 Redirect → /students/         │
   │ <───────────────────────────────── │
   │  GET /students/                    │
   │ ─────────────────────────────────> │
   │  200 OK (list page)                │
   │ <───────────────────────────────── │
```

```python
from django.shortcuts import redirect

return redirect('students:student_list')                # by URL name
return redirect('students:student_detail', student.id)  # URL name with an argument
```

---

## 06. Django Forms: `ModelForm`

A **`ModelForm`** builds a form **from a model**. It knows the fields, their types and their rules (`max_length`, `unique`, required or optional), so it can:

* generate the HTML inputs,
* validate the submitted data,
* show error messages and keep what the user typed,
* save the object to the database.

### Step 6.1: Create `forms.py`

```bash
touch students/forms.py
```

```python
# students/forms.py
from django import forms
from .models import Student


class StudentForm(forms.ModelForm):
    class Meta:
        model = Student
        fields = ['name', 'roll_no', 'email', 'cgpa', 'department', 'is_active']
```

| Part | Meaning |
| :--- | :--- |
| `forms.ModelForm` | A form connected to a model. |
| `model = Student` | The model to build the form from. |
| `fields = [...]` | Which fields the user may fill in, in this order. |

> `created_at` is not listed because Django fills it automatically. Always list the fields explicitly. Avoid `fields = '__all__'`, which can expose fields users should not edit.

### Step 6.2: The view

```python
# students/views.py
from django.shortcuts import render, redirect, get_object_or_404
from .models import Student
from .forms import StudentForm


def student_add(request):
    if request.method == 'POST':
        form = StudentForm(request.POST)       # form filled with submitted data
        if form.is_valid():                    # run all validation
            form.save()                        # INSERT into the database
            return redirect('students:student_list')
    else:
        form = StudentForm()                   # empty form

    return render(request, 'students/student_form.html', {'form': form})
```

This is the **standard pattern** for every form view. Learn it by heart:

```text
                    ┌───────────────┐
                    │ request.method│
                    └───────┬───────┘
              GET           │            POST
        ┌───────────────────┴───────────────────┐
        ▼                                       ▼
  form = StudentForm()               form = StudentForm(request.POST)
        │                                       │
        │                              form.is_valid() ?
        │                           No  │              │ Yes
        ▼                               ▼              ▼
  render(form)  <────────────  render(form with    form.save()
  (empty form)                 errors + old data)  redirect(...)
```

When the form is **invalid**, the code falls through to the last line and renders the **same** `form` object, which now carries the error messages and the values the user typed.

### Step 6.3: The template

`students/templates/students/student_form.html`:

```html
{% extends 'base.html' %}

{% block title %}Add Student{% endblock %}

{% block content %}
<h1>Add Student</h1>

<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Save</button>
    <a href="{% url 'students:student_list' %}">Cancel</a>
</form>
{% endblock %}
```

* **`{{ form.as_p }}`** prints every field with its label, input and errors, each wrapped in a `<p>`.
* Django does **not** print the `<form>` tag, the CSRF token or the submit button. You write those.

### Step 6.4: The URL

```python
# students/urls.py
urlpatterns = [
    path('', views.student_list, name='student_list'),
    path('add/', views.student_add, name='student_add'),
    path('<int:student_id>/', views.student_detail, name='student_detail'),
]
```

Open **http://127.0.0.1:8000/students/add/** and try it. The department appears as a **dropdown** and `is_active` as a **checkbox**, because the form reads the field types from the model.

---

## 07. Validation

### 7.1 What `is_valid()` checks

| Check | Comes from |
| :--- | :--- |
| Required fields are filled | Fields without `blank=True` |
| Text is not too long | `max_length` |
| Value has the right type | `DecimalField`, `EmailField`, ... |
| Value is not a duplicate | `unique=True` |
| Your own rules | `clean_<field>()` methods (below) |

Try submitting an empty form, a wrong email and a duplicate roll number. Each field shows its own error, and nothing the user typed is lost.

### 7.2 `cleaned_data`

After `is_valid()` returns `True`, the checked and converted values are in **`form.cleaned_data`** (a dictionary). For example `cleaned_data['cgpa']` is a `Decimal`, not a string.

### 7.3 Custom validation: `clean_<fieldname>()`

Our `cgpa` field accepts any number up to `9.99`. A CGPA must be between 0 and 4.

```python
# students/forms.py
class StudentForm(forms.ModelForm):
    class Meta:
        model = Student
        fields = ['name', 'roll_no', 'email', 'cgpa', 'department', 'is_active']

    def clean_cgpa(self):
        cgpa = self.cleaned_data['cgpa']
        if cgpa < 0 or cgpa > 4:
            raise forms.ValidationError("CGPA must be between 0.00 and 4.00.")
        return cgpa

    def clean_roll_no(self):
        return self.cleaned_data['roll_no'].upper()
```

* The method name must be **`clean_` + the field name**.
* Raise **`forms.ValidationError`** to reject the value. The message appears next to that field.
* Always **return** the value. You may return a changed value, as `clean_roll_no` does.

### 7.4 Client-side vs server-side validation

The browser also checks some things (`required`, `type="email"`) before sending. That is only for convenience, since anyone can switch it off or send a request without a browser.

> **Never trust data from the user.** Server-side validation (`is_valid()`) is the one that protects your database.

---

## 08. Customising the Form

### 8.1 Labels, help texts and widgets

```python
class StudentForm(forms.ModelForm):
    class Meta:
        model = Student
        fields = ['name', 'roll_no', 'email', 'cgpa', 'department', 'is_active']
        labels = {
            'roll_no': 'Roll Number',
            'cgpa': 'CGPA',
            'is_active': 'Currently enrolled',
        }
        help_texts = {
            'roll_no': 'Example: BSCS-01',
        }
        widgets = {
            'name': forms.TextInput(attrs={'placeholder': 'Full name', 'class': 'input'}),
            'cgpa': forms.NumberInput(attrs={'step': '0.01', 'min': '0', 'max': '4'}),
        }
```

A **widget** is the HTML element used to display a field. `attrs` adds HTML attributes (`class`, `placeholder`, ...), which is how you attach CSS classes to form inputs.

### 8.2 Rendering fields one by one

`{{ form.as_p }}` is quick, but for full control over the layout, loop over the fields:

```html
<form method="post">
    {% csrf_token %}

    {{ form.non_field_errors }}

    {% for field in form %}
        <div class="form-row">
            {{ field.label_tag }}
            {{ field }}
            {% if field.help_text %}<small>{{ field.help_text }}</small>{% endif %}
            {% for error in field.errors %}
                <span class="error">{{ error }}</span>
            {% endfor %}
        </div>
    {% endfor %}

    <button type="submit">Save</button>
</form>
```

| Template variable | Prints |
| :--- | :--- |
| `{{ field.label_tag }}` | `<label>` for the field |
| `{{ field }}` | The input element |
| `{{ field.errors }}` | Errors of that field |
| `{{ field.help_text }}` | Help text |
| `{{ form.non_field_errors }}` | Errors that don't belong to one field |

---

## 09. Update: Editing a Student

Editing uses the **same form class and the same template**. The only difference is passing the existing object with **`instance=`**.

```python
# students/views.py
def student_edit(request, student_id):
    student = get_object_or_404(Student, id=student_id)

    if request.method == 'POST':
        form = StudentForm(request.POST, instance=student)
        if form.is_valid():
            form.save()                                    # UPDATE, not INSERT
            return redirect('students:student_detail', student.id)
    else:
        form = StudentForm(instance=student)               # pre-filled form

    return render(request, 'students/student_form.html', {'form': form, 'student': student})
```

| Code | Result |
| :--- | :--- |
| `StudentForm()` | Empty form → `save()` **creates** a new row |
| `StudentForm(instance=student)` | Form pre-filled with the student's data |
| `StudentForm(request.POST, instance=student)` | `save()` **updates** that row |

> Forgetting `instance=student` in the `POST` branch is a classic bug: every "edit" creates a **new** student.

Make the shared template show the right heading:

```html
<h1>{% if student %}Edit {{ student.name }}{% else %}Add Student{% endif %}</h1>
```

```python
# students/urls.py
path('<int:student_id>/edit/', views.student_edit, name='student_edit'),
```

---

## 10. Delete: With Confirmation

Deleting changes data, so it must happen on `POST`, and the user should confirm first.

```python
# students/views.py
def student_delete(request, student_id):
    student = get_object_or_404(Student, id=student_id)

    if request.method == 'POST':
        student.delete()
        return redirect('students:student_list')

    return render(request, 'students/student_confirm_delete.html', {'student': student})
```

`students/templates/students/student_confirm_delete.html`:

```html
{% extends 'base.html' %}

{% block content %}
<h1>Delete Student</h1>
<p>Are you sure you want to delete <strong>{{ student.name }}</strong> ({{ student.roll_no }})?</p>

<form method="post">
    {% csrf_token %}
    <button type="submit">Yes, delete</button>
    <a href="{% url 'students:student_detail' student.id %}">Cancel</a>
</form>
{% endblock %}
```

```python
# students/urls.py
path('<int:student_id>/delete/', views.student_delete, name='student_delete'),
```

* `GET /students/2/delete/` shows the confirmation page. Nothing is deleted.
* `POST /students/2/delete/` deletes and redirects.

> Never delete with a plain link (`<a href=".../delete/">`). Links are `GET` requests; browsers, search engines and link previews may open them automatically.

---

## 11. Connecting the Pages

In the list template:

```html
<a href="{% url 'students:student_add' %}">+ Add Student</a>

<table>
    ...
    {% for s in students %}
    <tr>
        <td><a href="{% url 'students:student_detail' s.id %}">{{ s.name }}</a></td>
        <td>{{ s.roll_no }}</td>
        <td>
            <a href="{% url 'students:student_edit' s.id %}">Edit</a> |
            <a href="{% url 'students:student_delete' s.id %}">Delete</a>
        </td>
    </tr>
    {% endfor %}
</table>
```

Final `students/urls.py`:

```python
app_name = 'students'

urlpatterns = [
    path('', views.student_list, name='student_list'),                          # Read (all)
    path('add/', views.student_add, name='student_add'),                        # Create
    path('<int:student_id>/', views.student_detail, name='student_detail'),     # Read (one)
    path('<int:student_id>/edit/', views.student_edit, name='student_edit'),    # Update
    path('<int:student_id>/delete/', views.student_delete, name='student_delete'),  # Delete
]
```

---

## 12. Feedback with the Messages Framework

After a redirect the user lands on the list page with no sign that anything happened. The **messages framework** stores a one-time message that is shown on the **next** page and then disappears.

It is already enabled in `settings.py` (`django.contrib.messages`).

### Step 12.1: Add messages in views

```python
from django.contrib import messages


def student_add(request):
    if request.method == 'POST':
        form = StudentForm(request.POST)
        if form.is_valid():
            student = form.save()
            messages.success(request, f"Student {student.name} added successfully.")
            return redirect('students:student_list')
    ...
```

`form.save()` returns the saved object, so its fields can be used in the message.

| Function | Typical use | `message.tags` |
| :--- | :--- | :--- |
| `messages.success(...)` | Saved, updated | `success` |
| `messages.error(...)` | Something failed | `error` |
| `messages.warning(...)` | Deleted, caution | `warning` |
| `messages.info(...)` | General information | `info` |

### Step 12.2: Display them once, in `base.html`

```html
{% if messages %}
    {% for message in messages %}
        <div class="alert {{ message.tags }}">{{ message }}</div>
    {% endfor %}
{% endif %}
```

```css
/* style.css */
.alert   { padding: 10px; margin: 10px 0; border-radius: 4px; }
.success { background: #d4edda; color: #155724; }
.error   { background: #f8d7da; color: #721c24; }
.warning { background: #fff3cd; color: #856404; }
.error, .errorlist { color: #c0392b; }
```

Because it is in the base template, every page can show messages.

---

## 13. `forms.Form` (forms without a model)

Not every form saves to a model (contact form, search form). For those, use `forms.Form` and define the fields yourself:

```python
class ContactForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea)
```

The view pattern is the same, except there is no `save()`. You read `form.cleaned_data` and decide what to do with it.

| | `forms.Form` | `forms.ModelForm` |
| :--- | :--- | :--- |
| Fields | You define each one | Taken from the model |
| `save()` | No | Yes |
| Use for | Contact, search, login | Create / edit database records |

---

## 🚨 Troubleshooting

| Error | Cause | Fix |
| :--- | :--- | :--- |
| `Forbidden (403) CSRF verification failed` | `{% csrf_token %}` missing in a `POST` form. | Add it inside the `<form>` tag. |
| Form submits but nothing is saved, no error shown | Form is invalid and the template doesn't print errors, or `form.save()` is missing. | Print `{{ form.errors }}`; check the view pattern. |
| Data appears in the URL after submitting | Form uses `GET` (the default). | Add `method="post"`. |
| Record saved twice on refresh | `render()` used after a successful `POST`. | Use `redirect()`. |
| Editing creates a new record | `instance=student` missing in the `POST` branch. | `StudentForm(request.POST, instance=student)`. |
| `The view ... didn't return an HttpResponse object. It returned None` | A branch of the view has no `return` (often wrong indentation of the last `render`). | Make sure every path returns a response. |
| `ImproperlyConfigured: Creating a ModelForm without either the 'fields' attribute or the 'exclude' attribute` | `fields` missing in `Meta`. | Add `fields = [...]`. |
| `KeyError` in `clean_<field>` | Field already failed an earlier check, so it is not in `cleaned_data`. | Use `self.cleaned_data.get('field')` and check for `None`. |
| `NoReverseMatch` on `redirect` | Wrong URL name or missing argument. | Check `app_name:name` and pass the id. |
| Messages don't appear | Loop not in the template, or added after `redirect`. | Put the loop in `base.html`; call `messages...` **before** `return redirect`. |

---

## 📋 Quick Reference

```text
FORM CLASS (students/forms.py)
  class StudentForm(forms.ModelForm):
      class Meta:
          model = Student
          fields = ['name', 'roll_no', ...]
      def clean_cgpa(self): ... raise forms.ValidationError("...") ... return value

VIEW PATTERN
  if request.method == 'POST':
      form = StudentForm(request.POST)            # edit: add instance=student
      if form.is_valid():
          form.save()
          messages.success(request, "...")
          return redirect('students:student_list')
  else:
      form = StudentForm()                        # edit: StudentForm(instance=student)
  return render(request, 'students/student_form.html', {'form': form})

TEMPLATE
  <form method="post">
      {% csrf_token %}
      {{ form.as_p }}
      <button type="submit">Save</button>
  </form>

DELETE
  GET  → confirmation page        POST → obj.delete() → redirect

RULES
  Changes data? → POST.   POST form? → {% csrf_token %}.   Successful POST? → redirect().
```

---

## 🔜 Next Lecture

**Authentication & Authorization:** user registration, login and logout, protecting views with `@login_required`, and showing different content to logged-in users.
