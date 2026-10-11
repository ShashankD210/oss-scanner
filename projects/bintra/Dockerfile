# This file must set up the environment, install your project's dependencies and build the project.
# The scanner builds it with your repository as the build context, so `COPY . /src` puts the checkout inside
# the image. Network access is available during this Dockerfile build and is removed after this point, so
# all code that an agent might need to perform its tests must be fetched here.

FROM node:24.15
LABEL org.opencontainers.image.authors="Kai Kretschmann"

RUN apt-get update
RUN apt-get install -y gnupg2 iputils-ping

RUN wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | gpg --dearmor > /usr/share/keyrings/mongodb.gpg
RUN echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" > /etc/apt/sources.list.d/mongodb-org-7.0.list
RUN apt-get update && apt-get install -y mongodb-org mongodb-org-database mongodb-org-server mongodb-org-shell mongodb-org-mongos mongodb-org-tools

# Add Docker's official GPG key:
RUN apt-get install -y ca-certificates curl && install -m 0755 -d /etc/apt/keyrings && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && chmod a+r /etc/apt/keyrings/docker.asc
RUN echo "deb [ arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian bookworm stable" >/etc/apt/sources.list.d/docker.list
RUN apt-get update
RUN apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

RUN npm install -g npm@12.0.2

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# --- example for a CMake project; replace with your build ---
# Build the project as its maintainers build it, with debug info (-g) and frame pointers. A step that fails fails the
# build, and primary_contact gets the log.
RUN npm install
RUN NODE_ENV=test npm run pretest || echo "WARNING: simple unit test failed"

