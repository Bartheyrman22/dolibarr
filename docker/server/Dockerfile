FROM php:8.4-apache

RUN a2enmod rewrite
RUN a2enmod headers

RUN apt-get update && apt-get install -y git unzip zip npm sendmail

WORKDIR /var/www/html

COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/local/bin/
RUN install-php-extensions gd mysqli zip curl xdebug mbstring xml memprof openssl json intl calendar
ENV MEMPROF_PROFILE=dump_on_limit

COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer


# Xdebug logging
RUN mkdir /var/www/log
RUN chmod 777 /var/www/log

# set server name
RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf

RUN useradd -ms /bin/bash newuser
USER 1000
