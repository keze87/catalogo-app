# Etapa 1: Builder
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Etapa 2: Servidor web sin privilegios
FROM nginxinc/nginx-unprivileged:1.27-alpine
WORKDIR /usr/share/nginx/html

ENV API_HOST=catalogo-api \
    API_PORT=8000 \
    NGINX_ENVSUBST_FILTER=^API_

COPY --from=builder /app/dist .
COPY nginx/templates/default.conf.template /etc/nginx/templates/default.conf.template

EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]