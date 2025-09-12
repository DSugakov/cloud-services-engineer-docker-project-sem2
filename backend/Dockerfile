FROM golang:1.17-alpine AS builder

RUN apk add --no-cache ca-certificates && update-ca-certificates

WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .

ENV CGO_ENABLED=0 GOOS=linux GOARCH=amd64
RUN go build -trimpath -ldflags="-s -w" -o /out/api ./cmd/api

FROM alpine:3.19

# add non-root user and minimal tools for healthcheck
RUN adduser -D -H -s /sbin/nologin app && \
    apk add --no-cache ca-certificates wget && \
    update-ca-certificates

WORKDIR /
COPY --from=builder /out/api /api

EXPOSE 8081
USER app

HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD wget -qO- http://127.0.0.1:8081/health >/dev/null 2>&1 || exit 1

ENTRYPOINT ["/api"]
