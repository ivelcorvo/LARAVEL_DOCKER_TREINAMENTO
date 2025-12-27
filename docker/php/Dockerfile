FROM php:8.3-fpm-alpine

# Dependências do sistema e extensões PHP exigidas pelo Laravel 12
RUN apk add --no-cache \
    git \
    unzip \
    libpq-dev \
    oniguruma-dev \
    icu-dev \
    $PHPIZE_DEPS \
    && docker-php-ext-install \
        pdo \
        pdo_pgsql \
        mbstring \
        intl \
    && rm -rf /var/cache/apk/*

# Composer (oficial)
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Diretório da aplicação
WORKDIR /var/www/html

# Criar usuário não-root
RUN addgroup -g 1000 appuser \
    && adduser -G appuser -u 1000 -D appuser

# Garantir permissões para Laravel (storage e cache)
RUN chown -R appuser:appuser /var/www/html

# Trocar para usuário não-root
USER appuser
