FROM ruby:3.1.6-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    RAILS_ENV=test \
    RAILS_LOG_TO_STDOUT=true
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git subversion sqlite3 libsqlite3-dev \
      libxml2-dev libxslt1-dev pkg-config python3 \
    && rm -rf /var/lib/apt/lists/*

# Redmine is the plugin's host application; keep it separate from the scan target.
RUN git clone --depth 1 --branch 5.1.12 https://github.com/redmine/redmine.git /opt/redmine
COPY . /src
RUN ln -s /src /opt/redmine/plugins/readme_at_repositories
WORKDIR /opt/redmine

RUN printf 'test:\n  adapter: sqlite3\n  database: db/oss_scanner_test.sqlite3\n' > config/database.yml \
    && bundle config set --local without development \
    && bundle install --jobs 4 --retry 3
RUN bundle exec rake generate_secret_token \
    && bundle exec rake db:migrate \
    && bundle exec rake redmine:plugins:migrate
RUN bundle exec rake redmine:plugins:test NAME=readme_at_repositories

# No server starts automatically; the audit can exercise Rails and SQLite offline.
WORKDIR /src
