FROM php:8.5.12-fpm-alpine3.24

ENV TZ=Asia/Shanghai \
    LANG=C.UTF-8

# ---------------------------------------------------------------------------
# 构建期依赖（编译完即删除）：工具链 + 各扩展的 -dev 头文件
# ---------------------------------------------------------------------------
ARG BUILD_DEPS="autoconf \
    automake \
    binutils \
    dpkg-dev \
    file \
    g++ \
    gcc \
    libc-dev \
    libtool \
    linux-headers \
    make \
    pax-utils \
    pkgconf \
    re2c \
    curl-dev \
    freetype-dev \
    gettext-dev \
    icu-dev \
    imagemagick-dev \
    libjpeg-turbo-dev \
    libpng-dev \
    libwebp-dev \
    libxml2-dev \
    libzip-dev \
    oniguruma-dev \
    openssl-dev \
    pcre2-dev \
    sqlite-dev"

# ---------------------------------------------------------------------------
# 运行时依赖（长期保留）
#   icu-data-full : Alpine 3.16+ 的 icu-libs 默认只带 icu-data-en，intl 做多语言
#                   格式化会退化成英文；只做英文站点可换成 icu-data-en 省 ~20MB
#   imagemagick   : imagick 扩展运行时需要 MagickWand 库；若只需要扩展不需要
#                   convert/magick 命令，可换成 imagemagick-libs 进一步瘦身
#   git / unzip   : composer 拉包时会用到，不需要可移入 BUILD_DEPS 一起删掉
# ---------------------------------------------------------------------------
ARG RUN_DEPS="bash \
    ca-certificates \
    curl \
    git \
    icu-data-full \
    imagemagick \
    tzdata \
    unzip"

# ---------------------------------------------------------------------------
# 系统包
# ---------------------------------------------------------------------------
RUN --mount=type=cache,target=/var/cache/apk,sharing=locked \
    set -eux; \
    apk update; \
    apk add ${RUN_DEPS}; \
    apk add ${BUILD_DEPS}

# ---------------------------------------------------------------------------
# 用户：busybox 的 addgroup/adduser 没有 -o 选项，用 grep 判断是否存在
# ---------------------------------------------------------------------------
ARG WWW_UID=1000
ARG WWW_GID=1000
RUN set -eux; \
    nologin=/sbin/nologin; command -v nologin >/dev/null 2>&1 || nologin=/bin/false; \
    if ! grep -q "^www:" /etc/group; then addgroup -g "${WWW_GID}" www; fi; \
    if ! grep -q "^www:" /etc/passwd; then \
        adduser -u "${WWW_UID}" -G www -S -D -H -s "${nologin}" -h /web/wwwroot www; \
    fi; \
    id www

# ---------------------------------------------------------------------------
# 内置扩展
# ---------------------------------------------------------------------------
RUN set -eux; \
    docker-php-ext-configure gd --with-jpeg --with-freetype --with-webp; \
    docker-php-ext-install -j"$(nproc)" \
        pdo_mysql mysqli bcmath intl zip gd pcntl exif gettext

# ---------------------------------------------------------------------------
# 其他扩展（PIE）
#   说明：Alpine 里没有 sudo，--auto-install-build-tools 反而会失败；
#        这里已手动装齐工具链，所以改用 --no-build-tools-check 跳过检查。
#        --no-system-dependencies-check 同理（PIE 只认 Debian 包名）。
# ---------------------------------------------------------------------------
ARG REDIS_EXT=phpredis/phpredis:6.3.0
ARG IMAGICK_EXT=imagick/imagick:3.8.1
ARG YAF_EXT=laruence/yaf:^3.3.8
ARG YAC_EXT=laruence/yac:^3.0
ARG XDEBUG_EXT=xdebug/xdebug:^3.5

RUN --mount=type=bind,from=ghcr.io/php/pie:1-bin,source=/pie,target=/usr/local/bin/pie \
    set -eux; \
    pie -V; \
    pie install --no-cache -n --no-build-tools-check --no-system-dependencies-check "${REDIS_EXT}"; \
    pie install --no-cache -n --no-build-tools-check --no-system-dependencies-check "${IMAGICK_EXT}"; \
    pie install --no-cache -n --no-build-tools-check --no-system-dependencies-check "${YAF_EXT}"; \
    pie install --no-cache -n --no-build-tools-check --no-system-dependencies-check "${YAC_EXT}"; \
    pie install --no-cache -n --no-build-tools-check --no-system-dependencies-check "${XDEBUG_EXT}"; \
    rm -rf /root/.pie /root/.composer /tmp/pie

COPY --from=composer:2.8 /usr/bin/composer /usr/bin/composer

# ---------------------------------------------------------------------------
# 目录：socket 目录 / 日志目录 / web 根
# ---------------------------------------------------------------------------
RUN mkdir -p /web/wwwroot /var/log/php /var/log/xdebug \
    && chown -R www:www /web/wwwroot /var/log/php /var/log/xdebug

# ---------------------------------------------------------------------------
# 清理：
#   1) scanelf 扫出扩展 .so 实际链接的系统库 -> 安装为运行时依赖（防止 apk del 误删）
#   2) 删除构建期依赖（工具链 + dev 包）
#   3) 删除 PHP 源码、静态库、libtool 归档
#   4) 清空各类缓存
# ---------------------------------------------------------------------------
RUN set -eux; \
    ext_dir="$(php -r 'echo ini_get("extension_dir");')"; \
    run_deps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive "${ext_dir}" \
        | tr ',' '\n' \
        | sort -u \
        | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )"; \
    echo "runtime deps: ${run_deps}"; \
    apk add --no-cache ${run_deps}; \
    apk del --no-network ${BUILD_DEPS}; \
    docker-php-source delete; \
    rm -rf /usr/src/php*; \
    find /usr/local -name '*.a' -delete; \
    find /usr/local -name '*.la' -delete; \
    rm -rf /usr/share/man /usr/share/doc; \
    rm -rf /var/cache/apk/* /tmp/* /var/tmp/* \
           /root/.cache /root/.composer /root/.pie /root/.pearrc; \
    php -v; \
    php -m

USER root
WORKDIR /web/wwwroot

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD test -S /dev/shm/php-fpm-8.5.sock || exit 1

CMD ["php-fpm"]
