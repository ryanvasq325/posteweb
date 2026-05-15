# ============================================================================
# Estágio 1: Build de assets frontend (Node.js)
# Vite 8.0.10 exige Node 20.19+ ou 22.12+ — usamos LTS atual (22)
# alpine3.21 alinhado com seu PHP em alpine3.23 (ambos musl libc compatíveis)
# ============================================================================
FROM node:22-alpine3.21 AS assets

WORKDIR /build

# Copia primeiro só metadados — maximiza cache de layer Docker.
# Se package.json não mudou, npm ci é reutilizado de builds anteriores.
COPY package.json package-lock.json ./

# npm ci (não npm install) — build determinístico a partir do lockfile.
# --no-audit --no-fund acelera CI sem comprometer segurança (rode 'npm audit'
# em pipeline separado se quiser auditar).
RUN npm ci --no-audit --no-fund

# Copia config + fontes
COPY vite.config.js ./
COPY resources ./resources

# Build de produção — Rolldown gera public/assets/ com hashes
RUN npm run build

# ============================================================================
# Estágio 2: Runtime PHP (Alpine + PHP-FPM 8.5.6)
# ============================================================================
FROM php:8.5.6-fpm-alpine3.22

ENV TZ=${TZ} \
    COMPOSER_ALLOW_SUPERUSER=1

RUN apk add --no-cache \
    bash \
    git \
    unzip \
    icu-dev \
    postgresql-dev \
    oniguruma-dev \
    tzdata

RUN apk add --no-cache --virtual .build-deps $PHPIZE_DEPS \
    freetype-dev libjpeg-turbo-dev libpng-dev \
    postgresql-dev icu-dev oniguruma-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install pdo_pgsql intl mbstring gd \
    && pecl install igbinary && docker-php-ext-enable igbinary \
    && pecl install redis    && docker-php-ext-enable redis \
    && pecl install pcov     && docker-php-ext-enable pcov \
    && apk add --no-cache --virtual .runtime-deps \
    libpq icu-libs oniguruma libjpeg-turbo libpng freetype \
    && apk del .build-deps \
    && rm -rf /tmp/pear ~/.pearrc /var/cache/apk/*

COPY docker/php/php.ini /usr/local/etc/php/conf.d/custom.ini
COPY --from=composer:2.9 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www

# Copia o código da aplicação PHP
COPY . .

# Copia APENAS os assets compilados do estágio 1 — sobrescreve public/assets/
# A imagem final NÃO contém Node, npm, /build/, nem resources/.
COPY --from=assets /build/public/assets ./public/assets

EXPOSE 9000
CMD ["php-fpm"]