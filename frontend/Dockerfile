# ---- build stage ----
FROM node:16-alpine AS build

ENV NODE_ENV=production
WORKDIR /app

# Install deps (cache layer)
COPY package*.json ./
RUN npm ci && npm cache clean --force

# Accept API URL at build time
ARG VUE_APP_API_URL=http://localhost:8081
ENV VUE_APP_API_URL=${VUE_APP_API_URL}

# Copy sources and build
COPY . .
RUN npm run build

# ---- runtime stage ----
FROM nginx:1.25-alpine

# Security hardening
RUN adduser -D -g 'www' www && \
    mkdir -p /var/cache/nginx /var/run/nginx /var/log/nginx && \
    chown -R www:www /var/cache/nginx /var/run/nginx /var/log/nginx

# Minimal nginx config tuned for SPA
COPY --chown=www:www nginx.conf /etc/nginx/nginx.conf

# Copy built assets
COPY --from=build /app/dist /usr/share/nginx/html

USER www

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD wget -qO- http://127.0.0.1/ >/dev/null 2>&1 || exit 1

CMD ["nginx", "-g", "daemon off;"]
