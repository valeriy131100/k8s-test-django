# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).


## Как запустить dev-версию в Kubernetes
Установите [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/) и [minikube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/).
Также подготовьте предварительно запущенную базу данных.

Подготовьте minikube:
```bash
minikube start
minikube addons enable ingress
```

Соберите образ django-бэкенда:
```bash
eval $(minikube docker-env) # для Linux
minikube docker-env | Invoke-Expression # для Windows

cd backend_main_django
docker build -t django_app:{укажите версию} .
docker tag django_app:{указанная версия} django_app:latest
```

Подготовьте окружение - создайте ConfigMap "django-conf" с переменными среды указанными ниже. Параметр `ALLOWED_HOSTS` указывать не нужно, он уже будет заполнен тестовым доменом - `starburger.test`. 

Простейший способ создать ConfigMap - создать env-файл с переменными среды в формате `КЛЮЧ=ЗНАЧЕНИЕ`, а затем исполнить:
```bash
kubectl create configmap django-conf --from-env-file={ваш env-файл}
```

Если вам понадобится отредактировать созданный ConfigMap, то используйте:
```bash
kubectl edit configmap django-conf
```

Накатите деплой:
```bash
kubectl apply -f deploy.yml
```

Примените миграции базы данных:
```bash
kubectl create -f migrate.yml
```

Если это необходимо, то запустите еженедельную очистку сессий:
```bash
kubectl apply -f clearsessions.yml
```

Если вы запускаете minikube с помощью Docker и используете туннель для подключения, то добавьте следующее  в hosts-файл вашей системы (`/etc/hosts` на Linux и `C:\Windows\System32\drivers\etc` на Windows):
```
127.0.0.1 starburger.test
```

Иначе добавьте вывод следующей команды
```bash
echo "$(minikube ip) starburger.test"
```

Сервер будет доступен по адресу [http://starburger.test](http://starburger.test).


## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).
