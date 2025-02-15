# Урок 47. Media, static. Amazon EC2. Deployment, gunicorn + nginx.

![](https://res.cloudinary.com/practicaldev/image/fetch/s--q_bdVQkA--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/9am30hkx4lfoxubt48sv.png)

## Static и Media файлы

Static файлы - файлы которые не являются частью обязательных файлов для работы системы ( `*.py`, `*.html`), но
необходимы для изменения отображения (`*.css`, `*.js`, `*.jpg`). К таким файлам есть доступ из кода, и обычно они не
могут быть изменены с пользовательской стороны (шрифты на сайте, css стили, картинка на бекграунде итд.)

Media файлы - файлы загруженные пользователями (в независимости от привилегий), например, аватарки, картинки товаров,
голосовые сообщения :) К таким файлам доступа в репозитории нет и не должно быть.

### Где их хранить

Статик файлы, практически всегда разбросанные по приложениям проекта, что очень удобно при разработке, но при их
использовании гораздо проще хранить в одном месте, и тоже вне проекта.

Медиа файлы нужно хранить отдельно от статики, иначе можно получить большое кол-во проблем.

Проще всего создать две отдельные папки `static` и `media` или одну папку `files`, а в ней уже подпапки статики и медии.

### Как это настраивается в Django

В рамках Django, при базовом создании проекта, в `setting.py`, в `INSTALLED_APPS` автоматически
добавляется `django.contrib.staticfiles`, именно это приложение отвечает за то как будут обрабатываться статические
файлы.

```python
# settings.py
STATIC_URL = '/static/'
```

Указанный статик урл будет преобразован в урл по которому можно получить статические файлы.

Например, `"/static/"` или `"http://static.example.com/"`

В первом случае, при запросе к статике будет выполнен на текущий урл, с приставкой `http://127.0.0.1:8000/static/`

Во втором, запросы будут произведены, на отдельный урл.

### Где такой урл будет сгенерирован

В шаблоне где мы можем использовать темплейттег `static`:

```html
{% load static %}
<img src="{% static 'my_app/example.jpg' %}" alt="My image">
```

### Переменная DEBUG

В `settings.py` есть переменная `DEBUG` по дефолту она равна тру, но за что она отвечает?

В основном она отвечает за то как вести себя при ошибках (чаще всего 500-х), Если вы делаете неправильный запрос (не
туда, не те данные, итд.) то вы видите подробное описание того, почему ваш запрос не удался, на какой строчке кода упал,
или нет такого урла, но вот такие есть. Всё это отображается только потому что переменная ```DEBUG=True```. Запущенный
сайт никогда не покажет вам эту информацию.

### Переменная DEBUG и runserver

На самом деле если у вас ```DEBUG=True``` и вы запускаете команду `runserver`, то запускается еще
и ` django.contrib.staticfiles.views.serve()`

Который позволяет отображать статические файлы в процессе разработки, при загрузке проекта в реальное использование,
переменную `DEBUG` нужно установить в `False` и обрабатывать статические файлы внешними средствами, поговорим о них
ниже.

### STATICFILES_DIRS

Список папок в которых хранится ваша статика для разработки, чаще всего это папки статики в разных приложениях,
например `authenticate/static`, `billing/static`, итд.

НЕ ДОЛЖЕН СОДЕРЖАТЬ ЗНАЧЕНИЕ ИЗ ПЕРЕМЕННОЙ STATIC_ROOT !!!!

### STATIC_ROOT

Переменная, в которой хранится путь к папке, в которую всю найденную статику из папок в переменной STATICFILES_DIRS
соберёт команда `python manage.py collectstatic`

Как это работает на практике. При разработке используются статические файлы из папок разных приложений, а для
продакшена, настраивается скрипт, который при любом изменении будет запускать команду `collectstatic` которая будет
собирать всю статику в то место которое уже обрабатывается сторонними сервисами, о которых ниже.

### MEDIA_URL

По аналогии со статикой, такая же настройка для медиа.

### MEDIA_ROOT

Абсолютный путь к папке в которой мы будем хранить пользовательские файлы.

Медиа собирать не нужно, так как мы не можем её менять, это только пользовательская привилегия.

## Deployment

Что такое деплой? Это развертывание вашего проекта для использования его из интернета, а не локально.

![](https://pics.me.me/when-people-ask-how-the-deployment-is-going-im-fine-52721062.png)

Что для этого необходимо? Нужен сервер (на самом деле им может быть любое устройство, комп, телефон, итд.), но у сервера
есть одна особенность, ему нужно работать всегда, мы же не хотим, что бы наш сайт\приложение переставало работать.

Для того что бы организовать 100% аптайм чаще всего используются облачные сервера, большое кол-во компаний готово
предоставить такие сервера за оплату. Мой личный опыт говорит о том, что самые надёжные и самые часто используемые это
сервера компании Амазон, так же у амазона очень большая инфраструктура и экосистема для обслуживания сервером, но об
этом в следующий раз.

## Amazon EC2

Сервис который предоставляет нам выделенные мощности называется EC2

Для начала нам необходим аккаунт на платформе AWS (Amazon Web Services).

[Ссылка на AWS](https://aws.amazon.com/)

![](https://djangoalevel.s3.eu-central-1.amazonaws.com/lesson47/aws.png)

У амазона существуют сотни различных сервисов, для различных задач, но на данном этапе нас интересует только EC2.

EC2 это сервис, который позволяет запускать виртуальные сервера с различной мощностью, на различных операционных
системах.

Для нашего случая мы будем рассматривать самый слабый по характеристикам сервер(t2.nano) на базе линукса (Ubuntu Server
18.04 LTS).

![](https://djangoalevel.s3.eu-central-1.amazonaws.com/lesson47/ec2-ubuntu.png)

В следующем экране выбираем мощность сервера, и пару ключей для подключения по ssh.

## Общая теория по деплою Django

Для деплоя джанго приложения, используется два различных сервера, первый для запуска приложения локально на сервере,
второй используется как прокси, что бы выход в интернет связать с этим локально запущенном сервере и предоставить доступ
к статике и медиа. За обработку данных будет отвечать первый сервер, за безопасность и распределение нагрузок второй

Первый сервер это так называемый WSGI (Web-Server Gateway Interface), работает примерно так:

![](http://lectures.uralbash.ru/_images/server-app.png)

В качестве WSGI сервера может быть использовано достаточно большое кол-во разных серверов:

- uWSGI
- Bjoern
- uWSGI
- mod_wsgi
- Meinheld
- Gunicorn

И так далее, это далеко не полный список. Мы, в качестве примера, будем использовать Gunicorn (на моей практике самый
используемый сервер)

Так же существует технология ASGI это улучшение технологии WSGI, основные серверы для ASGI:

- Daphne
- Hypercorn
- Uvicorn

Так же часто используемые технологии, и скорее всего дальше будет использоваться всё чаще.

В качестве прокси сервера может быть использованы:

- Nginx
- Apache

### runserver

Может возникнуть идея, а почему бы не использовать команду рансервер, зачем нам вообще какие-то дополнительные сервера?
Команда ран сервер не предполагает даже относительно серьезных нагрузок, даже при условных 100 пользователях базовая
команда будет захлёбываться

## Пример развёртывания

Рассмотрим как развернуть наш проект на примере, EC2(Ubuntu 18.04) + Gunicorn + Nginx

Для начала необходимо зайти на наш EC2 сервер при помощи ssh.

Если вы используете Windows, то самый простой способ использовать ssh это либо клиент putty, либо установить Git CLI,
гит интерфейс поддерживает команду ssh.

Свежесозданный инстанс не содержит вообще ничего, даже интерпретатора питона, а значит нам необходимо его установить, но
вместе с ними, установим и другие нужные пакеты (бд, сервера итд.).

```
sudo apt update
sudo apt install python3-pip python3-dev python3-venv libpq-dev postgresql postgresql-contrib nginx curl
```

### База данных

Как создать базу данных и пользователя с доступом к ней вы уже знаете, заходим в консоль постгрес и делаем это:

```sudo -u postgres psql```

User: myuser

DB: mydb

password: mypass

## Переменные операционной системы

Для того что бы вносить некоторые параметры в код используются переменные операционной системы, допустим в
файле `settings.py` мы можем хранить пароль от базы данных, ключи от сервисов и много другой информации которую нельзя
разглашать, но в случае если вы оставите эти данные в коде, они попадут на гит, этого допускать нельзя.

Для использования переменных в ОС Linux используется команда `export var_name="value"`

Если просто в консоль внести команду экспорта она обнулится после перезагрузки инстанса, нас это не устраивает, поэтому
экспорт переменных нужно вносить в файл загружаемый при каждом запуске, например `~/.bashrc`,
открываем `sudo nano ~/.bashrc` и в самом конце дописываем:

```
export PROD='True';
export DBNAME='mydb';
export DBUSER='myuser';
export DBPASS='mypass';
```

Не забываем выполнить source, что бы применить эти изменения

Для gunicorn проще занести все переменные в отдельный файл, и мы будем использовать его в дальнейшем, создадим еще один файл с этими же переменными.

```sudo nano /home/ubuntu/.env```

```
PROD='True'
DBNAME='mydb'
DBUSER='myuser'
DBPASS='mypass'
```

### Правки в `settings.py`

Один из удобных способов разделить настройки на локальные и продакшен, это всё теже переменные операционной системы,
например, добавить в проект, на уровне файла `settings.py` еще два файла `settings_prod.py` и `settings_local.py`, в
основной файл нужно импортировать модуль `os` и в конце дописать:

```python
if os.environ.get('PROD'):
    try:
        from .settings_prod import *
    except ImportError:
        pass
else:
    try:
        from .settings_local import *
    except ImportError:
        pass
```

Тем самым мы сможем разделить настройки в зависимости от того есть ли в операционной системе переменная `PROD`.

В `settings_prod.py` укажем:

```python
import os

DEBUG = False
ALLOWED_HOSTS = ['54.186.155.252']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DBNAME'),
        'USER': os.environ.get('DBUSER'),
        'PASSWORD': os.environ.get('DBPASS'),
        'HOST': os.environ.get('DBHOST', '127.0.0.1'),
        'PORT': os.environ.get('DBPORT', '5432'),
    }
}
```

`DEBUG` фолс, так как нам не нужно отображать подробности ошибок.

В `ALLOWED_HOSTS` нужно указывать URL и/или IP по которому будет доступно приложенние, как указать там урл мы поговорим
на следующем занятии, а пока можно указать там IP, который мы получили у амазона после создания инстанса:

![](https://djangoalevel.s3.eu-central-1.amazonaws.com/lesson47/ec2-ip.png)

## git clone и виртуальное окружение

Клонируем код нашего проекта:

```git clone https://github.com/your-git/your-repo.git```

Создаём виртуальное окружение, я обычно создаю отдельную папку рабочей папке:

```
cd ~/
mkdir venv
python3 -m venv venv/
```

И активируем его:

```source ~/venv/bin/activate```

Переходим в раздел с проектом и устанавливаем всё что есть в `requirements.txt`:

```
cd ~/proj_name
pip install -r requirements.txt
```

Если локально вы не работали с базой постгрес, то необходимо доставить модуль для работы с ним:

```pip install psycopg2-binary```

После чего вы должны успешно применить миграции:

```python manage.py migrate```

## Проверяем работоспособность сервера

Для проверки что ваше приложение можно разворачивать запустим его через стандартный рансервер, но для того, что бы это
сработало необходимо разрешить использовать порт который мы будем использовать для теста, по стандарту это порт номер
8000:

```sudo ufw allow 8000```

Мы открыли порт со стороны сервера, но пока что он закрыт со стороны амазона, давайте временно откроем его тоже, для
этого идём на страницу амазона с описанием инстанса открываем вкладку `Security` и кликаем на название секьюрити группы:

![](https://djangoalevel.s3.eu-central-1.amazonaws.com/lesson47/Security-group.png)

Кликаем на `Edit inbound rules`

Добавляем правило `Custom TCP` для порта 8000:

![](https://djangoalevel.s3.eu-central-1.amazonaws.com/lesson47/edit-rules.png)

После этого можно запустить рансервер с такими правилами:

```python manage.py runserver 0.0.0.0:8000```

Если вы всё сделали правильно, то вы теперь можете открыть в браузере IP адрес который вам выдал амазон, с портом 8000:

```
# Например
http://54.186.155.252:8000/
```

Обратите внимание, сайт откроется БЕЗ СТАТИКИ, потому что рансервер при дебаге фолс, и не должен сёрвить статику.

## Проверяем gunicorn

Устанавливаем gunicorn

```pip install gunicorn```

Как вы помните gunicorn это wsgi сервер, если вы откроете папку с `settings.py`, то вы увидите там еще два
файла `wsgi.py` и `asgi.py`

Они нужны для того что бы запускать сервера в "боевом" режиме

Для проверки работы gunicorn запустим сервер через него:

```gunicorn --bind 0.0.0.0: 8000 Proj.wsgi```

Где `Proj` - это название папки в котором лежит файл `wsgi.py`

Опять же если вы всё сделали правильно, то тоже самый урл всё ещё будет работать

```
# Например
http://54.186.155.252:8000/
```

## Понятие сокет-файла

В линуксе, абсолютно всё это файл. Развёрнутый сервер, это тоже файл. Так вот, если мы используем два сервера, и один
слушает второй, то давайте разворачивать первый тоже как файл. Такой файл будет называться сокет-файлом.

## Демонизация gunicorn

Запускать сервер руками это очень увлекательно, но не очень эффективно, давайте демонизируем gunicorn для запуска
сервера в сокет-файл, и сделаем что бы этот сервер запускался сразу при запуске системы, что бы даже если мы
перезагрузим инстанс, сервер всё равно работал.

Воспользуемся встроенной в linux системой systemd (системная демонизация)

Создадим системный файл для описания сокета

```sudo nano /etc/systemd/system/gunicorn.socket```

```
#/etc/systemd/system/gunicorn.socket

[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/home/ubuntu/proj/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

Блок `Unit` отвечает за описание демона

Блок `Socket` отвечает за то, где будет находиться файл сокета, `proj` в данном случае название папки с проектом, `run`
название папки с файлом сокета (так почему-то принято называть)

Блок `Install` отвечает за автоматический запуск при запуске системы

Сохраните файл и закройте его

Теперь нужно создать файл сервиса, который и будет выполнять запуск:

```sudo nano /etc/systemd/system/gunicorn.service```

Названия сервиса и сокета должны совпадать

```
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/myprojectdir
EnvironmentFile=/home/ubuntu/.env
ExecStart=/home/ubuntu/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/home/ubuntu/myprojectdir/run/gunicorn.sock \
          myproject.wsgi:application

[Install]
WantedBy=multi-user.target
```

Блок `Unit`

- Description - описание
- Requires - связывает сервис с сокетом
- After - отвечает за то, что бы запускать модуль, только после того как будет доступ в интернет.

Блок `Service`

- User - пользователь который должен запускать скрипт, если мы используем стандартного юзера, то это будет `ubuntu`

- Group - Группа безопасности, по дефолту это `www-data`

- WorkingDirectory - папка из которой будет запускаться скрипт, в нашем случае это папка с проектом, там где
  лежит `manage.py`

- EnvironmentFile - файл с переменными
  
- ExecStart - сам скрипт, нам нужно запустить gunicorn из виртуального окружения, но мы не можем запустить
  сначала `source`, но при создании виртуального окружения, мы всего-то складываем все скрипты в другую папку

  `/home/ubuntu/venv/bin/gunicorn` Это физическое расположение скрипта
  `--access-logfile - --workers 3 --bind unix:/home/ubuntu/myprojectdir/run/gunicorn.sock myproject.wsgi:application`
  Это настройки самого скрипта, лог левел, это куда писать логи, воркер, это их колличество, бинд это куда сложить
  файл (unix: значит что будет файл), myproject - название папки где лежит `wsgi.py`

Блок `Install` отвечает за автоматический запуск при запуске системы для любого пользователя

Сохраняем файл и закрываем

Для запуска сокета нужно запустить его из системы:

```
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

В следующий раз этого делать не нужно, всё запустится автоматически.

### Проверим запуск сокета

```sudo systemctl status gunicorn.socket```

Если тут мы не видим никаких ошибок, то нужно проверить наличие файла сокета, если всё есть, то создание сокета
работает.

Проверим его активацию:

```sudo systemctl status gunicorn```

Должны увидеть примерно такой статус:

```
gunicorn.service - gunicorn daemon
   Loaded: loaded (/etc/systemd/system/gunicorn.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
```

Попробуем выполнить запрос к нашему сокету:

```curl --unix-socket /home/ubuntu/proj/run/gunicorn.sock localhost```

И проверим статус еще раз, теперь он будет `active`

### Рестарт юникорна

Теперь мы можем перезапускать наш сокет за 1 команду

```sudo systemctl restart gunicorn```

## Nginx

Nginx это веб сервер, гибкий и мощный веб-сервер.

Для проверки работы nginx давайте настроим базовый доступ к нжинксу для нашего IP адреса, и не забываем добавить 80 порт
в секьюрити группы амазона.

Настроим nginx, если вы установили nginx (мы сделали это первым действием на этом инстансе), то у вас будет
существовать папки с базовыми настройками nginx, давайте создадим новую настройку:

```sudo nano /etc/nginx/sites-available/myproject```

Где `myproject` это название вашего проекта

```
server {
    listen 80;
    server_name 54.186.155.252;
}
```

Сохранить и закрыть, пробросить симлинк в соседнюю папку которую по дефолту сервит nginx

```sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/```

ОБЯЗАТЕЛЬНО! Добавить 80 порт в разрешенные на амазоне в секьюрити группах!!

Если вы всё сделали правильно, то по вашему IP адресу без указания порта будет открыта базовая страница nginx:

![](https://djangoalevel.s3.eu-central-1.amazonaws.com/lesson47/nginx_welcome.png)

Для того, что бы nging начал проксировать наш проект нужно его указать:

```sudo nano /etc/nginx/sites-available/myproject```

```
server {
    listen 80;
    server_name some_IP_or_url;
    location / {
        include proxy_params;
        proxy_pass http://unix:/home/ubuntu/path/to/my/socket/gunicorn.sock;
    }
}


```

Перезапускаем nginx и проверяем

```sudo systemctl restart nginx```

Всё должно работать, но без статики

Вспомним самое начало лекции, что мы можем придумать куда команда `collectstatic` должна сложить статику. Запускаем команду и указываем nginx что нужно обрабатывать статику и медиа.

```
server {
    listen 80;
    server_name 34.221.249.152;

    location /static/ {
        root /home/ubuntu/deployment;
    }
    location /media/ {
        root /home/ubuntu/deployment;
    }
    location / {
        include proxy_params;
        proxy_pass http://unix:/home/ubuntu/path/to/my/socket/gunicorn.sock;
    }
}

```

Рестартуем нжинкс и наслаждаемся результатом!

Не забываем закрыть 8000 порт, и на инстансе и на амазоне в секьюрити группе!!

```sudo ufw delete allow 8000```