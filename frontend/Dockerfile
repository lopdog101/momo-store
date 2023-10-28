FROM node:16 AS builder
# Create app directory
WORKDIR /usr/src/app
ARG MOMO_URL=${MOMO_URL}
ARG VUE_APP_API_URL=https://${MOMO_URL}/api
ARG NODE_ENV=dev
COPY ./package.json .
RUN npm install
COPY . .
RUN npm run build
RUN ls -la /usr/src/app/dist


# release stage
FROM nginx:1.23.3-alpine
WORKDIR /usr/share/nginx/html
# copy artifacts from build stage 
COPY --from=builder /usr/src/app/dist/ .
# copy config for nginx
RUN rm /etc/nginx/conf.d/default.conf
COPY --from=builder /usr/src/app/nginx.tmpl /etc/nginx/conf.d/default.conf
# set port
EXPOSE 80
