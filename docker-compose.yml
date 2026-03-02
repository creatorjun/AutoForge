# 최신 Docker Compose는 version을 명시하지 않아도 됩니다.
name: autoforge-stack

services:
  # 1. AI 메인 백엔드 (AutoForge)
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: autoforge-api
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/autoforge
      - SPRING_DATASOURCE_USERNAME=autoforge_user
      - SPRING_DATASOURCE_PASSWORD=autoforge_pass
      - SPRING_DATA_REDIS_HOST=redis
      - SPRING_DATA_REDIS_PORT=6379
    depends_on:
      postgres:
        condition: service_healthy # DB가 응답할 때까지 대기
      redis:
        condition: service_healthy
    networks:
      - autoforge-network
    restart: unless-stopped

  # 2. 메인 데이터베이스 (PostgreSQL)
  postgres:
    image: postgres:16-alpine
    container_name: autoforge-postgres
    environment:
      POSTGRES_DB: autoforge
      POSTGRES_USER: autoforge_user
      POSTGRES_PASSWORD: autoforge_pass
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - autoforge-network
    # 헬스체크: DB가 실제로 쿼리를 받을 수 있는지 확인
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U autoforge_user -d autoforge"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: always

  # 3. 인메모리 캐시 & 워커 큐 (Redis)
  redis:
    image: redis:7.2-alpine
    container_name: autoforge-redis
    command: redis-server --appendonly yes # 영속성 보장
    volumes:
      - redis-data:/data
    networks:
      - autoforge-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: always

  # 4. 리버스 프록시 (Nginx - 보안 및 라우팅)
  nginx:
    image: nginx:alpine
    container_name: autoforge-nginx
    ports:
      - "80:80"
      # - "443:443" # SSL 적용 시 주석 해제
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - api
    networks:
      - autoforge-network
    restart: always

networks:
  autoforge-network:
    driver: bridge

volumes:
  postgres-data:
  redis-data: