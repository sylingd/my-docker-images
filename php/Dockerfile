FROM php:8.5.12-fpm

ENV TZ=Asia/Shanghai LANG=C.UTF-8 DEBIAN_FRONTEND=noninteractive

# 1) apt（最稳定的层放最前），加 cache mount
RUN --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    --mount=type=cache,target=/var/cache/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
    git curl unzip ca-certificates pkg-config libzip-dev libpng-dev libjpeg62-turbo-dev \
    libfreetype6-dev libwebp-dev libicu-dev libcurl4-openssl-dev libssl-dev libxml2-dev \
    libonig-dev libsqlite3-dev libmagickwand-dev imagemagick

# 2) 用户：ARG 只在这里生效
ARG WWW_UID=1000
ARG WWW_GID=1000
RUN groupadd -o -g "${WWW_GID}" www 2>/dev/null || true; \
    id -u www >/dev/null 2>&1 || useradd -o -u "${WWW_UID}" -g "${WWW_GID}" \
        -s /usr/sbin/nologin -M -d /web/wwwroot www; id www

# 3) 内置扩展（耗时最长，参数改动不应影响它）
RUN set -eux; \
    docker-php-ext-configure gd --with-jpeg --with-freetype --with-webp; \
    docker-php-ext-install -j"$(nproc)" pdo_mysql mysqli bcmath intl zip gd pcntl exif gettext; \
    docker-php-source delete

# 4) PECL → 换 install-php-extensions，版本写死
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/local/bin/

ARG EXT_IMAGICK=3.8.1
ARG EXT_REDIS=6.2.0
ARG EXT_YAF=3.3.8
ARG EXT_YAC=3.0.1
RUN install-php-extensions redis-${EXT_REDIS} imagick-${EXT_IMAGICK} yaf-${EXT_YAF} yac-${EXT_YAC}

# 5) Xdebug 开关：ARG 紧贴使用点，改它只重建这一层
ARG INSTALL_XDEBUG=true
RUN --mount=type=cache,target=/tmp/pear \
    if [ "${INSTALL_XDEBUG}" = "true" ]; then \
        install-php-extensions xdebug-3.5.3; \
    fi

COPY --from=composer:2.8 /usr/bin/composer /usr/bin/composer

# 6) 目录：socket 目录 / 日志目录 / web 根
RUN mkdir -p /web/wwwroot /var/log/php /var/log/xdebug \
    && chown -R www:www /web/wwwroot /var/log/php /var/log/xdebug

USER root
WORKDIR /web/wwwroot

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD test -S /dev/shm/php-fpm-8.5.sock || exit 1

CMD ["php-fpm"]
