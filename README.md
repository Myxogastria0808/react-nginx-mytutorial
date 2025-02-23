# react-nginx-mytutorial

## Dockerfile

```Dockerfile
FROM node:22.14.0-alpine3.21 AS node

# Install dependencies only when needed
FROM node AS deps
WORKDIR /app

# Install dependencies based on the preferred package manager
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
    if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
    elif [ -f package-lock.json ]; then npm ci; \
    elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm i --frozen-lockfile; \
    else echo "Lockfile not found." && exit 1; \
    fi


# Rebuild the source code only when needed
FROM node AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN \
    if [ -f yarn.lock ]; then yarn run build; \
    elif [ -f package-lock.json ]; then npm run build; \
    elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm run build; \
    else echo "Lockfile not found." && exit 1; \
    fi

# Production image, copy all the files and run react
FROM nginx:alpine AS runner
COPY ./nginx/nginx.conf /etc/nginx/nginx.conf
COPY --from=builder /app/dist /usr/share/nginx/html

ENTRYPOINT ["nginx", "-g", "daemon off;"]
EXPOSE 80
services:
  react-nginx-mytutorial:
    image: react-nginx-mytutorial
    container_name: client
    build:
      context: .
      dockerfile: ./Dockerfile
    ports:
      - "3000:80"

```

## dev.compose.yaml

```yaml
services:
  react-nginx-mytutorial:
    image: react-nginx-mytutorial
    container_name: client
    build:
      context: .
      dockerfile: ./Dockerfile
    ports:
      - "3000:80"
```

## prod.compose.yaml

```yaml
services:
  react-nginx-mytutorial:
    image: ghcr.io/myxogastria0808/react-nginx-mytutorial/client:main
    container_name: react-nginx-mytutorial
    build:
      context: .
      dockerfile: ./Dockerfile
    ports:
      - "3000:80"
```

## nginx.conf

```nginx.conf
user nginx;
worker_processes    auto;

events {
    worker_connections  1024;
    multi_accept on;
}

http {
    server {
        listen 80;
        server_name   localhost;
        charset UTF-8;
        include /etc/nginx/mime.types;

        gzip on;
        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 1;
        gzip_min_length 1024;
        gzip_types application/atom+xml
            application/javascript
            application/json
            application/rss+xml
            application/vnd.ms-fontobject
            application/x-font-ttf
            application/x-web-app-manifest+json
            application/xhtml+xml
            application/xml
            font/opentype
            image/svg+xml
            image/x-icon
            text/css
            text/plain
            text/x-component;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
}
```