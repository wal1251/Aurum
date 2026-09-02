FROM node:22-alpine AS build
WORKDIR /app
COPY package.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:1.27-alpine
# openssl generates the htpasswd hash for optional basic auth — see
# docker-entrypoint.d/20-basic-auth.sh.
RUN apk add --no-cache openssl
COPY --from=build /app/dist /usr/share/nginx/html
# templates/*.template are rendered to conf.d/*.conf by nginx's own
# docker-entrypoint.d/20-envsubst-on-templates.sh before nginx starts, which
# is how AURUM_BACKEND_HOST/PORT below reach the proxy_pass in the config.
COPY templates/default.conf.template /etc/nginx/templates/default.conf.template
COPY docker-entrypoint.d/20-basic-auth.sh /docker-entrypoint.d/20-basic-auth.sh
# Defaults that make docker-compose's fixed "backend" DNS name keep working
# unchanged; override per-environment (e.g. Railway's *.railway.internal)
# via the AURUM_BACKEND_HOST/AURUM_BACKEND_PORT service variables.
# NGINX_ENTRYPOINT_LOCAL_RESOLVERS opts into the base image's
# docker-entrypoint.d/15-local-resolvers.envsh, which exports
# NGINX_LOCAL_RESOLVERS from /etc/resolv.conf for the "resolver" directive in
# templates/default.conf.template — required for the dynamic (request-time)
# DNS lookup that keeps nginx from crashing if AURUM_BACKEND_HOST isn't
# resolvable yet at container startup.
ENV AURUM_BACKEND_HOST=backend \
    AURUM_BACKEND_PORT=8000 \
    NGINX_ENTRYPOINT_LOCAL_RESOLVERS=1
# sed strips any stray CR: cloned on Windows with core.autocrlf=true the
# script arrives as CRLF, the kernel reads the shebang as "/bin/sh\r", and
# nginx dies on boot with "not found" (exit 127). .gitattributes fixes this
# at the repository level; this line covers checkouts that predate it.
RUN sed -i 's/\r$//' /docker-entrypoint.d/20-basic-auth.sh \
    && chmod +x /docker-entrypoint.d/20-basic-auth.sh \
    && touch /etc/nginx/basic-auth.conf

EXPOSE 80

HEALTHCHECK --interval=10s --timeout=5s --start-period=5s --retries=5 \
    CMD wget -qO- http://127.0.0.1:80/api/health || exit 1
