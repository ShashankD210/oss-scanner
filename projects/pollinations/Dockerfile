FROM node:24-bookworm

ENV CI=true \
    WRANGLER_SEND_METRICS=false \
    TEST_ENV=test \
    TEST_VCR_MODE=replay-only

# The scanner supplies a fresh public checkout, never a maintainer's workspace.
COPY . /src
WORKDIR /src

RUN npm ci --no-audit --no-fund \
    && npm ci --prefix packages/mcp --no-audit --no-fund

# Compile both Workers and the dashboard without deploying or loading secrets.
RUN npm run build:staging --workspace pollinations-enter \
    && npm run build:staging --workspace pollinations-gen

# Exercise the contributor bootstrap without SOPS. Tests use the committed
# env.test bindings, so remove the generated example overrides afterwards.
RUN npm run decrypt-vars --workspace pollinations-enter \
    && rm /src/enter.pollinations.ai/.dev.vars

CMD ["bash"]
