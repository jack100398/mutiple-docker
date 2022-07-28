# Docker-compose LNMP 完整版

## 此為架構說明，詳情可見實際檔案

### 目錄結構
![](https://i.imgur.com/RRvdby9.png)

### php/fpm/Dockerfile

- [php:7.3-fpm 的dockerfile](https://hub.docker.com/layers/php/library/php/7.3-fpm/images/sha256-dba011c454175a5a191937c591f15d3b1eca68da81e8bfbd4efc1de0f1f166b0?context=explore)

- 因為是 php:7.3-fpm 基於 linux/amd64，所以這裡可以用 apt-get 安裝東東
- docker-php-ext-install .... 這個是docker 安裝 php 擴展的方式
```dockerfile=
FROM php:7.3-fpm

LABEL maintainer="Mahmoud Zalt <mahmoud@zalt.me>"

# Set Environment Variables
ENV DEBIAN_FRONTEND noninteractive

#
#--------------------------------------------------------------------------
# Software's Installation
#--------------------------------------------------------------------------
#
# Installing tools and PHP extentions using "apt", "docker-php", "pecl",
#

# Install "curl", "libmemcached-dev", "libpq-dev", "libjpeg-dev",
#         "libpng-dev", "libfreetype6-dev", "libssl-dev", "libmcrypt-dev",

RUN set -eux; \
    apt-get update; \
    apt-get upgrade -y; \ 
    apt-get install -y --no-install-recommends \
            curl \
            libmemcached-dev \
            libz-dev \
            libpq-dev \
            libjpeg-dev \
            libpng-dev \
            libfreetype6-dev \
            libssl-dev \
            libwebp-dev \
            libmcrypt-dev; \
    rm -rf /var/lib/apt/lists/*

RUN apt-get -y update
RUN apt-get -y install git

RUN set -eux; \
    # Install the PHP pdo_mysql extention
    docker-php-ext-install pdo_mysql; \
    # Install the PHP pdo_pgsql extention
    docker-php-ext-install pdo_pgsql; \
    # Install the PHP gd library
    docker-php-ext-configure gd \
            --with-jpeg-dir=/usr/lib \
            --with-webp-dir=/usr/lib \
            --with-freetype-dir=/usr/include/freetype2; \
    docker-php-ext-install gd; \
    docker-php-ext-install bcmath;\
    php -r 'var_dump(gd_info());'

RUN apt-get update && \
    apt-get install -y --no-install-recommends libzip-dev && \
    rm -r /var/lib/apt/lists/* && \
    docker-php-ext-install -j$(nproc) zip
```

- php/Dockerfile (如果本機有裝php就可審略此項)

```dockerfile=
FROM ubuntu:18.04

ENV DEBIAN_FRONTEND=noninteractive

WORKDIR /data/www

RUN apt update && \
    apt install -y git \
                   zip \
                   curl \
                   apt-utils \
                   software-properties-common

RUN add-apt-repository -y ppa:ondrej/php && \
    apt update && \
    apt install -y apache2 \
                php7.3 \
                php7.3-curl \
                php7.3-mbstring \
                php7.3-cli \
                php7.3-xml \
                php7.3-zip \
                php7.3-mysql \
                php7.3-gd \
                libapache2-mod-php7.3

# Install Composer and extensions
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
```

### nginx/conf.d/xxx.local.host.conf
- line 9 - 因為laravel的程式進入點是public/index.php 
- line22 - 因為php跟nginx 不在同一台機器上，所以用tcp的方式處理
- line23 - 這裡的php_fpm是docker-image 名稱

:::info
如果有新的專案，會需要更動的地方只有 `server_name` 跟 `root`，還有檔名。
例如我今天的檔名叫做 `gog.local.host.conf`，server_name跟root就是如下，我會建議跟設定檔同名
:::

```nginx=
server {
  listen 80;
  server_name gog.local.host;
  root /data/www/gog.local.host/public;

  add_header X-Frame-Options "SAMEORIGIN";
  add_header X-Content-Type-Options "nosniff";

  index index.php;

  charset utf-8;

  location / {
      try_files $uri $uri/ /index.php?$query_string;
  }

  location = /favicon.ico { access_log off; log_not_found off; }
  location = /robots.txt  { access_log off; log_not_found off; }

  error_page 404 /index.php;

  location ~ \.php$ {
      fastcgi_pass php_fpm:9000;
      fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
      #fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
      include fastcgi_params;
  }

  location ~ /\.(?!well-known).* {
      deny all;
  }
}
```

### php, php-fpm 個別的作用

- php 的作用是為了php-cli而已，所以如果本機有裝php，就可以不用額外使用php這個image了

- php-fpm 是為了處理網頁的請求，所以一定要裝，composer 安裝套件也都到 php-fpm這個container裡面安裝

### docker-composer.yml

- 將 ./www 掛載到所有 image 上
- 如果本機有安裝node的話，就不用使用node這個image
- ps: tty的意思->讓Docker分配一個虛擬終端（pseudo-tty）並綁定到 Container 的標準輸入上->也就是container運行完成之後不會離開(exit)
- 正常情況下，docker-compose 在 build 時，會自動建立network，所以不用特別設定，但是這邊為了保險，所以設定network
- mysql的環境設定值可以到[這裡](https://hub.docker.com/_/mysql)看
:::success
這一段的用意是讓docker-composer 在build iamge時，直接依照指定的dockerfile建立，在沒有建立image的情況下，會出現沒有該iamge的錯誤
```
build:
      context: .
      dockerfile: ./php/Dockerfile
```
:::

```yml=
services:
  # php-fpm 7.3
  php:
    build:
      context: .
      dockerfile: ./php/Dockerfile
    container_name: local-php
    tty: true
    restart: unless-stopped
    volumes:
      - ./www:/data/www
    networks:
      - grafana

  php-fpm:
    build:
      context: .
      dockerfile: ./php/fpm/Dockerfile
    container_name: php_fpm
    restart: unless-stopped
    volumes:
      - ./www:/data/www
    ports:
      - 9000:9000
    networks:
      - grafana

  # nginx
  nginx:
    image: nginx:latest
    container_name: local-nginx
    restart: unless-stopped
    ports:
      - 80:80
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./www:/data/www
    environment:
      - TZ=Asia/Taipei
    networks:
      - grafana

  node:
    image: node:14-alpine
    container_name: node
    tty: true
    ports:
      - 3000:80
    volumes:
      - ./www:/data/www
    networks:
      - grafana

  # mysql
  mysql:
    image: mysql:5.7
    container_name: local-mysql
    restart: always
    ports:
      - 3306:3306
    command:
      - --innodb-buffer-pool-size=64M
    volumes:
      - ./mysql:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: "aaaa1111"
      MYSQL_DATABASE: "mysql"
      MYSQL_USER: "phil_hsu"
      MYSQL_PASSWORD: "aaaa1111"
    networks:
      - grafana

networks:
  grafana:
```

### 注意事項

1. 如果是需要互相取用的container 有可能因為另外一個container還沒運行完成，所以會失敗 這時候加上 restart: always 即可

2. M1 的 mac 在docker-compose安裝mysql的問題[參考這裡](https://chilunhuang.github.io/posts/8942/)

###### tags: `Docker-Compose`