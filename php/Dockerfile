FROM php:8.5.12-fpm

ENV TZ=Asia/Shanghai LANG=C.UTF-8 DEBIAN_FRONTEND=noninteractive

# apt
RUN --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    --mount=type=cache,target=/var/cache/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
    git curl unzip ca-certificates pkg-config libzip-dev libpng-dev libjpeg62-turbo-dev \
    libfreetype6-dev libwebp-dev libicu-dev libcurl4-openssl-dev libssl-dev libxml2-dev \
    libonig-dev libsqlite3-dev libmagickwand-dev imagemagick libtool

# 用户
ARG WWW_UID=1000
ARG WWW_GID=1000
RUN groupadd -o -g "${WWW_GID}" www 2>/dev/null || true; \
    id -u www >/dev/null 2>&1 || useradd -o -u "${WWW_UID}" -g "${WWW_GID}" \
        -s /usr/sbin/nologin -M -d /web/wwwroot www; id www

# 内置扩展
RUN set -eux; \
    docker-php-ext-configure gd --with-jpeg --with-freetype --with-webp; \
    docker-php-ext-install -j"$(nproc)" pdo_mysql mysqli bcmath intl zip gd pcntl exif gettext; \
    docker-php-source delete

# 其他扩展
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/local/bin/

ARG EXT_IMAGICK=3.8.1
ARG EXT_REDIS=6.2.0
ARG EXT_YAF=3.3.8
ARG EXT_YAC=3.0.1
RUN install-php-extensions redis-${EXT_REDIS} imagick-${EXT_IMAGICK} yaf-${EXT_YAF} yac-${EXT_YAC}

ARG REDIS_EXT=phpredis/phpredis:6.3.0
ARG IMAGICK_EXT=imagick/imagick:3.8.1
ARG YAF_EXT=laruence/yaf:^3.3.8
ARG YAC_EXT=laruence/yac:^3.0
ARG XDEBUG_EXT=xdebug/xdebug:^3.5

RUN --mount=type=bind,from=ghcr.io/php/pie:1-bin,source=/pie,target=/usr/local/bin/pie \
    set -eux; \
    pie -V; \
    pie install --no-cache -n --auto-install-build-tools --no-system-dependencies-check "${REDIS_EXT}"; \
    pie install --no-cache -n --auto-install-build-tools --no-system-dependencies-check "${IMAGICK_EXT}"; \
    pie install --no-cache -n --auto-install-build-tools --no-system-dependencies-check "${YAF_EXT}"; \
    pie install --no-cache -n --auto-install-build-tools --no-system-dependencies-check "${YAC_EXT}"; \
    pie install --no-cache -n --auto-install-build-tools --no-system-dependencies-check "${XDEBUG_EXT}"; \
    rm -rf /root/.pie /root/.composer

COPY --from=composer:2.8 /usr/bin/composer /usr/bin/composer

# 目录：socket 目录 / 日志目录 / web 根
RUN mkdir -p /web/wwwroot /var/log/php /var/log/xdebug \
    && chown -R www:www /web/wwwroot /var/log/php /var/log/xdebug

USER root
WORKDIR /web/wwwroot

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD test -S /dev/shm/php-fpm-8.5.sock || exit 1

CMD ["php-fpm"]
