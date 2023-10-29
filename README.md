# Momo Store aka Пельменная №2

# Оглавление <!-- omit in toc -->

- [Общее](#общее)
  - [Frontend](#frontend)
  - [Backend](#backend)
  - [Локальный запуск](#локальный-запуск)
  - [Стратегии деплоя](#стратегии-деплоя)
  - [Структура репозитория приложения](#структура-репозитория-приложения)

<img width="900" alt="image" src="https://user-images.githubusercontent.com/9394918/167876466-2c530828-d658-4efe-9064-825626cc6db5.png">

## Структура репозитория приложения

```
.
├── backend
├── frontend
├── docker-compose.yml
├── helm-chart.gitlab-ci.yml
├── .gitlab-ci.yml
└── README.md
```

- Деплой осуществляется автоматизировано в указанный k8s кластер при помощи ArgoCD.
- В gitlab производится сборка контейнеров и кода, контейнеры хранятся в gitlab container registry, код backen и frontend дополнительно загружается на Nexus
- Helm чарты бэкэнда и фронтенда хранятся в Nexus репозитории.
- Статический анализ кода осуществляется с помощью Sonarqube.
- Версионирование бэкэнда и фронтенда автоматическое при внесении изменений (используется ID пайплайна), версии чартов необходимо менять вручную.

# Общее

Сборка приложения выполняется сразу в контейнеры, которые затем деплоятся в кластер Kubernetes, развернутый в Яндекс.Облаке.

## Frontend

Контейнер фронтенда собирается на базе образа `node:16.13.2` и публикуется в образе `nginx:stable`

При сборке необходимо указывать следующие переменные: 

```
VERSION - версия приложения (по-умолчанию формируется в пайплайне - 1.0.${CI_PIPELINE_ID})
```
После сборки контейнер тестируется, затем маркируется тэгом `latest` и публикуется в GitLab Container Registry


Dockerfile:

```dockerfile
# Stage 1 - Build UI
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
```

При запуске контейнера с фронтендом необходимо передать переменную окружения MOMO_URL, которая заменяет в коде приложения заранее размещенный шаблон. Это позволяет динамически менять адрес бэкенда без повторной сборки приложения.

## Backend

Сборка контейнера на базе образа `golang:1.18-alpine3.17` и затем артифакт сборки копируется в контейнер для минимализации итогового размера контейнера.

После тестирования контейнера, он публикуется в GitLab Container Registry с тегом `latest`.

Dockerfile:

```dockerfile

FROM golang:1.18-alpine as build

# Add a work directory
WORKDIR /app
# Cache and install dependencies
COPY go.mod go.sum ./
RUN go mod download
# Copy app files
COPY . .
# compile application
RUN go build -o /backend /app/cmd/api

##
## STEP 2 - DEPLOY
##
FROM alpine:3.15.0

# Add a work directory
WORKDIR /
# Copy built binary from builder
COPY --from=build /backend /backend
# Expose port
EXPOSE 8081
# Start app
ENTRYPOINT ["/backend"]
```
