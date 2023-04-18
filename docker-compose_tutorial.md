# Учебник

## Nginx

В папке создаем `docker-compose.yml`. 

в него вписываем:

```
version: '3.0'

# Services
services:

  # Nginx Service
  nginx:
    image: nginx:1.21
    ports:
      - 80:80
```

Запуск Docker-compose: 
`docker compose up -d`

Просмотр контейнеров и служб:
`docker compose ps`

остановки контейнеров:
`docker compose stop`

---

## PHP

```
version: '3.8'

# Services
services:

  # Nginx Service
  nginx:
    image: nginx:1.21
    ports:
      - 80:80
    volumes:
      - ./src:/var/www/php
      - ./docker/nginx/conf.d:/etc/nginx/conf.d
    depends_on:
      - php

  # PHP Service
  php:
    image: php:8.1-fpm
    working_dir: /var/www/php
    volumes:
      - ./src:/var/www/php
```

`depends_on: php` зависимость контейнера `nginx` от контейнера `php`.

`volumes:` - конфиг и файлы приложения

### конфиг для Nginx

`.docker/nginx/conf.d/php.conf`

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root   /var/www/php;
    index  index.php;

    location ~* \.php$ {
        fastcgi_pass   php:9000;
        include        fastcgi_params;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param  SCRIPT_NAME     $fastcgi_script_name;
    }
}
```

**Как рабтает**
Когда сервер получает запрос на порт 80 `listen 80`, то он отправляет запрост в папку **root** ` root /var/www/php;`. Там он ищет **индексный** файл `index index.php index.html;`. Когда он там находит этот файл, то смотрит что делать с эти файлом в инструкции **location**: `location ~ \.php$` - что делать с файлами *.php. 

`fastcgi_pass app:9000;` - Nginx перенаправлять запросы к файлам PHP на порт 9000 контейнера PHP. 
> Т.е. у нас есть каонтейнер nginx и php. Nginx перенаправляет запросы всехфайлов php в контенйр с php, где php из парсит и выводит. Так как сам по себе Nginx не поимает php.

### фаловая структура 

```
docker-tutorial/
├── docker/
│   └── nginx/
│       └── conf.d/
│           └── php.conf
├── src/
│   └── index.php
└── docker-compose.yml
```

## PHP

Проблема в том, что nginx и mysql прекрасно работают из коробки, а с php иначе. Он тоже работает, но в официальный образ не включены никакие расширения, которые могут понадобиться при работе. Именно поэтому мы будем собирать php через Dockerfile. 

```
FROM php:7.2-fpm

RUN apt-get update && apt-get install -y \
        curl \
        wget \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
    && pecl install mcrypt-1.0.1 \
    && docker-php-ext-install -j$(nproc) iconv mbstring mysqli pdo_mysql zip \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install -j$(nproc) gd \
    && docker-php-ext-enable mcrypt

# настройки для PHP, можно подключить в VOLUME compose
ADD php.ini /usr/local/etc/php/conf.d/40-custom.ini

WORKDIR /var/www

EXPOSE 9000

CMD ["php-fpm"]
```

Теперь `docker-compose` будет выглядеть так:

```
version: '3.0'

# Services
services:

  # Nginx Service
  nginx:
    image: nginx:1.21
    ports:
      - 80:80
    volumes:
      - ./src:/var/www/php
      - ./docker/nginx/conf.d:/etc/nginx/conf.d
    depends_on:
      - php

  # PHP Service
  php:
    #image: php:8.1-fpm
    build: ./docker/php
    working_dir: /var/www/php
    volumes:
      - ./docker/php/php.ini:/usr/local/etc/php/php.ini
      - ./docker/php/php.ini:/usr/local/etc/php/conf.d/local.ini
      - ./src:/var/www/php
```

B `docker/php/php.ini` прописать `error_reporting = E_ALL` - отображение всех ошибок


## MySQL

`docker-compose`:

```
# PHP Service
php:
  build: ./docker/php
  working_dir: /var/www/php
  volumes:
    - ./docker/php/php.ini:/usr/local/etc/php/php.ini
    - ./docker/php/php.ini:/usr/local/etc/php/conf.d/local.ini
    - ./src:/var/www/php
  depends_on:
    mysql:
      condition: service_healthy

# MySQL Service
mysql:
  image: mariadb:10.7.8
  environment:
    MYSQL_ROOT_PASSWORD: root
    MYSQL_DATABASE: demo
  volumes:
    - ./docker/mysql/my.cnf:/etc/mysql/conf.d/my.cnf
    - mysqldata:/var/lib/mysql
  healthcheck:
    test: mysqladmin ping -h 127.0.0.1 -u root --password=$$MYSQL_ROOT_PASSWORD
    interval: 5s
    retries: 10

# Volumes
volumes:

  mysqldata:
```

В `./docker/mysql/my.cnf` прописать: 

```
[mysqld]
collation-server     = utf8mb4_unicode_ci
character-set-server = utf8mb4
```


каждый раз, когда контейнер сервиса mysql уничтожается, база данных уничтожается вместе с ним. Чтобы сделать её постоянной, мы просто говорим контейнеру MySQL использовать том `mysqldata` для локального хранения данных.  В результате к контейнеру монтируется локальный каталог, с той лишь разницей, что вместо того, чтобы указывать, какой именно, мы позволяем Docker Compose выбрать место.

`healthcheck`. Она позволяет нам указать, при каком условии контейнер готов, а не просто запущен. В данном случае недостаточно запустить контейнер MySQL — мы также хотим создать базу данных до того, как контейнер PHP попытается получить к ней доступ. Без этой проверки состояния PHP-контейнер может попытаться получить доступ к базе данных, хотя она ещё не существует, что приведет к ошибкам соединения.

## phpMyAdmin

```
# PhpMyAdmin Service
phpmyadmin:
  image: phpmyadmin
  ports:
    - 8080:80
  environment:
    PMA_HOST: mysql
    PMA_USER: root
    PMA_PASSWORD: root
    PMA_PORT: 3306 # не обязателен, если есть сеть
  depends_on:
    mysql:
      condition: service_healthy
  networks:
    - app-network
```

`PMA_HOST: mysql` связь с базой доступна по имени ее сервиса (контейнера), т.е. mysql


### дерево проекта

```
docker-tutorial/
├── docker/
│   ├── mysql/
│   │   └── my.cnf
│   ├── nginx/
│   │   └── conf.d/
│   │       └── php.conf
│   └── php/
│       └── Dockerfile
├── src/
│   └── index.php
├── .env
├── .env.example
├── .gitignore
└── docker-compose.yml
```

