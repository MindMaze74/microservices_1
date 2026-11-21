
# Домашнее задание к занятию «Микросервисы: принципы» - Старцев Данила Антонович

Вы работаете в крупной компании, которая строит систему на основе микросервисной архитектуры.
Вам как DevOps-специалисту необходимо выдвинуть предложение по организации инфраструктуры для разработки и эксплуатации.

## Задача 1: API Gateway 

Предложите решение для обеспечения реализации API Gateway. Составьте сравнительную таблицу возможностей различных программных решений. На основе таблицы сделайте выбор решения.

Решение должно соответствовать следующим требованиям:
- маршрутизация запросов к нужному сервису на основе конфигурации,
- возможность проверки аутентификационной информации в запросах,
- обеспечение терминации HTTPS.

Обоснуйте свой выбор.

Для API Gateway я посмотрел несколько вариантов, которые чаще всего встречаются в микросервисных проектах: NGINX, Kong, Traefik, KrakenD и Tyk. Сравнивал по трём обязательным требованиям из задания: маршрутизация по конфигурации, проверка аутентификации в запросах и терминация HTTPS. Дополнительно смотрел на сложность эксплуатации и производительность, потому что в реальной работе шлюз придётся не только запустить, но и сопровождать.

NGINX это зрелый и очень быстрый reverse proxy. Маршрутизация задаётся в nginx.conf через location и proxy_pass, то есть всё прозрачно и предсказуемо. Проверку аутентификации можно сделать через auth_request: NGINX сам сходит внутренним запросом в сервис аутентификации и пропустит запрос дальше только при успешном ответе. HTTPS терминируется штатно через ssl_certificate и ssl_certificate_key. Внешняя база данных не нужна, всё запускается одним контейнером. По сути, это самый простой вариант, если не хочется тащить в инфраструктуру лишние компоненты.

Kong построен на базе OpenResty, то есть внутри там NGINX плюс LuaJIT. У него богатая экосистема плагинов: JWT, OAuth2, Key Auth, mTLS и многое другое. Маршруты настраиваются через Admin API, а конфигурация хранится в PostgreSQL или Cassandra. HTTPS поддерживается. Kong хорошо подходит крупным компаниям, которым нужны готовые политики безопасности и enterprise-поддержка. Но для небольшой инфраструктуры он тяжеловат: появляется база данных, миграции, отдельное обслуживание. Это уже заметная операционная нагрузка.

Traefik это cloud-native reverse proxy и ingress controller, который хорошо дружит с Docker и Kubernetes. Он умеет сам обнаруживать сервисы и динамически обновлять маршруты. Аутентификация делается через middleware, например ForwardAuth или BasicAuth. HTTPS тоже поддерживается, включая автоматическое получение сертификатов Let’s Encrypt. Traefik удобен в динамической контейнерной среде, но в задаче с фиксированным набором сервисов его главное преимущество не так важно. Настройка аутентификации через внешние middleware тоже не всегда выглядит проще, чем обычный auth_request.

KrakenD это stateless API Gateway на Go с декларативным JSON-конфигом. Он очень быстрый, не требует базы данных и умеет работать с JWT и OAuth2. HTTPS терминируется штатно. Но экосистема плагинов и готовых интеграций у него уже, чем у Kong или Tyk. Конфигурация больше ориентирована на агрегацию API, чем на классический reverse proxy с богатой логикой проверки запросов.

Tyk это уже полноценная платформа API management: dashboard, политики безопасности, аналитика, поддержка JWT и OAuth2. HTTPS и маршрутизация там есть. Но Tyk это тяжёлый стек из нескольких компонентов. Для трёх сервисов из задания он избыточен, и сопровождать его будет заметно сложнее.

В итоге я выбираю NGINX. Он полностью закрывает все обязательные требования. Маршрутизация задаётся конфигом, аутентификация делается через auth_request с вызовом сервиса security, HTTPS терминируется штатно. При этом NGINX не требует внешней БД и сложных миграций, как Kong или Tyk. Один контейнер и один конфигурационный файл, это удобно и для локального запуска, и для дальнейшей эксплуатации. По производительности NGINX тоже вполне достаточен для описанной инфраструктуры. Получается хороший баланс: всё нужное есть, лишней сложности нет.

## Задача 2: Брокер сообщений

Составьте таблицу возможностей различных брокеров сообщений. На основе таблицы сделайте обоснованный выбор решения.

Решение должно соответствовать следующим требованиям:
- поддержка кластеризации для обеспечения надёжности,
- хранение сообщений на диске в процессе доставки,
- высокая скорость работы,
- поддержка различных форматов сообщений,
- разделение прав доступа к различным потокам сообщений,
- простота эксплуатации.

Обоснуйте свой выбор.


Для брокера я смотрел на четыре варианта, которые обычно обсуждают в контексте микросервисов: RabbitMQ, Apache Kafka, Redis Streams и NATS JetStream. Сравнивал по требованиям из задания: кластеризация, хранение сообщений на диске, скорость, разные форматы сообщений, разграничение прав доступа и простота эксплуатации. Последний пункт для DevOps-специалиста часто оказывается решающим, потому что брокер придётся не только запустить, но и годами сопровождать.

1. RabbitMQ это классический брокер очередей с богатой моделью маршрутизации. Кластеризация поддерживается, причём современные quorum-очереди дают репликацию и устойчивость к отказам узлов. Сообщения можно хранить на диске: есть persistent-режим, а quorum-очереди рассчитаны на надёжное хранение с опорой на локальные SSD или NVMe. По скорости RabbitMQ уступает лидерам: в тестах он показывал порядка 31 тысячи сообщений в секунду с P99 около 125 мс, тогда как Kafka и NATS выдают заметно больше. Но для типовой микросервисной системы, где нет задачи обрабатывать миллионы событий в секунду, этой производительности обычно достаточно. С форматами RabbitMQ гибок: AMQP, MQTT, STOMP, плюс бинарные payload внутри. Права доступа устроены через vhosts и permissions, то есть можно изолировать окружения и ограничивать пользователей по очередям и exchange’ам. По эксплуатации RabbitMQ находится где-то посередине: сложнее, чем NATS или Redis, но проще, чем Kafka. Нужно следить за памятью, дисковыми порогами и настройкой quorum-очередей.

2. Apache Kafka это распределённый лог, заточенный под очень высокий throughput. В тестах он выдавал около 127 тысяч сообщений в секунду с P99 порядка 42 мс, то есть по скорости это один из лидеров. Кластеризация и репликация партиций есть, хранение на диске это его базовая модель. Форматы сообщений в основном бинарные, схемы обычно навешивают через Schema Registry. Права доступа реализуются через ACL на топики и группы консьюмеров, аутентификация через SASL или mTLS. Главный минус Kafka в том, что она сложна в эксплуатации. Исторически требовался ZooKeeper, сейчас есть KRaft, но управление партициями, ребалансировкой и оффсетами всё равно требует квалификации и времени. Для инфраструктуры, где нет потока событий на миллионы в секунду, эта сложность часто не оправдана.

3. Redis Streams выглядит привлекательно за счёт низкой задержки и простоты, если Redis в проекте уже есть. По скорости Streams дают около 69 тысяч сообщений в секунду с P99 порядка 27 мс, что быстрее RabbitMQ. Но с хранением и кластеризацией есть нюансы. Redis в основе своей in-memory, а Streams хоть и поддерживают персистентность, но она ограничена моделью памяти и требует ручного управления длиной стрима через XDEL или MAXLEN, иначе память растёт. Кластеризация Redis Cluster существует, но для сценариев очередей с гарантированной доставкой это не всегда удобно. Права доступа через ACL появились в Redis 6, но модель прав тоньше, чем в специализированных брокерах. В итоге Redis Streams хорошо подходят для лёгких сценариев или когда Redis уже является частью стека, но как основной брокер для ответственной микросервисной шины они дают меньше гарантий и удобства.

4. NATS JetStream это современный вариант с очень хорошим балансом скорости и простоты. В тестах JetStream показывал около 83 тысяч сообщений в секунду с P99 всего 18 мс, то есть по latency он лучше Kafka и RabbitMQ. Кластеризация встроена: JetStream использует Raft для репликации состояния стримов и консьюмеров, при этом остаётся «shared-nothing» архитектура. Хранение на диске включается через JetStream, Core NATS сам по себе сообщения не сохраняет. Права доступа реализуются через accounts, users и NKeys/JWT, то есть модель безопасности есть, но она требует аккуратной настройки, особенно при переходе в production. По эксплуатации NATS считается одним из самых простых: один бинарник на Go, минимальная конфигурация. Однако у JetStream есть свои тонкости с квотами на файловое хранилище и политиками retention, которые нужно осознанно настраивать.

Я выбираю RabbitMQ, потому что он лучше всего закрывает все требования из задания без экстремального усложнения эксплуатации.

Кластеризация и хранение на диске у RabbitMQ есть в зрелом виде через quorum-очереди. Скорость ниже, чем у Kafka или NATS, но для большинства микросервисных сценариев её достаточно, а взамен вы получаете предсказуемое поведение и богатую маршрутизацию. Форматы сообщений поддерживаются разные, включая AMQP и MQTT, что удобно, если в системе есть разнородные клиенты. Права доступа через vhosts и permissions дают понятную и проверенную модель разграничения, без необходимости строить отдельную систему авторизации.

Против Kafka я выбираю RabbitMQ из-за сложности эксплуатации. Kafka быстрее и лучше подходит для event streaming, но управление партициями, ребалансировкой и всей обвязкой требует заметно больше времени и квалификации. Против NATS JetStream RabbitMQ выигрывает в зрелости модели прав доступа и предсказуемости очередей, хотя NATS быстрее и проще в базовом развёртывании. Redis Streams я не беру как основной брокер, потому что при всех своих плюсах по скорости и простоте они дают меньше гарантий по хранению и кластеризации в сценариях ответственной доставки.

В сухом остатке RabbitMQ это не самый быстрый и не самый простой вариант в абсолюте, но самый сбалансированный под требования задания. Он даёт кластеризацию, дисковое хранение, разумную скорость, гибкие форматы и понятные права доступа, а сложность его эксплуатации ниже, чем у Kafka, при более зрелой модели безопасности, чем у Redis Streams или NATS JetStream.




## Задача 3: API Gateway * (необязательная)

### Есть три сервиса:

**minio**
- хранит загруженные файлы в бакете images,
- S3 протокол,

**uploader**
- принимает файл, если картинка сжимает и загружает его в minio,
- POST /v1/upload,

**security**
- регистрация пользователя POST /v1/user,
- получение информации о пользователе GET /v1/user,
- логин пользователя POST /v1/token,
- проверка токена GET /v1/token/validation.

### Необходимо воспользоваться любым балансировщиком и сделать API Gateway:

**POST /v1/register**
1. Анонимный доступ.
2. Запрос направляется в сервис security POST /v1/user.

**POST /v1/token**
1. Анонимный доступ.
2. Запрос направляется в сервис security POST /v1/token.

**GET /v1/user**
1. Проверка токена. Токен ожидается в заголовке Authorization. Токен проверяется через вызов сервиса security GET /v1/token/validation/.
2. Запрос направляется в сервис security GET /v1/user.

**POST /v1/upload**
1. Проверка токена. Токен ожидается в заголовке Authorization. Токен проверяется через вызов сервиса security GET /v1/token/validation/.
2. Запрос направляется в сервис uploader POST /v1/upload.

**GET /v1/user/{image}**
1. Проверка токена. Токен ожидается в заголовке Authorization. Токен проверяется через вызов сервиса security GET /v1/token/validation/.
2. Запрос направляется в сервис minio GET /images/{image}.

### Ожидаемый результат

Результатом выполнения задачи должен быть docker compose файл, запустив который можно локально выполнить следующие команды с успешным результатом.
Предполагается, что для реализации API Gateway будет написан конфиг для NGinx или другого балансировщика нагрузки, который будет запущен как сервис через docker-compose и будет обеспечивать балансировку и проверку аутентификации входящих запросов.
Авторизация
curl -X POST -H 'Content-Type: application/json' -d '{"login":"bob", "password":"qwe123"}' http://localhost/token

**Загрузка файла**

curl -X POST -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJib2IifQ.hiMVLmssoTsy1MqbmIoviDeFPvo-nCd92d4UFiN2O2I' -H 'Content-Type: octet/stream' --data-binary @yourfilename.jpg http://localhost/upload

**Получение файла**
curl -X GET http://localhost/images/4e6df220-295e-4231-82bc-45e4b1484430.jpg

### Ответ:
docker-compose.yaml. Официальные образы MinIO удалили из Docker Hub, а quay.io, openebs и Chainguard перестали отдавать их анонимно. Я заменил их на форк Silo (pgsty/silo, pgsty/mc), он совместим с S3 и теми же переменными. Заодно поменял устаревшую команду mc policy set на mc anonymous set, добавил сеть minio-net, пробросил порты 9000 и 9001 и убрал строку version, чтобы не было предупреждения.

requirements.txt для security. Старый Flask пытался импортировать escape из Jinja2, а его там больше нет. Я обновил Flask до версии не ниже 2.1.0, убрал дубликат PyJWT, поднял prometheus-flask-exporter.

gateway/nginx.conf. Убрал строку-заглушку TODO[MT]:, из-за которой nginx вообще не стартовал.

gateway/default.conf. Добавил internal и передачу заголовка Authorization в блок валидации, иначе auth_request ходил бы в security без токена и отклонял все запросы. Добавил location для /v1/token, /v1/register, /v1/user, /v1/upload и /v1/user/{image}, потому что в исходнике их не было и запросы уходили в 404. Для файлов из Silo заменил proxy_pass с URI на rewrite плюс proxy_pass без URI: так Nginx резолвит storage при старте, а не в рантайме, и 502 с ошибкой no resolver defined ушёл.

security/server.py. Дописал POST /v1/user для регистрации и GET /v1/user для информации о пользователе, их не было. Хранилище data переименовал в users, чтобы регистрация и логин работали с одним словарём. И самое важное: /v1/token/validation при невалидном токене возвращал строку с кодом 200, а auth_request считает любой 2xx успехом. Поменял на 401, теперь проверка токена реально работает. /v1/token стал возвращать JSON {"token": "..."} вместо голой строки.

Тестовый файл. test.jpg, созданный через echo, был текстом, и uploader отвечал Failed to detect file mime type. Я заменил его на настоящий JPEG из base64. Побайтное сравнение с исходником при скачивании не нужно: uploader по заданию сжимает картинку.

После всех правок docker-compose up поднимает всё без ошибок, а команды из раздела «Ожидаемый результат» выполняются успешно.

![скриншот 1](https://github.com/MindMaze74/microservices_1/blob/main/img/1.png)

![скриншот 2](https://github.com/MindMaze74/microservices_1/blob/main/img/2.png)

![скриншот 3](https://github.com/MindMaze74/microservices_1/blob/main/img/3.png)

![скриншот 4](https://github.com/MindMaze74/microservices_1/blob/main/img/4.png)

![скриншот 5](https://github.com/MindMaze74/microservices_1/blob/main/img/5.png)

![скриншот 6](https://github.com/MindMaze74/microservices_1/blob/main/img/6.png)

![скриншот 7](https://github.com/MindMaze74/microservices_1/blob/main/img/7.png)

 
Листинг команд
<details>
  <summary>Нажмите, чтобы увидеть Листинг команд</summary>

```bash
# Первоначальный запуск
docker-compose up --build

# Проверка статуса контейнеров
docker-compose ps
docker compose ps

# Просмотр логов отдельных сервисов
docker-compose logs gateway
docker-compose logs security
docker-compose logs uploader
docker-compose logs storage

# Перезапуск отдельных сервисов
docker-compose restart gateway
docker-compose restart security

# Полная остановка и пересборка
docker-compose down
docker-compose build --no-cache security
docker-compose up -d
docker-compose up

# Просмотр исходника сервиса
docker-compose exec security cat server.py

# Поиск объявленных маршрутов
docker-compose exec security grep -nE "@app|@application|route|Flask\(" server.py

# Просмотр содержимого рабочей директории
docker-compose exec security ls -la /app
docker-compose exec security ls -la /app/src

# Попытка импорта приложения (неудачная — приложение называется server, а не app)
docker-compose exec security python -c "from server import app; print(app.url_map)"

# Проверка health-эндпоинта Silo из контейнера gateway
docker-compose exec gateway wget -qO- http://storage:9000/minio/health/live

# Прямой запрос файла из Silo (минуя Nginx)
docker-compose exec gateway wget -O- http://storage:9000/images/da0d4b4b-6431-433e-b96b-79a31b98a557.jpg

# Скачивание файла напрямую из контейнера storage
docker-compose exec storage cat /data/images/da0d4b4b-6431-433e-b96b-79a31b98a557.jpg > stored.jpg

# Просмотр логов ошибок Nginx
docker-compose exec gateway cat /var/log/nginx/api.error.log

# Проверка содержимого бакета через mc
docker-compose run --rm createbuckets /bin/sh -c "
mc alias set storage http://storage:9000 STORAGE_ACCESS_KEY STORAGE_SECRET_KEY &&
mc ls storage/images
"

# Просмотр текущих конфигов
cat gateway/default.conf
cat gateway/nginx.conf

# Проверка синтаксиса конфигурации (при необходимости)
docker-compose exec gateway nginx -t

curl -i -X POST -H 'Content-Type: application/json' \
  -d '{"login":"bob", "password":"qwe123"}' \
  http://localhost/v1/register

curl -i -X POST -H 'Content-Type: application/json' \
  -d '{"login":"alice", "password":"qwe123"}' \
  http://localhost/v1/register

  # Через /v1/token
curl -i -X POST -H 'Content-Type: application/json' \
  -d '{"login":"bob", "password":"qwe123"}' \
  http://localhost/v1/token

# Через /token (как в задании)
curl -i -X POST -H 'Content-Type: application/json' \
  -d '{"login":"bob", "password":"qwe123"}' \
  http://localhost/token

# Сохранение токена в переменную
TOKEN=$(curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"login":"bob", "password":"qwe123"}' \
  http://localhost/v1/token | python3 -c "import sys,json;print(json.load(sys.stdin)['token'])")
echo $TOKEN


# С валидным токеном — 200 OK
curl -i -X GET -H "Authorization: Bearer $TOKEN" http://localhost/v1/user

# Без токена — 401 Unauthorized
curl -i -X GET http://localhost/v1/user

# С невалидным токеном — 401 Unauthorized
curl -i -X GET -H "Authorization: Bearer invalid.token.here" http://localhost/v1/user

# С валидным токеном — 200 OK
curl -i -X GET -H "Authorization: Bearer $TOKEN" http://localhost/v1/user

# Без токена — 401 Unauthorized
curl -i -X GET http://localhost/v1/user

# С невалидным токеном — 401 Unauthorized
curl -i -X GET -H "Authorization: Bearer invalid.token.here" http://localhost/v1/user

# Первая попытка (оказалась текстовым файлом, а не JPEG)
echo "test image content" > test.jpg

# Создание настоящего JPEG через base64
echo "/9j/4AAQSkZJRgABAQEAYABgAAD/2wBDAAgGBgcGBQgHBwcJCQgKDBQNDAsLDBkSEw8UHRofHh0aHBwgJC4nICIsIxwcKDcpLDAxNDQ0Hyc5PTgyPC4zNDL/2wBDAQkJCQwLDBgNDRgyIRwhMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjL/wAARCAABAAEDASIAAhEBAxEB/8QAFQABAQAAAAAAAAAAAAAAAAAAAAv/xAAUEAEAAAAAAAAAAAAAAAAAAAAA/8QAFQEBAQAAAAAAAAAAAAAAAAAAAAX/xAAUEQEAAAAAAAAAAAAAAAAAAAAA/9oADAMBAAIRAxEAPwCwAA8A/9k=" | base64 -d > test.jpg

# Проверка формата
file test.jpg


curl -i -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: octet/stream' \
  --data-binary @test.jpg \
  http://localhost/v1/upload

# Как в задании (через /upload)
curl -i -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: octet/stream' \
  --data-binary @test.jpg \
  http://localhost/upload

  # Защищённый вариант через /v1/user/{image}
curl -i -X GET -H "Authorization: Bearer $TOKEN" \
  -o downloaded.jpg \
  http://localhost/v1/user/da0d4b4b-6431-433e-b96b-79a31b98a557.jpg

# Публичный вариант через /images/{image} (как в задании)
curl -i -X GET \
  -o downloaded.jpg \
  http://localhost/images/da0d4b4b-6431-433e-b96b-79a31b98a557.jpg

# Чистое скачивание без заголовков в файле (-s вместо -i)
curl -s -X GET -H "Authorization: Bearer $TOKEN" \
  -o downloaded.jpg \
  http://localhost/v1/user/da0d4b4b-6431-433e-b96b-79a31b98a557.jpg

curl -s -X GET \
  -o downloaded.jpg \
  http://localhost/images/da0d4b4b-6431-433e-b96b-79a31b98a557.jpg

  # Определение типа файла
file downloaded.jpg

# Просмотр первых байтов (магические числа)
xxd downloaded.jpg | head -5
xxd downloaded.jpg | head -2

# Сравнение с исходным (не показательно — uploader сжимает)
cmp test.jpg downloaded.jpg && echo "Files are identical"

# Размеры файлов
ls -l test.jpg downloaded.jpg

# Через ImageMagick (если установлен)
identify downloaded.jpg

# Просмотр содержимого скачанного файла (если это HTML-ошибка)
cat downloaded.jpg

# 1. Статус всех контейнеров
docker-compose ps

# 2. Регистрация нового пользователя
curl -i -X POST -H 'Content-Type: application/json' \
  -d '{"login":"alice", "password":"qwe123"}' \
  http://localhost/v1/register

# 3. Получение токена
curl -i -X POST -H 'Content-Type: application/json' \
  -d '{"login":"bob", "password":"qwe123"}' \
  http://localhost/token

# 4. Загрузка файла
curl -i -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: octet/stream' \
  --data-binary @test.jpg \
  http://localhost/upload

# 5. Получение файла
curl -i -X GET \
  -o downloaded.jpg \
  http://localhost/images/<FILENAME_ИЗ_ШАГА_4>
file downloaded.jpg

# 6. Проверка работы auth_request
curl -i -X GET http://localhost/v1/user
curl -i -X GET -H "Authorization: Bearer invalid.token" http://localhost/v1/user
```
</details>



---

#### [Дополнительные материалы: как запускать, как тестировать, как проверить](https://github.com/netology-code/devkub-homeworks/tree/main/11-microservices-02-principles)

---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---