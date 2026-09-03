FROM node:15-alpine as builder
WORKDIR /app

# node-sass@5 ships no prebuilt binary for linux-musl-arm64, so it compiles
# libsass from source and needs a toolchain that node:15-alpine does not carry.
RUN apk add --no-cache python3 make g++

COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.21-alpine

# Provenance for GET /version. Supplied by the CD workflow as build args;
# the defaults keep local builds (k8s, docker run) working unchanged.
ARG GIT_SHA=unknown
ARG RUN_ID=unknown

WORKDIR /usr/share/nginx/html
COPY --from=builder /app/build .
COPY nginx.conf /etc/nginx/conf.d/default.conf
RUN printf '{"sha":"%s","run_id":"%s"}' "$GIT_SHA" "$RUN_ID" > version.json
ENTRYPOINT ["nginx", "-g", "daemon off;"]
