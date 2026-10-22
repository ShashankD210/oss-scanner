# The Otzaria website is a Next.js app backed by MongoDB.
FROM node:24-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    NEXT_TELEMETRY_DISABLED=1 \
    CI=true
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl gnupg python3 \
    && rm -rf /var/lib/apt/lists/*

# A local MongoDB, so the server can run offline (MONGODB_URI defaults to mongodb://127.0.0.1:27017/otzaria_db).
RUN curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | gpg --dearmor -o /usr/share/keyrings/mongodb-server-8.0.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg] https://repo.mongodb.org/apt/debian bookworm/mongodb-org/8.0 main" \
       > /etc/apt/sources.list.d/mongodb-org-8.0.list \
    && apt-get update && apt-get install -y --no-install-recommends mongodb-org-server \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Same install as the deploy workflow, which runs the plugin validator from its moving v1 tag.
RUN npm ci --legacy-peer-deps \
    && npm install "github:Otzaria/otzaria-plugin-validator#v1" --legacy-peer-deps --no-save

# Placeholder secrets for a local run only.
ENV NEXTAUTH_URL=http://localhost:3000 \
    NEXTAUTH_SECRET=oss-scanner-local-only \
    TRUSTED_PROXY_COUNT=0

RUN npm run build || echo "WARNING: npm run build failed"

# Run the tests so we know the image works; failures do not abort the build.
RUN (npm test || echo "WARNING: npm test reported failures")

# To exercise the server: mongod --fork --logpath /tmp/mongod.log --dbpath /tmp/db (after mkdir -p /tmp/db), then npm start.
