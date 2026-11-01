# Stage 1 - Build the application
FROM golang:1.21 AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -o auth-service .

# Stage 2 - Run the application
FROM alpine:3.20

WORKDIR /app

COPY --from=builder /app/auth-service .

EXPOSE 8001

CMD ["./auth-service"]