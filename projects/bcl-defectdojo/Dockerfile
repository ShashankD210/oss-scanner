# OSS Scanner — bancolombia/django-DefectDojo (#trunk)
# - Debian base: Anthropic scanner layer can `apt install python3` (3.11) without breaking dpkg.
# - Python 3.12 (backports): requirements.txt pins numpy 2.5.x / Django 6 which need >=3.12.
FROM debian:bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    PYCURL_SSL_LIBRARY=openssl \
    PYTHONUNBUFFERED=1 \
    PATH=/opt/venv/bin:/usr/local/bin:/usr/bin:/bin

RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl \
    && echo "deb http://deb.debian.org/debian bookworm-backports main" \
       > /etc/apt/sources.list.d/backports.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends -t bookworm-backports \
         python3.12 python3.12-venv python3.12-dev \
    && apt-get install -y --no-install-recommends \
         python3 python3-venv \
         git bash procps \
         build-essential gcc g++ nasm \
         libpq-dev postgresql postgresql-client \
         libopenjp2-7 libjpeg62 libtiff6 xmlsec1 uuid-runtime \
         libisal-dev \
         autoconf gettext zlib1g-dev libcurl4-openssl-dev pkg-config \
    && rm -rf /var/lib/apt/lists/* \
    && python3.12 -m venv /opt/venv \
    && /opt/venv/bin/pip install --upgrade pip

COPY . /src
WORKDIR /src
RUN ln -sf /src /app

RUN CPUCOUNT=1 /opt/venv/bin/pip wheel --wheel-dir=/tmp/wheels -r requirements.txt \
 && /opt/venv/bin/pip install --no-cache-dir --no-index --find-links=/tmp/wheels -r requirements.txt

RUN cp docker/wait-for-it.sh docker/secret-file-loader.sh docker/reach_database.sh \
       docker/entrypoint-unit-tests.sh / \
 && rm -f /readme.txt \
 && mkdir -p components/node_modules media/threat dojo/migrations

ENV DD_DEBUG=True \
    DD_LOG_LEVEL=ERROR \
    PYTHONWARNINGS=error \
    DD_DATABASE_ENGINE=django.db.backends.postgresql \
    DD_DATABASE_HOST=127.0.0.1 \
    DD_DATABASE_PORT=5432 \
    DD_DATABASE_USER=defectdojo \
    DD_DATABASE_PASSWORD=defectdojo \
    DD_DATABASE_NAME=test_defectdojo \
    DD_TEST_DATABASE_NAME=test_defectdojo \
    DD_CELERY_BROKER_SCHEME=sqla+sqlite \
    DD_CELERY_BROKER_USER="" \
    DD_CELERY_BROKER_PASSWORD="" \
    DD_CELERY_BROKER_HOST="" \
    DD_CELERY_BROKER_PORT=-1 \
    DD_CELERY_BROKER_PATH=/app/dojo.celerydb.sqlite \
    DD_CELERY_BROKER_PARAMS="" \
    DD_JIRA_EXTRA_ISSUE_TYPES=Vulnerability \
    DD_SECRET_KEY=oss-scanner-local-build-only \
    DD_CREDENTIAL_AES_256_KEY=0123456789abcdef0123456789abcdef \
    DD_DATABASE_READINESS_TIMEOUT=120

RUN service postgresql start \
 && su postgres -c "psql -v ON_ERROR_STOP=1 -c \"CREATE USER defectdojo WITH PASSWORD 'defectdojo';\"" \
 && su postgres -c "psql -v ON_ERROR_STOP=1 -c \"CREATE DATABASE test_defectdojo OWNER defectdojo;\"" \
 && /wait-for-it.sh 127.0.0.1:5432 -t 60 -- bash -c 'cd /app && python3 manage.py migrate --noinput' \
 && ( /wait-for-it.sh 127.0.0.1:5432 -t 60 -- /entrypoint-unit-tests.sh \
      || echo "WARNING: unit tests reported failures or timed out" )

USER root
ENTRYPOINT []
CMD ["bash"]
