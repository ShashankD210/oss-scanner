# WIT Calendar backend (WITCodingClub/calendar-backend), a Rails 8 app.
#
# /src    the checkout, bundled with every Gemfile group and with Tailwind CSS built, so the RSpec suite runs offline.
#
# PostgreSQL 17 with pgvector is installed in the image and the test database is created at build time. There is no
# init system in the container: run `witcal-services-start` before the specs. It is safe to run again.
#
#   witcal-services-start
#   cd /src && bundle exec rspec spec/requests/api/authentication_spec.rb
#   cd /src && bundle exec rspec                                 # the full suite; system specs use headless Chromium
FROM ruby:4.0-trixie

# The specs read the database from POSTGRES_*, and HASHID_SALT lets the app boot without config/master.key.
ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    BUNDLE_JOBS=8 \
    BUNDLE_RETRY=3 \
    RAILS_ENV=test \
    HASHID_SALT=oss_scanner_test_salt \
    POSTGRES_HOST=localhost \
    POSTGRES_PORT=5432 \
    POSTGRES_USER=postgres \
    POSTGRES_PASSWORD=postgres \
    BROWSER_PATH=/usr/bin/chromium

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git build-essential pkg-config procps \
      postgresql postgresql-contrib postgresql-17-pgvector libpq-dev \
      libvips libyaml-dev libssl-dev \
      chromium \
    && rm -rf /var/lib/apt/lists/*

# witcal-services-start starts PostgreSQL on its default local port.
RUN printf '%s\n' '#!/bin/sh' 'set -e' \
      'pg_ctlcluster "$(ls /etc/postgresql)" main start 2>/dev/null || true' \
      'until pg_isready -q; do sleep 0.5; done' \
      > /usr/local/bin/witcal-services-start \
    && chmod +x /usr/local/bin/witcal-services-start

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN bundle install && bin/rails tailwindcss:build

RUN witcal-services-start \
    && su postgres -c "psql -c \"ALTER USER postgres PASSWORD 'postgres'\"" \
    && bin/rails db:test:prepare

# Smoke-test the image without letting a failing spec abort the build.
RUN witcal-services-start \
    && (bundle exec rspec spec/requests/api/authentication_spec.rb spec/requests/admin/authorization_spec.rb \
        || echo "WARNING: smoke specs reported failures")
