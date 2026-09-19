FROM dunglas/frankenphp:1-php8.5

RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

# Required and commonly used Pimcore PHP extensions. Several core extensions
# (including fileinfo, iconv, OpenSSL and XML support) are already present in
# the upstream PHP image.
RUN install-php-extensions \
    amqp \
    apcu \
    bcmath \
    curl \
    exif \
    gd \
    imagick \
    intl \
    mbstring \
    opcache \
    pcntl \
    pdo_mysql \
    redis \
    soap \
    sockets \
    zip

# Composer and basic development tooling. Add optional Pimcore media binaries
# only when the application actually uses the corresponding features.
RUN apt-get update \
    && apt-get install -y --no-install-recommends git locales-all unzip \
    && rm -rf /var/lib/apt/lists/*

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
COPY conf.d/zz-pimcore.ini "$PHP_INI_DIR/conf.d/zz-pimcore.ini"

WORKDIR /var/www/html
