# PerfLocale, a multilingual plugin for WordPress.
#
# /src                                           the perflocale/perflocale checkout (the shipped plugin; it has no
#                                                build step and no Composer or npm dependencies)
# /var/www/html                                  a WordPress install served by Apache with mod_rewrite and .htaccess
# /var/www/html/wp-content/plugins/perflocale    a symlink to /src, so a patch in /src is live at once
#
# The database is MariaDB. Start MariaDB and Apache with `wp-start` (stop with `wp-stop`); the site listens on
# http://127.0.0.1/. WP-CLI runs as root: `wp --path=/var/www/html ...` (WP_CLI_ALLOW_ROOT is set).
FROM php:8.4-apache-trixie

ENV DEBIAN_FRONTEND=noninteractive \
    WP_CLI_ALLOW_ROOT=1 \
    WP_PATH=/var/www/html

# mysqli is the extension WordPress uses; the rest are the ones the WordPress hosting handbook recommends. The base
# image already provides OPcache.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git unzip curl procps mariadb-server mariadb-client \
      libfreetype6-dev libjpeg62-turbo-dev libpng-dev libwebp-dev libzip-dev libicu-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-install -j"$(nproc)" mysqli pdo_mysql gd zip intl exif bcmath \
    && rm -rf /var/lib/apt/lists/*

# WP-CLI, for setting up languages, content, users and options in reproductions.
RUN curl -fsSL -o /usr/local/bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
    && chmod +x /usr/local/bin/wp \
    && wp --info

# Production-like PHP: errors are logged, not shown; OPcache picks up edited files at once so a patch can be checked
# straight away.
RUN { echo 'memory_limit=256M'; \
      echo 'upload_max_filesize=64M'; \
      echo 'post_max_size=64M'; \
      echo 'error_reporting=E_ALL'; \
      echo 'display_errors=Off'; \
      echo 'log_errors=On'; \
      echo 'error_log=/dev/stderr'; \
      echo 'opcache.validate_timestamps=1'; \
      echo 'opcache.revalidate_freq=0'; \
    } > /usr/local/etc/php/conf.d/oss-scanner.ini

# Apache serves /var/www/html with .htaccess enabled (PerfLocale's language URLs need pretty permalinks).
RUN a2enmod rewrite headers expires \
    && printf '<Directory /var/www/html>\n  Options -MultiViews +FollowSymLinks\n  AllowOverride All\n  Require all granted\n</Directory>\nServerName 127.0.0.1\n' \
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

# scanner contract: the perflocale checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# WordPress core (latest release), plus WooCommerce downloaded but left inactive: PerfLocale's WooCommerce
# integration (src/WooCommerce/, addons/woocommerce/) is exercised with `wp plugin activate woocommerce`.
RUN rm -rf "$WP_PATH" && mkdir -p "$WP_PATH" \
    && wp core download --path="$WP_PATH" \
    && wp core version --path="$WP_PATH" \
    && ln -s /src "$WP_PATH/wp-content/plugins/perflocale"

# A working single-site WordPress on MariaDB with PerfLocale active, three languages (English default, German,
# French), pretty permalinks, one published page and post, and one user per role for access-control reproductions:
#   admin / admin               administrator (has unfiltered_html)
#   editor / editor             editor        (has unfiltered_html)
#   translator / translator     PerfLocale's Translator role (perflocale_translator: edits others' posts and pages,
#                               perflocale_translate, perflocale_use_mt; no unfiltered_html)
#   author / author             author
#   contributor / contributor   contributor
#   subscriber / subscriber     subscriber
#   (anonymous)                 no login
RUN wp-start \
    && mariadb -e "CREATE DATABASE wordpress; \
         CREATE USER 'wordpress'@'%' IDENTIFIED BY 'wordpress'; GRANT ALL ON wordpress.* TO 'wordpress'@'%';" \
    && wp config create --path="$WP_PATH" --dbname=wordpress --dbuser=wordpress --dbpass=wordpress \
         --dbhost=127.0.0.1 --skip-check \
    && wp core install --path="$WP_PATH" --url=http://127.0.0.1 --title="OSS Scanner PerfLocale" \
         --admin_user=admin --admin_password=admin --admin_email=admin@example.com --skip-email \
    && wp rewrite structure --path="$WP_PATH" '/%postname%/' \
    && printf '%s\n' '# BEGIN WordPress' '<IfModule mod_rewrite.c>' 'RewriteEngine On' 'RewriteBase /' \
         'RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]' 'RewriteRule ^index\.php$ - [L]' \
         'RewriteCond %{REQUEST_FILENAME} !-f' 'RewriteCond %{REQUEST_FILENAME} !-d' 'RewriteRule . /index.php [L]' \
         '</IfModule>' '# END WordPress' > "$WP_PATH/.htaccess" \
    && (wp language core install --path="$WP_PATH" de_DE fr_FR || echo "WARNING: core language packs not installed") \
    && wp plugin activate --path="$WP_PATH" perflocale \
    && for spec in "en_US --default" "de_DE" "fr_FR"; do \
         set -- $spec; \
         wp perflocale languages list --path="$WP_PATH" --format=csv | cut -d, -f3 | grep -qx "$1" \
           || wp perflocale languages add --path="$WP_PATH" $spec; \
       done \
    && wp perflocale languages list --path="$WP_PATH" \
    && wp plugin install --path="$WP_PATH" woocommerce \
    && for role in editor author contributor subscriber; do \
         wp user create --path="$WP_PATH" "$role" "$role@example.com" --role="$role" --user_pass="$role"; \
       done \
    && wp user create --path="$WP_PATH" translator translator@example.com --role=perflocale_translator \
         --user_pass=translator \
    && wp post create --path="$WP_PATH" --post_type=page --post_status=publish --post_title='Sample page' \
         --post_content='<!-- wp:paragraph --><p>A page in the default language.</p><!-- /wp:paragraph -->' \
    && wp post create --path="$WP_PATH" --post_type=post --post_status=publish --post_title='Sample post' \
         --post_content='<!-- wp:paragraph --><p>A post in the default language.</p><!-- /wp:paragraph -->' \
    && mkdir -p "$WP_PATH/wp-content/uploads" "$WP_PATH/wp-content/languages" \
    && chown -R www-data:www-data "$WP_PATH/wp-content/uploads" "$WP_PATH/wp-content/languages" \
    && wp user list --path="$WP_PATH" --fields=user_login,roles \
    && wp plugin list --path="$WP_PATH" --fields=name,status,version \
    && echo "front page: HTTP $(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1/)" \
    && echo "German front page: HTTP $(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1/de/)" \
    && wp-stop
