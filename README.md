# Лабораторная работа №4

## Дедлайн сдачи работы без пенальти

TBD

## Как выполнять

Сделать форк вашей лабораторной работы номер три с названием up09-lab4-{ваша_фамилия}

## 1. Создание функционала выхода из аккаунта

1. Добавьте новый путь в `users/urls.py`

```python
path('logout/', views.logout_view, name="logout"),
```

2. Создайте новое представление для приложения пользователей

```python
from django.contrib.auth import login, logout
...
def logout_view(request):
    if request.method == "POST":
        logout(request)
        return redirect("posts:list")
```

3. Создайте кнопку для выхода из аккаунта

```python
{% if user.is_authenticated %}
    <form class="logout" action="{% url 'users:logout' %}" method="post">
        {% csrf_token %}
        <button title="User Logout">Выйти</button>
    </form>
{% else %}
    <a href="{% url 'users:register' %}">
        Зарегистрироваться
    </a>
    <a href="{% url 'users:login' %}">
        Войти
    </a>
{% endif %}
```

## 2. Создание авторизированных страниц

1. Создайте новый маршрут в `posts/urls.py`

```python
urlpatterns = [
    path('', views.posts_list, name="list"),
    path('new-post/', views.post_new, name="new-post"),
    path('<slug:slug>', views.post_page, name="page"),
]
```

2. Создайте новое представление для приложения постов

```python
from django.contrib.auth.decorators import login_required
...
@login_required(login_url="/users/login/")
def post_new(request):
    return render(request, 'posts/post_new.html')
```

3. Создайте новый шаблон `posts/post_new.html`

```html
{% extends 'layout.html' %} {% block title %} Новый пост {% endblock %}
{% block content %}
<section>
    <h1>Новый пост</h1>
</section>
{% endblock %}
```

4. Добавьте ссылку на страницу создания нового поста в навигацию

```html
...
{% if user.is_authenticated %}
<a href="{% url 'posts:new-post' %}">Новый пост</a>
...
```

> [!NOTE]
> Проверьте, что кнопка работает (если вы зашли под пользователем - переводит на страницу с новым постом, если не под пользователем, то переводит на страницу логина)

5. Добавьте код в шаблон `users/templates/users/login.html`

```html
{% if request.GET.next %}
    <input type="hidden" name="next" value="{{ request.GET.next }}" />
{% endif %}
```

Этот код создает скрытое поле ввода, в которое помещается адрес откуда юзер был переведен на эту страницу, чтобы в случае успешного логина отправить его обратно на предыдущую страницу.

Также для этого функционала необходимо изменить представление `login_view`

```python
...
        if form.is_valid(): 
            login(request, form.get_user())
            if 'next' in request.POST:
                return redirect(request.POST.get('next'))
            else:
                return redirect("posts:list")
    else: 
        form = AuthenticationForm()
...
```

## 3. Создание формы для постов

1. Создайте форму `posts/forms.py`

```python
from django import forms 
from . import models 

class CreatePost(forms.ModelForm): 
    class Meta: 
        model = models.Post
        fields = ['title','body','slug','banner']
```

- Класс `Meta` - зарезервированный класс в Django
- Как полe передается массив строк, где указаны __только нужные для ввода__ поля (отсутствует date, который задается автоматически)

2. Измение представление `post_new`, чтобы оно могло обрабатывать POST запрос (создавая пост) и GET (отправляя форму на создание поста)

```python
from . import forms 

@login_required(login_url="/users/login/")
def post_new(request):
    if request.method == 'POST': 
        form = forms.CreatePost(request.POST, request.FILES) 
        if form.is_valid():
            newpost = form.save(commit=False) 
            newpost.author = request.user 
            newpost.save()
            return redirect('posts:list')
    else:
        form = forms.CreatePost()
    return render(request, 'posts/post_new.html', { 'form': form })
```

3. Измените шаблон `post_new.html`, добавив в него форму для создания нового поста

```html
{% extends 'layout.html' %}

{% block title %}
   New Post
{% endblock %}

{% block content %}
    <section>
        <h1>New Post</h1>
        <form class="form-with-validation" action="{% url 'posts:new-post' %}" method="post" enctype="multipart/form-data">
            {% csrf_token %} 
            {{ form }}
            <button class="form-submit">Добавить пост</button>
        </form>
    </section>
{% endblock %}
```

4. Добавьте автора в модель `Post`

```python
from django.contrib.auth.models import User
...
author = models.ForeignKey(User, on_delete=models.CASCADE, default=None)
```

_Не забудьте применить миграцию_

> [!WARNING]
> 
> __Задание 1__
> 
> Добавьте указание автора в шаблоны, связанные с постами


> [!WARNING]
> 
> __Задание 2__
> 
> Создайте функционал добавления сообществ через кастомную форму

## 4. Приготовление сервиса к продакшену

1. Изменение режима сервера

Для того, чтобы подготовить сервис к развороту на сервере - необходимо отключить DEBUG режим и добавить адреса, с которых разрешено получать доступ к ресурсу в `lab1/settings.py`

```python
DEBUG = False

ALLOWED_HOSTS = ["localhost","127.0.0.1"]
```

2. Изменение метода раздачи статичных материалов

- В `lab1/urls.py` необходимо добавить новые маршруты, которые будут раздавать статические файлы

```python
from django.urls import path, include, re_path
from django.views.static import serve

urlpatterns = [
    re_path(r'^media/(?P<path>.*)$', serve, {'document_root': settings.MEDIA_ROOT}),
    re_path(r'^static/(?P<path>.*)$', serve, {'document_root': settings.STATIC_ROOT}),
    ...
```

_Удалите строчку, где мы добавляли статические файлы методом добавления нового элемента к массиву в конце файла_

- В `lab1/settings.py` необходимо обновить методы для сбора и хранения статических файлов

    - Удалить `import os`
    - Заменить весь блок, где указываются статические файлы на

```python
STATIC_URL = 'static/'
MEDIA_URL = 'media/'

STATIC_ROOT = BASE_DIR / 'assets'
MEDIA_ROOT = BASE_DIR / 'media'

STATICFILES_DIRS = [
    BASE_DIR / 'static'
]
```

3. Организуйте статические файлы согласно новой структуре

`py manage.py collectstatic`

> [!WARNING]
> 
> __Задание 3__
> 
> Добавьте стилей на сайт (качество верстки будет оцениваться)
