# Burst Statistics, the free WordPress analytics plugin.
#
# /src                          the burst-statistics git checkout (the repository named in project.yaml), with its PHP
#                               dev dependencies installed. The built JavaScript and CSS are committed, so no asset build
#                               is needed.
# /wordpress                    a WordPress install (latest stable release) that Apache serves; /var/www/html is a
#                               symlink to it, because the plugin's PHPUnit bootstrap loads /var/www/html/wp-load.php.
# /wordpress/wp-content/plugins/burst-statistics
#                               a symlink to /src, so the site runs the checkout and a patch in /src is live at once.
#
# WordPress lives at /wordpress rather than under /var/www because the tracking beacon, /src/endpoint.php, loads
# WordPress itself with SHORTINIT and looks for wp-load.php relative to its own resolved path (/src): three levels up,
# then one directory below that. From /src that is /, and /wordpress is the directory it finds.
#
# The site is served by Apache with mod_rewrite and AllowOverride All. The database is MariaDB. Start both with
# `wp-start`; the site listens on http://127.0.0.1/.
FROM php:8.2-apache-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    COMPOSER_ALLOW_SUPERUSER=1 \
    COMPOSER_NO_INTERACTION=1 \
    COMPOSER_MEMORY_LIMIT=-1 \
    WP_CLI_ALLOW_ROOT=1

# The base image already provides OPcache. mysqli is the extension WordPress itself uses; the rest are the ones the
# WordPress hosting handbook recommends.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git unzip curl procps mariadb-server mariadb-client \
      libfreetype6-dev libjpeg62-turbo-dev libpng-dev libwebp-dev libzip-dev libicu-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-install -j"$(nproc)" mysqli pdo_mysql gd zip intl exif bcmath \
    && rm -rf /var/lib/apt/lists/*
COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

# WP-CLI, for setting up content, users and options in reproductions.
RUN curl -fsSL -o /usr/local/bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
    && chmod +x /usr/local/bin/wp \
    && wp --info

# Production-like PHP: errors are logged, not shown; OPcache picks up edited files immediately so a patch can be
# checked straight away.
RUN { echo 'memory_limit=256M'; \
      echo 'error_reporting=E_ALL'; \
      echo 'display_errors=Off'; \
      echo 'log_errors=On'; \
      echo 'error_log=/dev/stderr'; \
      echo 'opcache.validate_timestamps=1'; \
      echo 'opcache.revalidate_freq=0'; \
    } > /usr/local/etc/php/conf.d/oss-scanner.ini

# Apache serves /var/www/html (a symlink to /wordpress) with .htaccess enabled.
RUN a2enmod rewrite headers expires \
    && printf '<Directory /var/www/html>\n  Options -MultiViews +FollowSymLinks\n  AllowOverride All\n  Require all granted\n</Directory>\n<Directory /wordpress>\n  Options -MultiViews +FollowSymLinks\n  AllowOverride All\n  Require all granted\n</Directory>\n<Directory /src>\n  Options -MultiViews +FollowSymLinks\n  AllowOverride All\n  Require all granted\n</Directory>\nServerName 127.0.0.1\n' \
         > /etc/apache2/conf-available/wordpress.conf \
    && a2enconf wordpress

# wp-start starts MariaDB and Apache (there is no init system in the container). It is safe to run again.
RUN printf '%s\n' '#!/bin/sh' 'set -e' \
      'mkdir -p /run/mysqld && chown mysql:mysql /run/mysqld' \
      'mariadb-admin ping --silent 2>/dev/null || (mysqld_safe >/dev/null 2>&1 &)' \
      'until mariadb-admin ping --silent 2>/dev/null; do sleep 0.5; done' \
      'pgrep -x apache2 >/dev/null || apachectl start' \
      > /usr/local/bin/wp-start \
    && printf '%s\n' '#!/bin/sh' 'apachectl stop 2>/dev/null; mariadb-admin shutdown 2>/dev/null; true' \
      > /usr/local/bin/wp-stop \
    && chmod +x /usr/local/bin/wp-start /usr/local/bin/wp-stop

# scanner contract: the burst-statistics checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# PHP dev dependencies (PHPUnit, yoast/phpunit-polyfills, PHPCS with WPCS).
RUN composer install --no-progress

# WordPress core, plus the e-commerce plugins Burst integrates with (installed, not active), so their integrations can
# be exercised offline with `wp plugin activate`.
RUN rm -rf /var/www/html \
    && mkdir -p /wordpress \
    && ln -s /wordpress /var/www/html \
    && wp core download --path=/wordpress \
    && for plugin in woocommerce easy-digital-downloads; do \
         curl -fsSL -o "/tmp/$plugin.zip" "https://downloads.wordpress.org/plugin/$plugin.latest-stable.zip" \
         && unzip -q "/tmp/$plugin.zip" -d /wordpress/wp-content/plugins \
         && rm "/tmp/$plugin.zip"; \
       done \
    && ln -s /src /wordpress/wp-content/plugins/burst-statistics

# Install a working single-site WordPress on MariaDB with Burst active, and one user per default role for
# access-control reproductions:
#   admin / admin               administrator (has the Burst capabilities view_burst_statistics,
#                               manage_burst_statistics and view_sales_burst_statistics)
#   editor / editor             editor
#   author / author             author
#   contributor / contributor   contributor
#   subscriber / subscriber     subscriber
#   (anonymous)                 no login
RUN wp-start \
    && mariadb -e "CREATE DATABASE wordpress; \
         CREATE USER 'wordpress'@'%' IDENTIFIED BY 'wordpress'; \
         GRANT ALL ON wordpress.* TO 'wordpress'@'%';" \
    && wp config create --path=/wordpress --dbname=wordpress --dbuser=wordpress --dbpass=wordpress --dbhost=127.0.0.1 \
         --skip-check \
    && wp core install --path=/wordpress --url=http://127.0.0.1 --title="OSS Scanner Burst Statistics" \
         --admin_user=admin --admin_password=admin --admin_email=admin@example.com --skip-email \
    && for role in editor author contributor subscriber; do \
         wp user create --path=/wordpress "$role" "$role@example.com" --role="$role" --user_pass="$role"; \
       done \
    && wp rewrite structure --path=/wordpress '/%postname%/' \
    && printf '%s\n' '# BEGIN WordPress' '<IfModule mod_rewrite.c>' 'RewriteEngine On' 'RewriteBase /' \
         'RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]' 'RewriteRule ^index\.php$ - [L]' \
         'RewriteCond %{REQUEST_FILENAME} !-f' 'RewriteCond %{REQUEST_FILENAME} !-d' 'RewriteRule . /index.php [L]' \
         '</IfModule>' '# END WordPress' > /wordpress/.htaccess \
    && wp plugin activate --path=/wordpress burst-statistics \
    && wp plugin list --path=/wordpress --fields=name,status,version \
    && wp db query --path=/wordpress "SHOW TABLES LIKE 'wp_burst_%'" \
    && mkdir -p /wordpress/wp-content/uploads \
    && chown -R www-data:www-data /wordpress/wp-content \
    && wp-stop

# Smoke-test the site, the tracking beacon and a slice of the PHPUnit suite, without letting a failing test abort the
# build.
RUN wp-start \
    && curl -fsS -o /dev/null http://127.0.0.1/ \
    && curl -fsS -o /dev/null --data 'request=test' http://127.0.0.1/wp-content/plugins/burst-statistics/endpoint.php \
    && (vendor/bin/phpunit --configuration phpunit.xml.dist || echo "WARNING: phpunit reported failures") \
    && wp-stop

# Keep build products out of the checkout's `git status`.
RUN printf 'vendor/\n' >> /src/.git/info/exclude || true

WORKDIR /src
