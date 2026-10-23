# Weave (patchworklabsorg/weave), the Patchwork Labs identity provider and OAuth 2.0 / OpenID Connect server, a Rails 8
# app.
#
# /src    the checkout, bundled with every Gemfile group, with JavaScript and Tailwind CSS built, so the RSpec and
#         Minitest suites run offline.
#
# PostgreSQL 17 is installed in the image and the test databases are created at build time. There is no init system
# in the container: run `weave-services-start` before the tests. It is safe to run again.
#
#   weave-services-start
#   cd /src && bundle exec rspec spec/requests/oauth/introspection_spec.rb
#   cd /src && bundle exec rspec                                 # the RSpec suite
#   cd /src && bin/rails test                                    # the Minitest suite; system tests use headless Chromium
FROM ruby:4.0.7-trixie

# The test databases connect to localhost with PGUSER and PGPASSWORD. HASHID_SALT stands in for the salt in the
# encrypted credentials, which the image does not have. SE_OFFLINE keeps Selenium Manager to the driver installed here.
ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    BUNDLE_JOBS=8 \
    BUNDLE_RETRY=3 \
    RAILS_ENV=test \
    HASHID_SALT=oss_scanner_test_salt \
    PGUSER=postgres \
    PGPASSWORD=postgres \
    SE_OFFLINE=true

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git build-essential pkg-config procps unzip \
      postgresql postgresql-contrib libpq-dev \
      libyaml-dev libssl-dev libargon2-dev \
      chromium chromium-driver \
    && rm -rf /var/lib/apt/lists/*

# Chromium won't start as root without --no-sandbox; Debian's launcher reads extra flags from /etc/chromium.d.
RUN echo 'export CHROMIUM_FLAGS="$CHROMIUM_FLAGS --no-sandbox"' > /etc/chromium.d/no-sandbox

# The JavaScript is bundled with Rollup through Bun.
COPY --from=oven/bun:1 /usr/local/bin/bun /usr/local/bin/bun

# weave-services-start starts PostgreSQL on its default local port.
RUN printf '%s\n' '#!/bin/sh' 'set -e' \
      'pg_ctlcluster "$(ls /etc/postgresql)" main start 2>/dev/null || true' \
      'until pg_isready -q; do sleep 0.5; done' \
      > /usr/local/bin/weave-services-start \
    && chmod +x /usr/local/bin/weave-services-start

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN bundle install \
    && bun install --frozen-lockfile \
    && bun run build \
    && bin/rails tailwindcss:build

RUN weave-services-start \
    && su postgres -c "psql -c \"ALTER USER postgres PASSWORD 'postgres'\"" \
    && bin/rails db:create db:schema:load

# Smoke-test the image without letting a failing test abort the build.
RUN weave-services-start \
    && (bundle exec rspec spec/requests/oauth/introspection_spec.rb spec/requests/oauth/openid_connect_spec.rb \
        || echo "WARNING: smoke tests reported failures")
