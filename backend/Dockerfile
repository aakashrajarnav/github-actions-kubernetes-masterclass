# Stage 1: Build
FROM golang:1.26-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux \
	go build -trimpath -ldflags="-s -w" -o /skillpulse .

# Stage 2: CA certificates (needed for TLS clients)
FROM alpine:3.23 AS certs
RUN apk --no-cache add ca-certificates

# Stage 3: Minimal runtime
FROM scratch

COPY --from=builder /skillpulse /skillpulse
COPY --from=certs /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 8080

ENTRYPOINT ["/skillpulse"]
