# Отчёт по практике (группа 1)

**Студент:** Лукашёв Сергей
**Направление:** Информационные системы и программирование  
**Период практики:** 11 мая – 06 июня 2026 г.  
**Сервер:** CloudX (Ubuntu 24.04), IP: 89.208.175.173 / 10.100.8.189  

---

## 📌 Содержание

1. Используемое ПО  
2. Настройка сервера  
3. Пользователи и права  
4. Мониторинг и управление  
5. Веб‑сервер NGINX  
6. Docker, Git, PostgreSQL  
7. Python, виртуальное окружение, Django  
8. Проект Django + СУБД  
9. Приложение с формами (модели, формы, шаблоны)  
10. Доступ к проекту (порт 8080, SSH‑туннель)  
11. Документирование и Git  
12. Итог выполнения задания  

---

## 1. Используемое ПО

- **VS Code** + Remote‑SSH (удалённая разработка)  
- **MobaXTerm** (основной SSH‑клиент, X11, туннели)  
- **Git**, **Docker**, **PostgreSQL 16**  
- **Python 3.12**, **Django 6.0.5**, **psycopg2‑binary**  

---

## 2. Настройка сервера

Подключение к серверу:


ssh -p 2233 cxuser@89.208.175.

Локализация (п. 2 задания):

        bash
        sudo locale-gen ru_RU.UTF-8
        sudo update-locale LANG=ru_RU.UTF-8
        export LANG=ru_RU.UTF-8
Проверка:

        bash
        locale | grep LANG
        # LANG=ru_RU.UTF-8

3. Пользователи и права sudo (п. 3)
Создание пользователей alex, zahar, sergei и добавление в группу sudo:

       bash
       sudo adduser sergei
       sudo usermod -aG sudo sergei
       groups sergei
       # sergei : sergei sudo

5. Инструменты мониторинга и управление (п. 4, 5)
Установка утилит:

        bash
        sudo apt update
        sudo apt install -y htop iotop nethogs iftop cockpit
        Cockpit запущен и доступен (порт 9090):

   bash

        sudo systemctl enable --now cockpit
        sudo ufw allow 9090
5. Веб‑сервер NGINX (п. 6, 7)

        sudo apt install -y nginx
        sudo systemctl enable --now nginx
Настройка:

порт 80 – страница‑заглушка

порт 8080 – прокси на Django (при необходимости)

        nginx
        server {
            listen 80;
            server_name 89.208.175.173;
            location / {
                proxy_pass http://127.0.0.1:8080;
            }
        }
6. Docker, Git, PostgreSQL (п. 8–10)
bash
sudo apt install -y docker.io git postgresql
docker --version
git --version
systemctl status postgresql
PostgreSQL: создана БД django_db и пользователь django_user:

sql
CREATE DATABASE django_db;
CREATE USER django_user WITH PASSWORD 'postgres123';
ALTER USER django_user WITH SUPERUSER;
GRANT ALL PRIVILEGES ON DATABASE django_db TO django_user;
7. Python, виртуальное окружение, Django (п. 12–14)
bash
sudo apt install -y python3-pip python3-venv
mkdir -p ~/django_project
cd ~/django_project
python3 -m venv venv
source venv/bin/activate
pip install django psycopg2-binary
django-admin --version   # 6.0.5
8. Проект Django + СУБД (п. 16)
bash
django-admin startproject myproject .
myproject/settings.py:

python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'django_db',
        'USER': 'django_user',
        'PASSWORD': 'postgres123',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
ALLOWED_HOSTS = ['*']
Миграции и суперпользователь:

bash
python manage.py migrate
python manage.py createsuperuser
9. Приложение с формами ввода/вывода (todo)
9.1. Создание приложения
bash
python manage.py startapp todo
9.2. Модель (todo/models.py)
python
from django.db import models

class Item(models.Model):
    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name
9.3. Форма (todo/forms.py)
python
from django import forms
from .models import Item

class ItemForm(forms.ModelForm):
    class Meta:
        model = Item
        fields = ['name', 'description']
9.4. Представления (todo/views.py)
python
from django.shortcuts import render, redirect
from .models import Item
from .forms import ItemForm

def item_list(request):
    items = Item.objects.all().order_by('-created_at')
    return render(request, 'todo/item_list.html', {'items': items})

def add_item(request):
    if request.method == 'POST':
        form = ItemForm(request.POST)
        if form.is_valid():
            form.save()
            return redirect('item_list')
    else:
        form = ItemForm()
    return render(request, 'todo/add_item.html', {'form': form})
9.5. URL‑маршруты (todo/urls.py)
python
from django.urls import path
from . import views

app_name = 'todo'
urlpatterns = [
    path('', views.item_list, name='item_list'),
    path('add/', views.add_item, name='add_item'),
]
9.6. Подключение маршрутов (myproject/urls.py)
python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('todo.urls')),
]
9.7. Регистрация приложения в INSTALLED_APPS
python
INSTALLED_APPS = [
    ...
    'todo',
]
9.8. Шаблоны
todo/templates/todo/item_list.html – отображение всех записей
todo/templates/todo/add_item.html – форма добавления

9.9. Миграции
bash
python manage.py makemigrations
python manage.py migrate
10. Доступ к проекту на порту 8080 (п. 17)
Запуск сервера разработки:

bash
python manage.py runserver 0.0.0.0:8000
Локальный SSH‑туннель (с рабочей станции):

bash
ssh -L 8080:localhost:8000 -p 2233 sergei@89.208.175.173
Сайт доступен в браузере:

text
http://localhost:8080
11. Документирование работы (п. Доп. опции)
Рабочий процесс и решения описаны в данном MD‑файле (GitHub).

Для контроля версий используется Git.

Проект можно открыть в VS Code через Remote‑SSH.

12. Результаты выполнения задания
№	Задание	Выполнено
1	Установка VS Code, MobaXTerm	✅
2	Локаль сервера – русская	✅
3	Пользователи, группа sudo	✅
4	Утилиты мониторинга	✅
5	Cockpit	✅
6	NGINX	✅
7	Порт 80 – заглушка; порт 8080 – web‑приложение	✅
8	Docker	✅
9	Git	✅
10	PostgreSQL	✅
11	pgAdmin4	⚠️ (опционально)
12	Python, виртуальное окружение	✅
13	Django	✅
14	UV для Python	✅
15	VS Code Remote SSH	✅
16	Django‑проект + СУБД, формы ввода/вывода	✅
17	Доступ на порту 8080 через SSH‑туннель	✅
✨ Заключение
В ходе практики полностью настроен сервер под управлением Ubuntu 24.04, развёрнуто веб‑приложение на Django с использованием PostgreSQL, реализованы формы для ввода и отображения данных. Проект доступен через SSH‑туннель на порту 8080 и задокументирован в репозитории GitHub. Все требования практического задания выполнены.

Репозиторий: ссылка на GitHub
Студенты: Лукашёв С., Прасолов З.
