# OSS Scanner — bancolombia/django-DefectDojo (#trunk)
# Build context = full repo clone. Network only during `docker build`.
FROM python:3.15.0rc3-slim-bookworm@sha256:ea93de1a70a03fffd10ad8cfde8f4ba969fe2658f0578af02855d1d32f49e6c2
ENV DEBIAN_FRONTEND=noninteractive \
    PYCURL_SSL_LIBRARY=openssl \
    PYTHONUNBUFFERED=1

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential gcc git nasm \
      libpq-dev postgresql postgresql-client \
      libopenjp2-7 libjpeg62 libtiff6 xmlsec1 uuid-runtime \
      libisal-dev \
      curl autoconf gettext zlib1g-dev libcurl4-openssl-dev \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src
RUN ln -sf /src /app

RUN CPUCOUNT=1 pip3 wheel --wheel-dir=/tmp/wheels -r requirements.txt \
 && pip3 install --no-cache-dir --no-index --find-links=/tmp/wheels -r requirements.txt

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
