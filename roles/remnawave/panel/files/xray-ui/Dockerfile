# syntax=docker/dockerfile:1
# Многостадийная сборка xray-config-ui-editor: статики + angie.
# В финальный образ попадает только dist/ и конфиг angie.

ARG ANGIE_IMAGE=docker.angie.software/angie:1.10.1-debian
ARG XRAY_UI_RELEASE=main

# --- stage 1: сборка статики -------------------------------------------
FROM oven/bun:1 AS build
ARG XRAY_UI_RELEASE

RUN apt-get update \
 && apt-get install -y --no-install-recommends git ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# Релиз = тег или ветка; если такого ref нет — собираем дефолтную ветку.
RUN git clone --depth 1 --branch "${XRAY_UI_RELEASE}" \
      https://github.com/bropines/xray-config-ui-editor.git /src \
 || git clone --depth 1 \
      https://github.com/bropines/xray-config-ui-editor.git /src

WORKDIR /src
# --base переопределяет base из vite.config.ts ('/xray-config-ui-editor/'),
# PWA scope/start_url в конфиге выводятся из base автоматически.
RUN bun install \
 && bun run build --base=/xray-ui/

# --- stage 2: отдача статики -------------------------------------------
FROM ${ANGIE_IMAGE}

# Конфиг angie (/etc/angie/angie.conf) монтируется снаружи, в образе —
# только собранная статика.
COPY --from=build /src/dist /var/www/xray-ui

EXPOSE 8090
