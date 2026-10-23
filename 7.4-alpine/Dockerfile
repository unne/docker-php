FROM alpine:3.15 AS builder

# 1. Instalar paquetes de compilación y apache2
RUN apk add --no-cache \
    build-base \
    autoconf \
    bison \
    re2c \
    curl-dev \
    libxml2-dev \
    libxslt-dev \
    gmp-dev \
    oniguruma-dev \
    openssl-dev \
    postgresql-dev \
    libsodium-dev \
    libzip-dev \
    libpng-dev \
    jpeg-dev \
    freetype-dev \
    libmemcached-dev \
    zlib-dev \
    git \
    pkgconf \
    sqlite-dev \
    curl \
    tar \
    unzip \
    apache2 \
    apache2-dev

WORKDIR /build

ENV CFLAGS="-O2 -DOPENSSL_API_COMPAT=0x10100000L"

# 2. Compilar PHP 7.4 y memcached en un SOLO paso RUN
RUN curl -L -o php-src.zip "https://github.com/php/php-src/archive/refs/heads/PHP-7.4-CVE-2026-109618-109619.zip" && \
    curl -L -o memcached.tgz "https://pecl.php.net/get/memcached-3.2.0.tgz" && \
    unzip php-src.zip && \
    tar -xzf memcached.tgz && \
    rm php-src.zip memcached.tgz && \
    mv php-src-* php-src-dir && \
    cd php-src-dir && \
    ./buildconf --force && \
    EXTENSION_DIR=/usr/lib/php7/modules ./configure \
    --prefix=/usr \
    --program-suffix=7 \
    --libdir=/usr/lib/php7 \
    --datadir=/usr/share/php7 \
    --includedir=/usr/include/php7 \
    --sysconfdir=/etc/php7 \
    --localstatedir=/var \
    --mandir=/usr/share/man \
    --with-config-file-path=/etc/php7 \
    --with-config-file-scan-dir=/etc/php7/conf.d \
    --with-apxs2=/usr/bin/apxs \
    --enable-option-checking=fatal \
    --enable-ctype \
    --with-curl \
    --enable-dom \
    --enable-exif \
    --enable-fileinfo \
    --enable-gd \
    --with-jpeg \
    --with-freetype \
    --with-gmp \
    --with-iconv \
    --enable-mbstring \
    --with-openssl \
    --enable-pcntl \
    --with-pdo-pgsql \
    --enable-phar \
    --enable-session \
    --enable-simplexml \
    --with-sodium \
    --enable-tokenizer \
    --with-zip \
    --enable-xml \
    --enable-xmlwriter \
    --with-xsl \
    --enable-cli && \
    make -j$(nproc) && \
    make install && \
    mkdir -p /php-root/etc/apache2 && \
    cp /etc/apache2/httpd.conf /php-root/etc/apache2/httpd.conf && \
    make INSTALL_ROOT=/php-root install && \
    rm -rf /php-root/etc/apache2 /php-root/var && \
    mkdir -p /php-root/etc/php7/conf.d && \
    cp php.ini-production /php-root/etc/php7/php.ini && \
    ln -sf php7 /php-root/usr/bin/php && \
    cd /build/memcached-3.2.0 && \
    /usr/bin/phpize7 && \
    ./configure --with-php-config=/usr/bin/php-config7 && \
    make -j$(nproc) && \
    make INSTALL_ROOT=/php-root install && \
    echo "extension=memcached.so" > /php-root/etc/php7/conf.d/20_memcached.ini


FROM alpine:3.15 AS cli

# Por defecto se utiliza la timezone de Buenos Aires
ENV TZ=America/Argentina/Buenos_Aires

RUN apk --no-cache add \
    bash \
    tzdata \
    libsodium \
    libxml2 \
    libxslt \
    gmp \
    oniguruma \
    libcurl \
    libssl1.1 \
    libcrypto1.1 \
    libpq \
    libzip \
    libpng \
    libjpeg-turbo \
    freetype \
    libmemcached-libs \
    sqlite-libs \
    zlib \
    curl \
    && rm -rf /var/cache/apk/*
COPY --from=builder /php-root/ /
RUN echo "date.timezone  = $TZ" > /etc/php7/conf.d/80-siu-timezone.ini
COPY common/siu-entrypoint.d/* /siu-entrypoint.d/
COPY common/entrypoint.sh /usr/local/bin/entrypoint.sh

ENTRYPOINT ["entrypoint.sh"]

FROM cli AS web

RUN apk --no-cache add \
    apache2 \
    && rm -rf /var/cache/apk/* \
    && mkdir -p /var/log/apache2 \
    && ln -s /dev/stderr /var/log/apache2/error.log \
    && ln -s /dev/stdout /var/log/apache2/access.log \
    #fix warning https://stackoverflow.com/a/56645177"
    && sed -i 's/^Listen 80$/Listen 0.0.0.0:8080/' /etc/apache2/httpd.conf \
    && printf '%s\n' \
    'LoadModule php7_module modules/libphp7.so' \
    'DirectoryIndex index.php index.html' \
    '<FilesMatch \.php$>' \
    '    SetHandler application/x-httpd-php' \
    '</FilesMatch>' > /etc/apache2/conf.d/php7-module.conf
COPY common/apache/* /etc/apache2/conf.d/
COPY common/php/* /etc/php7/conf.d/

CMD ["httpd", "-D", "FOREGROUND"]

EXPOSE 8080

FROM web AS rootless

ARG USER=siu
ENV USER=$USER
ARG UID=222

RUN adduser -h /usr/local/app -u $UID -G www-data -S $USER \
    && chown -R $USER.0 \
    /var/run/apache2 \
    /var/log/apache2 \
    /etc/php7/conf.d \
    /etc/apache2/conf.d \
    /var/www/localhost/htdocs \
    /siu-entrypoint.d \
    && chmod -R g=u \
    /etc/passwd \
    /var/run/apache2 \
    /var/log/apache2 \
    /etc/php7/conf.d \
    /etc/apache2/conf.d \
    /var/www/localhost/htdocs \
    /siu-entrypoint.d
USER $USER
