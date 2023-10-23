# syntax=docker/dockerfile:1

##
## STEP 1 - BUILD
##

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