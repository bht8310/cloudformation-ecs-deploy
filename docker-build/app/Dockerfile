FROM php:7.2-fpm

RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
RUN php composer-setup.php
RUN php -r "unlink('composer-setup.php');"
RUN mv composer.phar /usr/bin/composer

RUN apt-get update \
    && apt-get install -y git \
    && apt-get install -y nodejs \
    && apt-get install -y npm \
    && apt-get install -y zip unzip

RUN docker-php-ext-install pdo_mysql

ADD ./app /app

WORKDIR /app

#RUN cp ./.env.prod .env
#RUN chown -R www-data.www-data storage
#RUN composer install

