FROM golang:1.20-alpine AS builder

WORKDIR /app
COPY go.* ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o publisher

FROM alpine:3.18
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=builder /app/publisher .
EXPOSE 8080
CMD ["./publisher"]