# syntax=docker/dockerfile:1

# ---------- Stage 1: build ----------
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Copia só os arquivos de módulo primeiro - cache de camada não
# invalida a cada mudança de código, só quando as dependências mudam.
COPY go.mod go.sum* ./
RUN go mod download

COPY . .

# Build estático (CGO_ENABLED=0): sem dependência de libc na imagem
# final, permitindo usar "scratch" ou "distroless" com segurança.
# -ldflags "-s -w" remove símbolos de debug, reduzindo o tamanho do binário.
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /donation-service .

# ---------- Stage 2: imagem final ----------
# distroless: sem shell, sem gerenciador de pacotes, sem nada além do
# runtime mínimo - reduz drasticamente a superfície de ataque e o
# número de CVEs (ótimo para o scan do Trivy/SCA no pipeline).
FROM gcr.io/distroless/static-debian12:nonroot AS runtime

WORKDIR /app

COPY --from=builder /donation-service .

# A imagem distroless "nonroot" já roda como usuário não-root por padrão.
USER nonroot:nonroot

EXPOSE 8082

CMD ["./donation-service"]
