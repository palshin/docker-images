FROM alpine:3.13 AS iconv-build
RUN apk add --no-cache gnu-libiconv

# build application runtime, image page: <https://hub.docker.com/_/php>
FROM php:8.0.12-cli-alpine

LABEL org.opencontainers.image.authors="eo.palshin@gmail.com"

# fix iconv issue <https://github.com/docker-library/php/issues/1121>
COPY --from=iconv-build /usr/lib/preloadable_libiconv.so /usr/lib/preloadable_libiconv.so
ENV LD_PRELOAD /usr/lib/preloadable_libiconv.so php

# install composer, image page: <https://hub.docker.com/_/composer>
COPY --from=composer:2.1.9 /usr/bin/composer /usr/bin/composer

# install install-php-extensions tool, docs: <https://github.com/mlocati/docker-php-extension-installer>
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/local/bin/

ENV COMPOSER_HOME="/tmp/composer" \
  TZ=UTC \
  PHP_OPCACHE_VALIDATE_TIMESTAMPS="0" \
  PHP_OPCACHE_MAX_ACCELERATED_FILES="20000" \
  PHP_OPCACHE_MEMORY_CONSUMPTION="350" \
  PHP_OPCACHE_MAX_WASTED_PERCENTAGE="15"

RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

RUN set -x \
    # install permanent dependencies
    && apk add --no-cache postgresql-libs icu-libs npm \
    && apk add --no-cache --virtual .build-deps postgresql-dev autoconf openssl make g++ \
    && install-php-extensions swoole pdo_pgsql opcache \
        pcntl intl gd zip bcmath exif redis 1>/dev/null \
    # install supercronic (for laravel task scheduling), project page: <https://github.com/aptible/supercronic>
    && wget -q "https://github.com/aptible/supercronic/releases/download/v0.1.12/supercronic-linux-amd64" \
         -O /usr/bin/supercronic \
    && chmod +x /usr/bin/supercronic \
    && mkdir /etc/supercronic \
    && echo '*/1 * * * * php /app/artisan schedule:run' > /etc/supercronic/laravel \
    # generate self-signed SSL key and certificate files
    && openssl req -x509 -nodes -days 1095 -newkey rsa:2048 \
        -subj "/C=CA/ST=QC/O=Company, Inc./CN=mydomain.com" \
        -addext "subjectAltName=DNS:mydomain.com" \
        -keyout /etc/ssl/private/selfsigned.key \
        -out /etc/ssl/certs/selfsigned.crt \
    && chmod 644 /etc/ssl/private/selfsigned.key \
    # make clean up
    && docker-php-source delete \
    && apk del .build-deps \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /tmp/pear \
    # show installed PHP modules
    && php -m \
    # create unprivileged user
    && adduser \
        --disabled-password \
        --shell "/sbin/nologin" \
        --uid "10001" \
        --gecos "" \
        "appuser"

COPY ./php.ini ${PHP_INI_DIR}/conf.d/88-marabel.ini