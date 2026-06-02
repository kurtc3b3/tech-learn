## What & When

**Django** is a **batteries-included** Python web framework — ORM, migrations, admin UI, auth, sessions, forms, and URL routing in one opinionated stack. Best when the product is a **monolith** (CMS, internal ops tool, SaaS with server-rendered pages + REST API via Django REST Framework).

Use Django when:

- You need **admin CRUD** on models without building it
- **Migrations** and relational models are central — PostgreSQL/MySQL
- **Auth** (users, groups, permissions) is built-in
- Team prefers **conventions** over assembling Flask extensions
- **Django REST Framework (DRF)** exposes APIs alongside HTML views

For **API-only microservices**, prefer [[API - FastAPI]]. For tiny tools, [[Web — Flask]]. Overview: [[Web]].

```bash
pip install django
django-admin startproject mysite .
python manage.py startapp blog
# API: pip install djangorestframework
# DB: pip install psycopg[binary]  # PostgreSQL
```

---

## Django vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Full-stack monolith + admin | **Django** | ORM, migrations, auth |
| Minimal custom layout | [[Web — Flask]] | Microframework |
| Async OpenAPI API | [[API - FastAPI]] | ASGI-first |
| SQLAlchemy (not Django ORM) | [[ORM - SQLAlchemy]] + Flask/FastAPI | Outside Django ecosystem |
| Event-loop legacy | [[Web — Tornado]] | Niche |
| Template engine | Django templates or Jinja2 | Flask uses [[Python — Jinja2 Package]] |

---

## Project Layout (MTV)

Django uses **Model–Template–View** (similar to MVC):

```text
mysite/
  manage.py
  mysite/
    settings.py
    urls.py
    wsgi.py
  blog/
    models.py      # Model
    views.py       # View (logic)
    urls.py
    templates/     # Template
    admin.py
```

| Piece | Role |
| --- | --- |
| **Model** | Database schema + queries (`models.py`) |
| **Template** | HTML presentation |
| **View** | Request handler — calls models, picks template |
| **URLconf** | Maps paths to views |

---

## Models & Migrations

```python
# blog/models.py
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    body = models.TextField()
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return self.title
```

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
```

> [!tip] Admin for free Register models in `admin.py` to get CRUD UI immediately.

```python
# blog/admin.py
from django.contrib import admin
from .models import Post

@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ["title", "published", "created_at"]
    list_filter = ["published"]
    prepopulated_fields = {"slug": ("title",)}
    search_fields = ["title", "body"]
```

---

## Views & URLs

### Function-based view

```python
# blog/views.py
from django.shortcuts import render, get_object_or_404
from .models import Post

def post_list(request):
    posts = Post.objects.filter(published=True)
    return render(request, "blog/list.html", {"posts": posts})

def post_detail(request, slug):
    post = get_object_or_404(Post, slug=slug, published=True)
    return render(request, "blog/detail.html", {"post": post})
```

```python
# blog/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path("", views.post_list, name="post_list"),
    path("<slug:slug>/", views.post_detail, name="post_detail"),
]

# mysite/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("blog/", include("blog.urls")),
]
```

### Class-based views

```python
from django.views.generic import ListView, DetailView
from .models import Post

class PostListView(ListView):
    model = Post
    queryset = Post.objects.filter(published=True)
    template_name = "blog/list.html"
    context_object_name = "posts"

class PostDetailView(DetailView):
    model = Post
    slug_field = "slug"
    template_name = "blog/detail.html"
```

---

## Django REST Framework (API)

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Post

class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ["id", "title", "slug", "body", "created_at"]
```

```python
# blog/views.py
from rest_framework import viewsets
from .models import Post
from .serializers import PostSerializer

class PostViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Post.objects.filter(published=True)
    serializer_class = PostSerializer
    lookup_field = "slug"
```

```python
# blog/urls.py (with DRF router)
from rest_framework.routers import DefaultRouter
from .views import PostViewSet

router = DefaultRouter()
router.register("posts", PostViewSet, basename="post")
urlpatterns = router.urls
```

`settings.py`:

```python
INSTALLED_APPS = [
    ...
    "rest_framework",
]
```

Compare OpenAPI auto-generation: [[API - FastAPI — OpenAPI Specification]] — FastAPI generates schema from type hints; DRF needs drf-spectacular or similar for parity.

---

## Settings & Environment

```python
# mysite/settings.py (excerpt)
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
DEBUG = os.environ.get("DJANGO_DEBUG", "0") == "1"
ALLOWED_HOSTS = os.environ.get("ALLOWED_HOSTS", "localhost").split(",")

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.environ["DB_NAME"],
        "USER": os.environ["DB_USER"],
        "PASSWORD": os.environ["DB_PASSWORD"],
        "HOST": os.environ.get("DB_HOST", "localhost"),
        "PORT": os.environ.get("DB_PORT", "5432"),
    }
}
```

Use [[Python — python-dotenv]] locally; split settings modules (`settings/production.py`) for larger teams.

---

## Auth

Built-in user model, login views, permissions:

```python
from django.contrib.auth.decorators import login_required

@login_required
def dashboard(request):
    return render(request, "dashboard.html")
```

DRF: `permission_classes`, `IsAuthenticated`, token/JWT via `djangorestframework-simplejwt`.

For JWT-centric APIs without Django admin, [[API - FastAPI — Dependency Injection & User Management]] is often simpler.

---

## Testing

```python
# blog/tests.py
from django.test import TestCase, Client
from .models import Post

class PostTests(TestCase):
    def setUp(self):
        self.client = Client()
        Post.objects.create(title="Hi", slug="hi", body="...", published=True)

    def test_list(self):
        r = self.client.get("/blog/")
        self.assertEqual(r.status_code, 200)
        self.assertContains(r, "Hi")
```

Also compatible with [[Unit Testing - pytest]] via `pytest-django`.

---

## Production

```bash
pip install gunicorn whitenoise  # static files
gunicorn mysite.wsgi:application -w 4 -b 0.0.0.0:8000
```

| Concern | Approach |
| --- | --- |
| Static files | `collectstatic`, WhiteNoise, or CDN |
| DB | PostgreSQL (not SQLite in prod) |
| Cache | Redis via `django-redis` |
| Tasks | [[Processing — Celery]] + `django-celery-beat` |
| ASGI / WebSocket | Django 4+ Channels (or separate FastAPI service) |

---

## Django ORM vs SQLAlchemy

This vault documents **SQLAlchemy** heavily ([[ORM - SQLAlchemy]]). Django ORM is parallel but different:

| | Django ORM | SQLAlchemy |
| --- | --- | --- |
| Integration | Native to Django | Flask, FastAPI, scripts |
| Migrations | `makemigrations` | Alembic ([[ORM - Migrations]]) |
| Admin | Automatic | Manual / Flask-Admin |
| Async | Django 4.1+ async ORM | [[ORM - Async]] |

Pick **one ORM per service** — do not mix Django ORM and SQLAlchemy in the same Django app without strong reason.

---

## When Django Is the Wrong Choice

- **API-only** service with no admin — [[API - FastAPI]]
- **Serverless** functions — cold start + ORM overhead
- **Heavy async I/O** — possible but FastAPI is smoother
- Team already standardized on SQLAlchemy + thin routes

---

## Quick Reference

| Task | Command / pattern |
| --- | --- |
| New project | `django-admin startproject name .` |
| New app | `python manage.py startapp app` |
| Migrate | `makemigrations` → `migrate` |
| Shell | `python manage.py shell` |
| Run dev | `python manage.py runserver` |
| Admin | `/admin/` after `createsuperuser` |
| List queryset | `Model.objects.filter(...)` |
| DRF viewset | `viewsets.ModelViewSet` + router |

---

## Related Notes

- [[Web]]
- [[Web — Flask]]
- [[Web — Tornado]]
- [[API - FastAPI]]
- [[ORM - SQLAlchemy]]
- [[Processing — Celery]]
- [[Python Development]]

---

## Tags

#web #django #orm #admin #drf #python #backend #monolith #wsgi
