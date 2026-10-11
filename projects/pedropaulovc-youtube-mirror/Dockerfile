FROM node:24-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    HUSKY=0 \
    CI=true \
    WRANGLER_SEND_METRICS=false \
    PLAYWRIGHT_BROWSERS_PATH=/opt/playwright-browsers

COPY . /src
WORKDIR /src

# Include development tools, native workerd/esbuild packages, and Chromium for
# the Node-side provisioning/browser helpers; never run those live helpers here.
RUN npm ci --include=dev \
    && ./node_modules/.bin/playwright install --with-deps chromium

# Use the checked-in resource-neutral configs, NOT environment rendering or
# deployment scripts. As in npm run build, dry-run bundles every Worker without
# deploying. Separate output directories retain all six Worker bundles/maps.
RUN for config in wrangler.mirror-*.jsonc; do \
      name="${config#wrangler.}"; name="${name%.jsonc}"; \
      ./node_modules/.bin/wrangler deploy --config "$config" --dry-run --outdir "dist/$name" || exit 1; \
    done

CMD ["/bin/bash"]
