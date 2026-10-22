FROM node:24-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    CI=true \
    NODE_ENV=test

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        git \
        python3 \
        make \
        g++ \
    && rm -rf /var/lib/apt/lists/*

# The scanner requires the checked-out repository at /src.
COPY . /src
WORKDIR /src

# Fetch all dependencies while network access is available.
RUN npm ci

# Local configuration prevents packaging from reading AWS Parameter Store.
RUN cp .env.example .env \
    && npx serverless package --stage local

# Keep the deterministic, service-independent checks in the image and verify
# that the source can be reviewed without production credentials or Redis.
RUN npm run lint:check \
    && npm run test:unit
