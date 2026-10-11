FROM node:24-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    PLAYWRIGHT_BROWSERS_PATH=/opt/playwright-browsers

# The complete checkout includes the real trace ZIPs in test/assets.
COPY . /src
WORKDIR /src

# Fetch dev dependencies and the lockfile-matched browser while networking exists.
# Keep browser binaries and OS libraries available for offline tests/rendering.
RUN npm ci --include=dev \
    && ./node_modules/.bin/playwright install --with-deps chromium \
    && npm run build

CMD ["/bin/bash"]
