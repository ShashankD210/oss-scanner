# Mantle framework (the mantle-framework/* monorepo).
#
# /src             the mantle-framework git checkout (the repository named in project.yaml), with its Composer
#                  dependencies installed
# /opt/wordpress   the WordPress install the test suite runs against, with its test library, the plugins the suite
#                  loads, and a MariaDB database, all set up by a first run of the suite during this build
#
# Mantle's test bootstrap (tests/bootstrap.php) rsyncs /src into /opt/wordpress/wp-content/plugins/mantle-framework and
# re-runs PHPUnit there, so edits made in /src are picked up on every run. Start MariaDB with `mantle-start` first.
FROM php:8.4-cli-trixie

ENV DEBIAN_FRONTEND=noninteractive \
    COMPOSER_ALLOW_SUPERUSER=1 \
    COMPOSER_NO_INTERACTION=1 \
    COMPOSER_MEMORY_LIMIT=-1 \
    WP_CLI_ALLOW_ROOT=1 \
    WP_CORE_DIR=/opt/wordpress \
    CACHEDIR=/opt/wordpress-cache \
    WP_DB_NAME=wordpress_unit_tests \
    WP_DB_USER=root \
    WP_DB_PASSWORD=root \
    WP_DB_HOST=127.0.0.1

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git unzip curl wget rsync procps mariadb-server mariadb-client \
      libfreetype6-dev libjpeg62-turbo-dev libpng-dev libwebp-dev libzip-dev libicu-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-install -j"$(nproc)" mysqli pdo_mysql gd zip intl exif bcmath pcntl \
    && rm -rf /var/lib/apt/lists/*
COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

# WP-CLI, which the WordPress install script uses and which is handy for reproductions.
RUN curl -fsSL -o /usr/local/bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
    && chmod +x /usr/local/bin/wp \
    && wp --info

RUN { echo 'memory_limit=-1'; \
      echo 'error_reporting=E_ALL'; \
    } > /usr/local/etc/php/conf.d/oss-scanner.ini

# mantle-start starts MariaDB (there is no init system in the container) and lets root sign in with the password the
# test suite expects. It is safe to run again.
RUN printf '%s\n' '#!/bin/sh' 'set -e' \
      'mkdir -p /run/mysqld && chown mysql:mysql /run/mysqld' \
      'mariadb-admin ping --silent 2>/dev/null || (mysqld_safe >/dev/null 2>&1 &)' \
      'until mariadb-admin ping --silent 2>/dev/null; do sleep 0.5; done' \
      > /usr/local/bin/mantle-start \
    && printf '%s\n' '#!/bin/sh' 'mariadb-admin shutdown 2>/dev/null; true' > /usr/local/bin/mantle-stop \
    && chmod +x /usr/local/bin/mantle-start /usr/local/bin/mantle-stop \
    && mantle-start \
    && mariadb -e "ALTER USER 'root'@'localhost' IDENTIFIED VIA unix_socket OR mysql_native_password USING PASSWORD('root'); \
         CREATE USER IF NOT EXISTS 'root'@'127.0.0.1' IDENTIFIED BY 'root'; \
         GRANT ALL ON *.* TO 'root'@'127.0.0.1' WITH GRANT OPTION; FLUSH PRIVILEGES;" \
    && mantle-stop

# scanner contract: the mantle-framework checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN composer install --no-progress

# The first run of the suite downloads WordPress, its test library and the plugins tests/bootstrap.php asks for into
# /opt/wordpress. Later runs find them there and work offline. Run it single-site and multisite so both configs exist,
# and do not let a failing test abort the build.
RUN mantle-start \
    && (vendor/bin/phpunit tests/Database/Model/ModelTest.php || echo "WARNING: phpunit reported failures") \
    && (WP_MULTISITE=1 vendor/bin/phpunit tests/Database/Model/ModelTest.php || echo "WARNING: multisite phpunit reported failures") \
    && test -f /opt/wordpress/wp-load.php \
    && mantle-stop

# Keep build products out of the checkout's `git status`.
RUN printf '.phpunit.result.cache\n.phpunit.cache/\n' >> /src/.git/info/exclude || true
