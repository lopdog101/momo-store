# multistage dockerfile

# build stage

FROM node:16.19-alpine AS builder
# Create app directory
WORKDIR /usr/src/app
COPY . .
RUN npm install 
ARG VUE_APP_API_URL=http://${DEV_HOST}:8081
RUN npm run build


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