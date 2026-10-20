# Mad Framework is a Laravel package (madbuilder/framework). The checkout goes to /src, and a Laravel 13
# host application at /app requires it through a Composer path repository, so the package is exercised
# exactly as installed apps load it (vendor/madbuilder/framework -> /src).
FROM php:8.4-cli-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    COMPOSER_ALLOW_SUPERUSER=1 \
    COMPOSER_NO_INTERACTION=1

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git unzip sqlite3 \
    && rm -rf /var/lib/apt/lists/*

# PHP extensions used by the package and its dependencies (dompdf, openspout, bacon-qr-code, i18n).
ADD --chmod=0755 https://github.com/mlocati/docker-php-extension-installer/releases/download/2.12.0/install-php-extensions /usr/local/bin/
RUN install-php-extensions gd intl zip bcmath pcntl exif opcache \
    && printf 'memory_limit=-1\n' > /usr/local/etc/php/conf.d/zz-mad.ini
COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Host application: fresh Laravel 13 + the package from /src (symlinked by the path repository).
RUN composer create-project laravel/laravel:^13.0 /app --prefer-dist \
    && cd /app \
    && composer config repositories.mad-framework '{"type": "path", "url": "/src", "options": {"symlink": true}}' \
    && composer require 'madbuilder/framework:*@dev' --with-all-dependencies \
    && sed -i -e 's/^DB_CONNECTION=.*/DB_CONNECTION=sqlite/' -e 's/^APP_DEBUG=.*/APP_DEBUG=true/' .env \
    && touch database/database.sqlite \
    && php artisan key:generate --force \
    && php artisan migrate --force

# Smoke check: the provider boots and registers its routes. Does not abort the build.
RUN cd /app && (php artisan route:list --no-ansi | tail -n 20 || echo "WARNING: route:list failed")
