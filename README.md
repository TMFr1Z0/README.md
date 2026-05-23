# Отчёт по практике

**Студент:** Лукашов Сергей

**Направление:** Информационные системы и программирование

**Дата:** 11 мая - 06 июня 2026 г.

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
- **MobaXTerm** (использовался как основной SSH-клиент)

```bash
# Проверка установки MobaXTerm (локально)
# MobaXTerm Personal Edition v26.3
```

---

## 2. Изменение локали сервера на русскую

Подключение к серверу:

```bash
ssh -i ~/Desktop/edu_ke -p 2233 cuxer@89.208.175.173
```

Смена локали:

```bash
sudo locale-gen ru_RU.UTF-8
sudo update-locale LANG=ru_RU.UTF-8 LC_ALL=ru_RU.UTF-8
export LANG=ru_RU.UTF-8
export LC_ALL=ru_RU.UTF-8

# Проверка
locale
date
```

**Результат:** Система отображает дату и сообщения на русском языке.

---

## 3. Создание пользователей, включённых в группу SUDO

Создан пользователь `sergei`:

```bash
sudo adduser sergei
sudo usermod -aG sudo sergei
groups sergei  # sergei : sergei sudo users docker
```

Настройка SSH-доступа для `sergei`:

```bash
sudo mkdir -p /home/sergei/.ssh
sudo cp /home/cxuser/.ssh/authorized_keys /home/sergei/.ssh/
sudo chown -R sergei:sergei /home/sergei/.ssh
sudo chmod 700 /home/sergei/.ssh
sudo chmod 600 /home/sergei/.ssh/authorized_keys
```

**Результат:** Пользователь `sergei` имеет права sudo и может подключаться по SSH.

---

## 4. Установка ПО для мониторинга

```bash
sudo apt update
sudo apt install -y htop iotop net-tools nethogs iftop sysstat nmon
sudo systemctl enable --now sysstat
```

**Установленные инструменты:**

| Инструмент | Назначение |
|------------|------------|
| htop | Интерактивный просмотр процессов |
| nmon | Системный мониторинг |
| iotop | Мониторинг дисковых операций |
| iftop | Мониторинг сетевого трафика |
| nethogs | Трафик по процессам |
| sysstat | Сбор статистики (sar) |

```bash
# Проверка
htop --version   # htop 3.3.0
nmon --version   # nmon 16p
sar -V           # sysstat 12.6.1
```

---

## 5. Установка системы управления сервером (Cockpit)

```bash
sudo apt install -y cockpit
sudo systemctl enable --now cockpit
sudo ufw allow 9090
```

**Проверка:**

```bash
sudo ss -tlnp | grep 9090
# LISTEN 0 4096 *:9090 *:*
curl -I http://localhost:9090
# HTTP/1.1 200 OK
```

**Доступ:** `https://89.208.175.173:9090` (требуется открыть порт в CloudX)

---

## 6-7. Установка и настройка WEB-сервера NGINX

### Установка NGINX

```bash
sudo apt install -y nginx
sudo systemctl enable --now nginx
```

### Страница-заглушка на порту 80

```bash
sudo tee /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Сервер работает</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 50px; }
        h1 { color: #4CAF50; }
    </style>
</head>
<body>
    <h1>Страница-заглушка</h1>
    <p>Сервер успешно настроен и работает.</p>
</body>
</html>
EOF
```

### Настройка прокси для Django (порт 80 → 8080)

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo tee /etc/nginx/sites-available/django << 'EOF'
server {
    listen 80;
    server_name 89.208.175.173;
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
sudo ln -s /etc/nginx/sites-available/django /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 8. Установка Docker

```bash
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker
sudo usermod -aG docker sergei
```

**Проверка:**

```bash
docker --version
# Docker version 27.5.1, build 3f4a2b1
```

---

## 9. Установка Git, настройка

```bash
sudo apt install -y git
git config --global user.name "sergei"
git config --global user.email "sergei@example.com"
```

**Проверка:**

```bash
git --version
# git version 2.43.0
```

---

## 10. Установка СУБД PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable --now postgresql
```

### Создание базы данных и пользователя

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE django_db;
CREATE USER django_user WITH PASSWORD 'secure_password_123';
ALTER USER django_user WITH SUPERUSER;
GRANT ALL PRIVILEGES ON DATABASE django_db TO django_user;
\q
```

---

## 11. Установка pgAdmin4 (web-система управления СУБД)

```bash
sudo docker run -d --name pgadmin4 -p 5050:80 \
  -e PGADMIN_DEFAULT_EMAIL=admin@example.com \
  -e PGADMIN_DEFAULT_PASSWORD=admin123 \
  -e PGADMIN_CONFIG_SERVER_MODE=False \
  --restart unless-stopped \
  dpage/pgadmin4
```

**Проверка:**

```bash
sudo docker ps | grep pgadmin4
```

**Доступ:** `http://89.208.175.173:5050` (логин: `admin@example.com`, пароль: `admin123`)

---

## 12-13. Установка Python, создание виртуального окружения, установка Django

```bash
sudo apt install -y python3-pip python3-venv python3-dev
mkdir -p ~/django_project
cd ~/django_project
python3 -m venv venv
source venv/bin/activate
pip install django psycopg2-binary
```

**Проверка:**

```bash
python --version    # Python 3.12.3
django-admin --version  # Django 6.0.5
```

### Создание проекта Django

```bash
django-admin startproject myproject .
```

### Настройка подключения к PostgreSQL (`myproject/settings.py`)

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'django_db',
        'USER': 'django_user',
        'PASSWORD': 'secure_password_123',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
ALLOWED_HOSTS = ['89.208.175.173', 'localhost', 'srv-171-28', '127.0.0.1']
```

### Миграции и суперпользователь

```bash
python manage.py migrate
python manage.py createsuperuser
# Username: sergei
# Email: (skip)
# Password: [задан при создании]
```

---

## 14. Использование UV для Python (дополнительно)

```bash
pip install uv
uv pip install django psycopg2-binary
```

UV работает значительно быстрее стандартного pip.

---

## 15. Настройка VS Code для удалённой разработки по SSH

### Установка расширения Remote-SSH в VS Code

### Конфигурация SSH (`C:\Users\Имя\.ssh\config`)

```ssh-config
Host cloud-server
    HostName 89.208.175.173
    Port 2233
    User sergei
    IdentityFile C:\Users\Имя\.ssh\edu_ke
```

### Подключение

1. `F1` → `Remote-SSH: Connect to Host` → `cloud-server`
2. Ввод пароля (если требуется)
3. Открытие папки: `/home/sergei/django_project`

**Результат:** Возможность редактировать файлы на сервере и выполнять команды через встроенный терминал VS Code.

---

## 16. Создание проекта на Django в связке с СУБД

### Создание приложения

```bash
cd ~/django_project
python manage.py startapp myapp
```

### Модель (`myapp/models.py`)

```python
from django.db import models

class Item(models.Model):
    name = models.CharField(max_length=100, verbose_name="Название")
    description = models.TextField(verbose_name="Описание")
    created_at = models.DateTimeField(auto_now_add=True, verbose_name="Дата создания")

    def __str__(self):
        return self.name
```

### Форма (`myapp/forms.py`)

```python
from django import forms
from .models import Item

class ItemForm(forms.ModelForm):
    class Meta:
        model = Item
        fields = ['name', 'description']
```

### Представления (`myapp/views.py`)

```python
from django.shortcuts import render, redirect
from .models import Item
from .forms import ItemForm

def item_list(request):
    items = Item.objects.all().order_by('-created_at')
    return render(request, 'myapp/item_list.html', {'items': items})

def add_item(request):
    if request.method == 'POST':
        form = ItemForm(request.POST)
        if form.is_valid():
            form.save()
            return redirect('item_list')
    else:
        form = ItemForm()
    return render(request, 'myapp/add_item.html', {'form': form})
```

### URL-маршруты приложения (`myapp/urls.py`)

```python
from django.urls import path
from . import views

app_name = 'myapp'

urlpatterns = [
    path('', views.item_list, name='item_list'),
    path('add/', views.add_item, name='add_item'),
]
```

### Подключение в основном `myproject/urls.py`

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('myapp.urls')),
]
```

### Регистрация приложения в `settings.py`

```python
INSTALLED_APPS = [
    ...
    'myapp',
]
```

### Шаблоны

**`myapp/templates/myapp/item_list.html`** — список элементов с кнопкой добавления

**`myapp/templates/myapp/add_item.html`** — форма добавления элемента

### Применение миграций

```bash
python manage.py makemigrations
python manage.py migrate
```

---

## 17. Доступ проекта на порту 8080

### Запуск сервера разработки

```bash
python manage.py runserver 0.0.0.0:8080
```

### Проверка локально на сервере

```bash
curl -I http://localhost:8080
# HTTP/1.1 200 OK
# Server: WSGIServer/0.2 CPython/3.12.3
```

### SSH-туннель для доступа с локального ПК

Так как порт 8080 не был открыт в CloudX, использован SSH-туннель:

```powershell
# В PowerShell на локальном ПК
ssh -L 8080:localhost:8080 -p 2233 sergei@89.208.175.173
```

В браузере: `http://localhost:8080`

**Результат:** Web-приложение с формами ввода и вывода данных доступно по адресу `http://localhost:8080`.

---

## 18. Сложности и их решения

| Сложность | Решение |
|-----------|---------|
| Ошибка `permission denied for schema public` в PostgreSQL | Выполнено `ALTER USER django_user WITH SUPERUSER;` |
| Порт 8080 недоступен снаружи (блокировка CloudX) | Использован SSH-туннель: `ssh -L 8080:localhost:8080 ...` |
| Ошибка 502 Bad Gateway при обращении к порту 80 | Перезапущен Django и NGINX, проверена конфигурация прокси |
| Порт 8080 занят NGINX | Остановлен NGINX: `sudo systemctl stop nginx` |
| Команда `python manage.py runner` вместо `runserver` | Исправлено на `python manage.py runserver` |
| Ошибка `ModuleNotFoundError: No module named 'myapp'` после удаления | Удалён `myapp` из `INSTALLED_APPS`, затем создан заново |

---

## 19. Заключение

**Все пункты практического задания выполнены в полном объёме:**

| № | Задание | Статус |
|---|---------|--------|
| 1 | Установка VS Code, MobaXTerm | ✅ |
| 2 | Изменение локали на русскую | ✅ |
| 3 | Создание пользователя с правами sudo | ✅ |
| 4 | Установка ПО для мониторинга | ✅ |
| 5 | Установка Cockpit | ✅ |
| 6 | Установка NGINX | ✅ |
| 7 | Порт 80 – заглушка, порт 8080 – web-приложение | ✅ |
| 8 | Установка Docker | ✅ |
| 9 | Установка Git | ✅ |
| 10 | Установка PostgreSQL | ✅ |
| 11 | Установка pgAdmin4 | ✅ |
| 12-13 | Установка Python, Django | ✅ |
| 14 | Использование UV | ✅ |
| 15 | VS Code Remote SSH | ✅ |
| 16 | Django проект с формами ввода/вывода | ✅ |
| 17 | Доступ на порту 8080 | ✅ |
