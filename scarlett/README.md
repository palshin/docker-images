# Marabel Docker Image

Базовый образ для приложения на стеке:
- PHP 8
- Laravel Octane with Swoole
- Redis
- SQL Server

Supercronic для запуска задач по расписанию.

Окружение настроено для использования JIT и opcache.

Образ production ready, но может быть использован и при разработке.

Dockerfile самого приложения может выглядеть так:

```dockerfile
FROM epalshin/scarlett:latest

# copy start container script
COPY ./docker/start-container /usr/local/bin/start-container
RUN chmod +x /usr/local/bin/start-container

# use directory with application sources by default
WORKDIR /app

# fix rights issue (workdir creates directory with root permissions)
RUN chown -R appuser:appuser /app
RUN chmod 755 /app

# use an unprivileged user by default
USER appuser:appuser

# copy composer (json|lock) files for dependencies layer caching
COPY --chown=appuser:appuser ./composer.* package*.json /app/

# install composer dependencies (autoloader MUST be generated later!)
# and node dependecies (Laravel Octane needs chokidar for autoreloading)
RUN composer install -n --no-dev --no-cache --no-ansi --no-autoloader --no-scripts --prefer-dist \
    && npm install

# copy application sources into image (completely)
COPY --chown=appuser:appuser . /app/

RUN set -x \
    # generate composer autoloader and trigger scripts
    && composer dump-autoload -n --classmap-authoritative \
    # "fix" composer issue "Cannot create cache directory /tmp/composer/cache/..." for docker-compose usage
    && chmod -R 777 ${COMPOSER_HOME}/cache \
    # create the symbolic links configured for the application
    && php ./artisan storage:link

# default image entrypoint
ENTRYPOINT ["start-container"]
```