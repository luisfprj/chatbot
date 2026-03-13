# Infraestrutura — starter-kit

> **Projeto:** starter-kit
> **Data de criação:** 07/03/2026
> **Última atualização:** 08/03/2026

Documento complementar: [decisoes-compartilhadas.md](decisoes-compartilhadas.md)

---

## 1. Visão Geral dos Containers

```text
┌────────────────────────────────────────────────────────┐
│                 Docker Compose Network                │
│                 (starterkit-network)                  │
│                                                       │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   nginx     │  │     api      │  │   postgres   │  │
│  │    :80 ───────▶│    :8080     │──▶│    :5432     │  │
│  │             │  │              │  │              │  │
│  │ Frontend    │  │ Spring Boot  │  │ PostgreSQL   │  │
│  │ estático    │  │ Java 21      │  │ 16           │  │
│  └─────────────┘  └──────────────┘  └──────────────┘  │
└────────────────────────────────────────────────────────┘
```

**Portas expostas ao host:**

- `80` para Nginx em produção.
- `8080` para backend apenas em desenvolvimento local.
- `5432` para PostgreSQL apenas em desenvolvimento local.

**Convenções operacionais adotadas:**

- Timezone padrão da aplicação: `America/Sao_Paulo`.
- Datas persistidas no banco com timezone.
- Estado fora do banco, quando existir, entra no escopo de backup e restore.

---

## 2. Docker Compose — Desenvolvimento

### `docker-compose.dev.yml`

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16-alpine
    container_name: starterkit-postgres
    environment:
      POSTGRES_DB: starterkit
      POSTGRES_USER: starterkit
      POSTGRES_PASSWORD: starterkit_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - starterkit-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U starterkit"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:

networks:
  starterkit-network:
    driver: bridge
```

**Em desenvolvimento:**

- Backend roda localmente via IDE ou Gradle.
- Frontend roda localmente via Vite.
- Apenas o PostgreSQL roda no Docker.
- Nginx não é obrigatório em dev; o proxy do Vite cobre o fluxo local.
- A porta `5432` fica exposta para ferramentas como DBeaver.

---

## 3. Docker Compose — Produção

### `docker-compose.yml`

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16-alpine
    container_name: starterkit-postgres
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - starterkit-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USERNAME}"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    container_name: starterkit-api
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: ${DB_NAME}
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      JWT_SECRET: ${JWT_SECRET}
      JWT_EXPIRATION: ${JWT_EXPIRATION:-900000}
      JWT_REFRESH_EXPIRATION: ${JWT_REFRESH_EXPIRATION:-604800000}
      CORS_ORIGINS: ${CORS_ORIGINS:-http://localhost:80}
      APP_TIMEZONE: ${APP_TIMEZONE:-America/Sao_Paulo}
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - starterkit-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8080/api/actuator/health | grep UP || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  nginx:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: starterkit-nginx
    ports:
      - "80:80"
    depends_on:
      api:
        condition: service_healthy
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - starterkit-network
    restart: unless-stopped

volumes:
  postgres_data:

networks:
  starterkit-network:
    driver: bridge
```

### `.env` (exemplo)

```env
# Banco de Dados
DB_NAME=starterkit
DB_USERNAME=starterkit
DB_PASSWORD=mudar_em_producao

# JWT
JWT_SECRET=chave-secreta-com-pelo-menos-256-bits-mudar-em-producao
JWT_EXPIRATION=900000
JWT_REFRESH_EXPIRATION=604800000
APP_TIMEZONE=America/Sao_Paulo

# CORS
CORS_ORIGINS=http://localhost:80
```

> **Importante:** `.env` não deve ser versionado. Fornecer apenas `.env.example`.

---

## 4. Dockerfiles

### Backend — `/api/Dockerfile`

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY gradle/ gradle/
COPY gradlew build.gradle settings.gradle ./
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon
COPY src/ src/
RUN ./gradlew bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=build /app/build/libs/*.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Frontend — `/frontend/Dockerfile`

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## 5. Nginx — Configuração

### `nginx/nginx.conf`

```nginx
server {
    listen 80;
    server_name localhost;

    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options SAMEORIGIN;
    add_header Referrer-Policy no-referrer;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 1000;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://api:8080/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 30s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf)$ {
        root /usr/share/nginx/html;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location ~ /\. {
        deny all;
    }
}
```

---

## 6. Portas e Serviços

### Produção

| Serviço | Porta interna | Porta exposta | Acesso externo |
| ------- | ------------- | ------------- | -------------- |
| Nginx | 80 | 80 | ✅ |
| Backend | 8080 | — | ❌ |
| PostgreSQL | 5432 | — | ❌ |

### Desenvolvimento

| Serviço | Porta | Acesso |
| ------- | ----- | ------ |
| Vite | 5173 | `http://localhost:5173` |
| Backend | 8080 | `http://localhost:8080` |
| PostgreSQL | 5432 | `localhost:5432` |
| Swagger UI | 8080 | `http://localhost:8080/api/swagger-ui/index.html` |

---

## 7. Spring Boot — Profiles de Configuração

### `application.yml`

```yaml
spring:
  application:
    name: starter-kit
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
  jackson:
    time-zone: ${APP_TIMEZONE:America/Sao_Paulo}

management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      probes:
        enabled: true

server:
  port: 8080
  servlet:
    context-path: /api
```

### `application-dev.yml`

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/starterkit
    username: starterkit
    password: starterkit_dev
  jpa:
    show-sql: true

logging:
  level:
    com.starterkit: DEBUG
    org.springframework.security: DEBUG

springdoc:
  swagger-ui:
    enabled: true
```

### `application-prod.yml`

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    show-sql: false

logging:
  level:
    com.starterkit: INFO
    org.springframework.security: WARN
  pattern:
    console: '{"timestamp":"%d{yyyy-MM-dd''T''HH:mm:ss.SSSXXX}","level":"%level","logger":"%logger","traceId":"%X{traceId:-}","message":"%msg"}%n'

app:
  timezone: ${APP_TIMEZONE:America/Sao_Paulo}

springdoc:
  swagger-ui:
    enabled: false
```

---

## 8. Volumes e Persistência

| Volume | Container | Path | Descrição |
| ------ | --------- | ---- | --------- |
| `postgres_data` | postgres | `/var/lib/postgresql/data` | Dados do banco |

**Regra de operação:** se um projeto derivado passar a usar arquivos operacionais, esses diretórios devem entrar formalmente no backup, restore, monitoramento e checklist de deploy.

---

## 9. Comandos Úteis

### Desenvolvimento

```bash
docker compose -f docker-compose.dev.yml up -d
./gradlew bootRun --args='--spring.profiles.active=dev'
npm run dev
docker compose -f docker-compose.dev.yml down
```

### Produção

```bash
docker compose up -d --build
docker compose logs -f
docker compose logs -f api
docker compose down
docker compose down -v
```

### Regra de troubleshooting

Em incidentes de ambiente, validar nesta ordem:

1. health check do PostgreSQL.
2. health check da API.
3. variáveis obrigatórias, incluindo timezone.
4. conectividade proxy `/api`.
5. volumes e permissões de escrita, quando existirem arquivos operacionais.

---

## 10. Git e GitHub

### Repositório

| Dado | Valor |
| ---- | ----- |
| Plataforma | GitHub |
| Repositório | `git@github.com:luisfprj/starter-kit.git` |
| URL | https://github.com/luisfprj/starter-kit |
| Tipo | Mono-repo |

### Branches

| Branch | Uso | Protegida |
| ------ | --- | --------- |
| `main` | Código estável | Sim |
| `develop` | Desenvolvimento ativo | Não |

---

## 11. .gitignore

```gitignore
.env
*.log
.DS_Store
Thumbs.db

api/.gradle/
api/build/
api/bin/
api/out/

frontend/node_modules/
frontend/dist/
frontend/.vite/

postgres_data/

.vscode/
.idea/
*.swp
*.swo
```