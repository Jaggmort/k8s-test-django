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

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

# Запуск сайта в Kubernetes

## Скачиваем проект

Переходим в будущее место расположения проекта и потом скачиваем его:

`git clone (https://github.com/Jaggmort/k8s-test-django.git)`


## Переменные окружения

Для работы необходимо будет создать файл `secret.yaml` с следующим содержимым:

`
apiVersion: v1 \
kind: Secret \
metadata: \
  name: django-app-secret \
type: Opaque \
stringData: \
    SECRET_KEY: "Тут введите SECRET_KEY" \
    DATABASE_URL: "Введите DATABASE_URL" \
`

Где переменные должны быть введены закодированными в формате base64:
`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Запускаем сайт в Kubernetes

Переходим в каталог сайта:

`cd k8s-test-django\backend_main_django`

Создаём образ Django-приложения в minikube:

`minikube image build -t django-app .`

Применяем secret.yaml:

`kubectl apply -f secret.yaml`

Запускаем сайт в kubernetes:

`cd k8s-test-django\kuber`

`kubectl apply -f django-app.yaml`

`kubectl apply -f django-app-balancer.yaml`

Чтобы получить доступ к сайту необходимо включить ingress:

`minikube addons enable ingress`

`kubectl apply -f django-app-ingress.yaml`

Проверить работу сайта без домена можно следующим образом:

`minikube ip`

Добавить следующую строчку в hosts (etc/hosts - Linux, c:\Windows\system32\drivers\etc\hosts):

`"minikube ip" star-burger.info`

Зайти на сайт по адресу: `star-burger.info`

Чтобы примернить миграции создаем следующию задачу(job):

`kubectl apply -f django-migrate.yaml`

Для автоматического удаления сессий используем сделующую запланированную задчу (cronjob):

`kubectl apply -f clear-sessions-cronjob.yaml`
