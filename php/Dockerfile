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
    apt install -y php8.1 \
                php8.1-fpm \
                php8.1-curl \
                php8.1-mbstring \
                php8.1-cli \
                php8.1-xml \
                php8.1-zip \
                php8.1-mysql \
                php8.1-gd
                #php8.1-xdebug

# RUN docker-php-ext-install bcmath

# Install Composer and extensions
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
# 強制降級為 composer 1.x 版，避免 2.x 版不相容所產生的問題
# RUN composer self-update --1
