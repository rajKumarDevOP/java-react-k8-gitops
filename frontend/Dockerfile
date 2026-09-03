FROM node:20-alpine AS build

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci

COPY . .
ENV VITE_API_BASE_URL=https://uat.svkm.ac.in/feedbackapi
RUN npm run build

FROM nginx:alpine

RUN rm -f /etc/nginx/conf.d/default.conf

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html

RUN sed -i 's|^pid .*;|pid /tmp/nginx.pid;|' /etc/nginx/nginx.conf \
       && mkdir -p \
        /var/cache/nginx/client_temp \
        /var/cache/nginx/proxy_temp \
        /var/cache/nginx/fastcgi_temp \
        /var/cache/nginx/uwsgi_temp \
        /var/cache/nginx/scgi_temp \
        /var/run \
    && chown -R 1000:1000 \
        /var/cache/nginx \
        /var/run \
        /var/log/nginx \
        /usr/share/nginx/html \
        /etc/nginx

USER 1000

EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
