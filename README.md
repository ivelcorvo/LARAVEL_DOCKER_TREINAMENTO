# Passo a passo — Laravel + Postgres + Docker (moderno)

## 1) Estrutura de pastas

Crie a pasta do projeto e a estrutura:

```
meu_backend (nome do seu projeto backend)/
├─ docker-compose.yml
├─ docker/
│  ├─ php/
│  │  └─ Dockerfile
│  └─ nginx/
│     └─ default.conf
└─ backend/     (vazia — o Laravel entra aqui)
```

## 2) Arquivos

### 2.1 docker/php/Dockerfile

#### opção 1) mais simples

```dockerfile
FROM php:8.3-fpm-alpine

RUN apk add --no-cache git unzip libpq-dev $PHPIZE_DEPS && docker-php-ext-install pdo pdo_pgsql

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

# Usuário não-root (boa prática para dev)
RUN addgroup -g 1000 appuser && adduser -G appuser -u 1000 -D appuser
USER appuser
```

#### opção 2) mais profissional

OBS: A pasta backend deve estar 100% vazia ANTES de rodar o create-project.  
esta opção preenche está pasta então vc precisa apagar

```dockerfile
FROM php:8.3-fpm-alpine

# Dependências do sistema e extensões PHP exigidas pelo Laravel 12
RUN apk add --no-cache     git     unzip     libpq-dev     oniguruma-dev     icu-dev     $PHPIZE_DEPS     && docker-php-ext-install         pdo         pdo_pgsql         mbstring         intl     && rm -rf /var/cache/apk/*

# Composer (oficial)
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Diretório da aplicação
WORKDIR /var/www/html

# Criar usuário não-root
RUN addgroup -g 1000 appuser     && adduser -G appuser -u 1000 -D appuser

# Garantir permissões para Laravel (storage e cache)
RUN chown -R appuser:appuser /var/www/html

# Trocar para usuário não-root
USER appuser
```

### 2.2 docker/nginx/default.conf

```nginx
server {
    listen 80;
    server_name localhost;
    root /var/www/html/public;

    index index.php index.html;
    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app_NOME:9000;                # PRECISA SUBISTITUIR O NOME (MESMO NOME DO CONTAINER LARAVEL APP)
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### 2.3 docker-compose.yml

Remova qualquer linha version: (obsoleta).

```yaml
services:
  db:
    image: postgres:15-alpine
    container_name: #nome container banco ex:nome_postgres_db 
    restart: unless-stopped
    environment:
      # TROCAS OS VALORES DESSAS CHAVES
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: secret
    volumes:
      - nome_dbdata:/var/lib/postgresql/data # nome precisa ser o mesmo do final no arquivo em volumes
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"] # -U precisa ser o mesmo usuario usado em POSTGRES_USER (appuser no caso)
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
    container_name: #nome container aplicação ex: nome_laravel_app
    restart: unless-stopped
    volumes:
      - ./backend:/var/www/html
    depends_on:
      db:
        condition: service_healthy

  nginx:
    image: nginx:alpine
    container_name: #nome container nginx ex: teste_nginx_proxy
    restart: unless-stopped
    ports:
      - "8080:80"     # se der conflito, troque para "8081:80"
    volumes:
      - ./backend:/var/www/html
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app

volumes:
  nome_dbdata: # mudar o nome para evitar conflito (USAR O MESMO NOME ACIMA EM DB)
```

## 3) Subir o ambiente

Na raiz do projeto (onde está o docker-compose.yml):

```bash
docker compose up -d --build
```

## 4) Instalar o Laravel (dentro do container)

```bash
docker compose exec app composer create-project laravel/laravel .
```

## 5) Configurar o banco no .env

```env
DB_CONNECTION=pgsql
DB_HOST=db
DB_PORT=5432
DB_DATABASE=appdb
DB_USERNAME=appuser
DB_PASSWORD=secret
```

## 6) Finalizar setup do Laravel

```bash
docker compose exec app php artisan key:generate
docker compose exec app php artisan migrate
docker compose exec app chmod -R 775 storage bootstrap/cache
```

## 7) Testar

http://localhost:8080

================================================================================================================

depois clonar do github

================================================================================================================

🟩 ETAPA 1 — Criar o arquivo .env

PowerShell:
copy backend\.env.example backend\.env

🟩 ETAPA 2 — Editar o .env

DB_CONNECTION=pgsql
DB_HOST=db
DB_PORT=5432
DB_DATABASE=expert_appdb
DB_USERNAME=LEVIappuser
DB_PASSWORD=secret

🟩 ETAPA 3 — Subir Docker
docker compose up -d --build

🟩 ETAPA 4 — Instalar dependências
docker compose exec app composer install

🟩 ETAPA 5 — Gerar chave
docker compose exec app php artisan key:generate

🟩 ETAPA 6 — Migrar
docker compose exec app php artisan migrate
