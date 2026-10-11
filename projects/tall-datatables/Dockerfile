FROM php:8.5-cli-trixie
ENV DEBIAN_FRONTEND=noninteractive \
    COMPOSER_ALLOW_SUPERUSER=1 \
    COMPOSER_NO_INTERACTION=1

# PHP extensions as in CI (.github/workflows/tests.yml), plus sockets for pestphp/pest-plugin-browser. pdo_sqlite, mbstring, dom, iconv and curl ship with the image.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git unzip gnupg libzip-dev libicu-dev libpng-dev libjpeg-dev libfreetype6-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j"$(nproc)" zip intl gd bcmath pcntl exif sockets \
    && curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_22.x nodistro main" > /etc/apt/sources.list.d/nodesource.list \
    && apt-get update && apt-get install -y --no-install-recommends nodejs \
    && rm -rf /var/lib/apt/lists/*
COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Same steps as CI: resolve all dependencies including dev (Testbench, Pest, Livewire test plugins), then build the
# frontend assets the Blade views reference. The repository ships no composer.lock, so composer resolves here.
RUN composer update --prefer-dist --no-progress
RUN npm ci && npm run build

# Unit and Feature tests run on in-memory sqlite (tests/TestCase.php). Browser tests need Playwright and are skipped.
RUN ./vendor/bin/testbench package:test --testsuite=Unit,Feature || echo "WARNING: tests reported failures"
