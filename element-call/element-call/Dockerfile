FROM node:22 AS build

COPY . .

RUN yarn
RUN yarn build

FROM nginxinc/nginx-unprivileged:alpine

COPY --from=build dist /app
COPY --from=build config/nginx.conf /etc/nginx/conf.d/default.conf

USER root

RUN rm -rf /usr/share/nginx/html

USER 101
