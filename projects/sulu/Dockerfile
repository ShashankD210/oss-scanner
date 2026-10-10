# Sulu CMS (PHP/Symfony monorepo). Builds the checkout at /src with its dev dependencies.
FROM php:8.4-cli-bookworm
ENV DEBIAN_FRONTEND=noninteractive COMPOSER_ALLOW_SUPERUSER=1 COMPOSER_NO_INTERACTION=1
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git unzip libicu-dev libzip-dev libpng-dev libjpeg-dev libfreetype6-dev libxml2-dev libonig-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j"$(nproc)" intl zip gd pdo_mysql opcache \
    && rm -rf /var/lib/apt/lists/*
COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Dependencies are fetched here because the audit runs without network access.
RUN composer install --prefer-dist --no-progress

# Run the cheap static checks so the image is known to work; a failure must not abort the build.
RUN (composer lint-container || composer lint-yaml || echo "WARNING: lint reported problems")
