# Отчёт по практике

**Студент:** Лукашёв Сергей

**Направление:** Информационные системы и программирование

**Дата:** 11 мая – 06 июня 2026 г.

**Сервер:** 89.208.175.173:2233 (CloudX, Ubuntu 24.04)

---

## Содержание

1. Установка клиентского ПО
2. Изменение локали сервера
3. Создание пользователей с правами sudo
4. Установка ПО для мониторинга
5. Установка системы управления сервером (Cockpit)
6. Установка и настройка WEB-сервера NGINX
7. Порт 80 – страница-заглушка, порт 8080 – для web-приложения
8. Установка Docker
9. Установка Git, настройка
10. Установка СУБД PostgreSQL
11. Установка web-системы управления СУБД (pgAdmin4)
12. Установка Python, создание виртуального окружения
13. Установка Django
14. Использование UV для Python
15. Настройка VS Code для удалённой разработки по SSH
16. Создание проекта на Django в связке с СУБД
17. Доступ проекта на порту 8080
18. Сложности и их решения
19. Заключение

---

## 1. Установка клиентского ПО

На локальный ПК установлены:
- **VS Code** с расширением Remote-SSH
- **MobaXTerm** (основной SSH-клиент)

```bash

# MobaXTerm Personal Edition v26.3
2. Изменение локали сервера на русскую
Подключение к серверу:

bash
ssh -p 2233 cxuser@89.208.175.173
Смена локали:

bash
sudo locale-gen ru_RU.UTF-8
sudo update-locale LANG=ru_RU.UTF-8
export LANG=ru_RU.UTF-8
Результат: Система отображает дату и сообщения на русском языке.

3. Создание пользователей, включённых в группу SUDO
Создан пользователь sergei:

bash
sudo adduser sergei
sudo usermod -aG sudo sergei
groups sergei
Настройка SSH-доступа для sergei:

bash
sudo mkdir -p /home/sergei/.ssh
sudo cp /home/cxuser/.ssh/authorized_keys /home/sergei/.ssh/
sudo chown -R sergei:sergei /home/sergei/.ssh
sudo chmod 700 /home/sergei/.ssh
sudo chmod 600 /home/sergei/.ssh/authorized_keys
Результат: Пользователь sergei имеет права sudo.

4. Установка ПО для мониторинга
bash
sudo apt update
sudo apt install -y htop iotop nethogs iftop
Установленные инструменты: htop, iotop, iftop, nethogs.

5. Установка системы управления сервером (Cockpit)
bash
sudo apt install -y cockpit
sudo systemctl enable --now cockpit
Доступ: https://89.208.175.173:9090

6–7. Установка и настройка NGINX
bash
sudo apt install -y nginx
sudo systemctl enable --now nginx
Страница-заглушка на порту 80:

bash
echo "<h1>Сервер работает</h1>" | sudo tee /var/www/html/index.html
8. Установка Docker
bash
sudo apt install -y docker.io
docker --version
9. Установка Git
bash
sudo apt install -y git
git config --global user.name "Sergei"
git config --global user.email "sergei@example.com"
10. Установка PostgreSQL
bash
sudo apt install -y postgresql
sudo systemctl enable --now postgresql
Создание базы данных и пользователя:

bash
sudo -u postgres psql
sql
CREATE DATABASE django_db;
CREATE USER django_user WITH PASSWORD 'postgres123';
ALTER USER django_user WITH SUPERUSER;
GRANT ALL PRIVILEGES ON DATABASE django_db TO django_user;
\q
11. Установка pgAdmin4
bash
sudo docker run -d --name pgadmin4 -p 5050:80 \
  -e PGADMIN_DEFAULT_EMAIL=admin@example.com \
  -e PGADMIN_DEFAULT_PASSWORD=admin \
  dpage/pgadmin4
Доступ: http://89.208.175.173:5050 (логин: admin@example.com, пароль: admin)

12–13. Установка Python и Django
bash
sudo apt install -y python3-pip python3-venv
mkdir -p ~/django_project
cd ~/django_project
python3 -m venv venv
source venv/bin/activate
pip install django psycopg2-binary
bash
python --version      # Python 3.12.3
django-admin --version # Django 6.0.5
14. Использование UV для Python
bash
pip install uv
uv pip install django psycopg2-binary
15. Настройка VS Code для удалённой разработки по SSH
Установлено расширение Remote-SSH.
Конфиг ~/.ssh/config:

ssh-config
Host cloud-server
    HostName 89.208.175.173
    Port 2233
    User sergei
Подключение: F1 → Remote-SSH: Connect to Host → cloud-server

16. Создание проекта Django в связке с СУБД
bash
django-admin startproject myproject .
settings.py:

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
Миграции:

bash
python manage.py migrate
python manage.py createsuperuser
Создание приложения todo:

bash
python manage.py startapp todo
Модель (todo/models.py):

python
from django.db import models

class Item(models.Model):
    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
Форма (todo/forms.py):

python
from django import forms
from .models import Item

class ItemForm(forms.ModelForm):
    class Meta:
        model = Item
        fields = ['name', 'description']
Представления (todo/views.py):

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
URL‑маршруты (todo/urls.py):

python
from django.urls import path
from . import views

app_name = 'todo'
urlpatterns = [
    path('', views.item_list, name='item_list'),
    path('add/', views.add_item, name='add_item'),
]
Главный urls.py:

python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('todo.urls')),
]
Шаблоны:

todo/templates/todo/item_list.html – список записей

todo/templates/todo/add_item.html – форма добавления

Миграции:

bash
python manage.py makemigrations
python manage.py migrate
17. Доступ проекта на порту 8080
Запуск сервера:

bash
python manage.py runserver 0.0.0.0:8000
SSH-туннель (с локального ПК):

powershell
ssh -L 8080:localhost:8000 -p 2233 sergei@89.208.175.173
В браузере: http://localhost:8080

18. Сложности и их решения
Сложность	Решение
Ошибка permission denied for schema public в PostgreSQL	Выполнено ALTER USER django_user WITH SUPERUSER;
Порт 8080 недоступен снаружи	Использован SSH-туннель
Ошибка NameError: name 'include' is not defined	Добавлен импорт include в urls.py
Порт 8000 уже используется	Найден и остановлен процесс через pkill -f runserver
19. Заключение
Все пункты практического задания выполнены:

№	Задание	Статус
1	Установка VS Code, MobaXTerm	✅
2	Изменение локали на русскую	✅
3	Создание пользователя с правами sudo	✅
4	Установка ПО для мониторинга	✅
5	Установка Cockpit	✅
6	Установка NGINX	✅
7	Порт 80 – заглушка, порт 8080 – web-приложение	✅
8	Установка Docker	✅
9	Установка Git	✅
10	Установка PostgreSQL	✅
11	Установка pgAdmin4	✅
12–13	Установка Python, Django	✅
14	Использование UV	✅
15	VS Code Remote SSH	✅
16	Django проект с формами ввода/вывода	✅
17	Доступ на порту 8080	✅
