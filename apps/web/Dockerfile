# syntax=docker.io/docker/dockerfile:1.22-labs@sha256:4c116b618ed48404d579b5467127b20986f2a6b29e4b9be2fee841f632db6a86
# Context must be the root of the monorepo

# Builder
FROM --platform=$BUILDPLATFORM node:24-bullseye@sha256:27e462f5db2402700867dfa8ec35e3a68b127fdf61b505db0dd6ab98c38284bb AS builder

# Support custom branch of the js-sdk. This also helps us build images of element-web develop.
ARG USE_CUSTOM_SDKS=false
ARG JS_SDK_REPO="https://github.com/matrix-org/matrix-js-sdk.git"
ARG JS_SDK_BRANCH="master"

WORKDIR /src

# Install dependencies
COPY --parents package.json pnpm-lock.yaml pnpm-workspace.yaml patches scripts **/package.json /src/
RUN corepack enable
# use local registry
#RUN pnpm config set registry http://raspi.home329:4873/ && pnpm config set fetch-retry-maxtimeout 120000 && pnpm config set network-concurrency 2
#конфигурируем зеркала PNPM и скрипт переключения между зеркалами
RUN pnpm config set registry https://registry.npmmirror.com/


RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    --mount=type=cache,target=/root/.npm \
    --mount=type=bind,source=.git,target=/src/.git /src/scripts/docker-link-repos.sh && \
    pnpm install

# Build
COPY --link --exclude=.git --exclude=apps/web/docker . /src
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    --mount=type=cache,target=/root/.npm \
    --mount=type=bind,source=.git,target=/src/.git /src/scripts/docker-package.sh

# Copy the config now so that we don't create another layer in the app image
RUN cp /src/apps/web/config.sample.json /src/apps/web/webapp/config.json

# App
FROM nginxinc/nginx-unprivileged:alpine-slim

# Need root user to install packages & manipulate the usr directory
USER root
#используем другое зеркало репозиториев
# Install jq and moreutils for sponge, both used by our entrypoints
RUN echo "https://mirror.selectel.ru/alpine/v3.21/main" > /etc/apk/repositories && \
    echo "https://mirror.selectel.ru/alpine/v3.21/community" >> /etc/apk/repositories && \
    apk update && apk add --no-cache jq moreutils

COPY --from=builder /src/apps/web/webapp /app

# Override default nginx config. Templates in `/etc/nginx/templates` are passed
# through `envsubst` by the nginx docker image entry point.
COPY /apps/web/docker/nginx-templates/* /etc/nginx/templates/
COPY /apps/web/docker/docker-entrypoint.d/* /docker-entrypoint.d/

RUN rm -rf /usr/share/nginx/html \
  && ln -s /app /usr/share/nginx/html

# Run as nginx user by default
USER nginx

# HTTP listen port
ENV ELEMENT_WEB_PORT=80

HEALTHCHECK --start-period=5s CMD wget -q --spider http://localhost:$ELEMENT_WEB_PORT/config.json


# Export stage - копирует собранные файлы для вывода на хост
FROM scratch AS export
COPY --from=builder /src/apps/web/webapp /output/