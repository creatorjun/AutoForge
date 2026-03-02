# docker.md

# Docker 운영 가이드 — AutoForge

## Dockerfile (Multi-Stage, Java 21, Non-Root)

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY . .
RUN ./gradlew clean bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
COPY --from=builder /app/build/libs/autoforge-backend-*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseZGC", "-XX:+ZGenerational", "-jar", "app.jar"]
```

## Nginx — `nginx/default.conf`

```nginx
server {
    listen 80;
    server_name api.autoforge.com;

    location /api/ {
        proxy_pass         http://autoforge-api:8080;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;

        proxy_buffering           off;
        proxy_cache               off;
        proxy_set_header          Connection '';
        proxy_http_version        1.1;
        chunked_transfer_encoding off;
        proxy_read_timeout        1800s;
    }
}
```

> SSL 적용 시 `listen 443 ssl http2;` 로 변경하고 인증서 경로를 추가합니다.

## 환경 변수 파일 (`.env`)

프로젝트 루트에 `.env` 파일을 생성하고 아래 값을 채웁니다. `.env`는 절대 Git에 커밋하지 않습니다.

```env
JWT_SECRET=your-very-long-random-secret-key
LLM_OPENAI_API_KEY=sk-...
LLM_ANTHROPIC_API_KEY=sk-ant-...
S3_BUCKET_NAME=autoforge-artifacts
S3_REGION=ap-northeast-2
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
```

## 명령어

```bash
# 전체 스택 빌드 및 실행
docker compose up --build -d

# API 서버 로그 실시간 확인
docker compose logs -f api

# 스택 중지
docker compose down

# 볼륨까지 초기화 (DB 데이터 삭제)
docker compose down -v

# 특정 서비스만 재시작
docker compose restart api

# 컨테이너 내부 접속
docker compose exec postgres psql -U autoforge_user -d autoforge
docker compose exec redis redis-cli
```

## 헬스체크 흐름

```
Docker Compose 시작
  → postgres: pg_isready 통과 (service_healthy)
  → redis:    redis-cli ping 통과 (service_healthy)
  → api:      depends_on 조건 만족 후 기동
  → nginx:    api 기동 후 라우팅 시작
```
