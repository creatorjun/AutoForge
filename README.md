# AutoForge: 자율주행 코드 생성 AI 백엔드 마스터 플랜 및 상세 구현 명세서


## 1. 프로젝트 개요 및 아키텍처 철학
**AutoForge**는 사용자의 요구사항을 바탕으로 멀티 LLM 협의체(Multi-Agent System)가 스스로 코드를 분석, 작성, 검증, 수정하여 최종 산출물을 제공하는 엔터프라이즈 AI 서비스입니다.

### 1.1. 설계 원칙 (Extreme Hexagonal & DDD)
* **Infrastructure Agnostic**: 도메인 객체(Entity)는 외부 세계(DB, 프레임워크, 라이브러리)를 전혀 모릅니다. `import org.springframework...` 또는 `import javax.persistence...` 구문이 도메인 계층에 존재할 수 없습니다.
* **Rich Domain Model**: 비즈니스 룰과 상태 변경 로직은 서비스(Service) 클래스가 아닌 도메인 엔티티 내부에 캡슐화됩니다.
* **CQRS**: 명령(Command)과 조회(Query)의 책임을 분리하여, 복잡한 데이터 조회는 도메인 모델을 우회해 성능을 극대화합니다.

### 1.2. 디렉토리 구조 (Package Structure)
의존성 방향은 항상 바깥쪽(Adapter)에서 안쪽(Domain)을 향해야 합니다.

```text
com.autoforge.backend
├── domain                  # [의존성 없음] 순수 Java 핵심 비즈니스 로직
│   ├── model               # AITask, WorkflowStep, CodeSnapshot, ScoreVO
│   ├── exception           # 도메인 전용 예외 처리
│   └── service             # 순수 도메인 서비스 (필요시)
├── application             # [Domain 의존] 애플리케이션 유스케이스 및 포트
│   ├── port
│   │   ├── in              # Inbound Port (UseCase 인터페이스)
│   │   └── out             # Outbound Port (SPI - Repository, LLMClient 인터페이스)
│   └── service             # Inbound Port의 구현체 (트랜잭션 관리, 도메인 오케스트레이션)
├── adapter                 # [Application, Domain 의존] 외부와의 통신 (Spring 등)
│   ├── in                  # Web, Event Listener
│   │   ├── web             # REST Controllers, DTOs
│   │   └── event           # SQS, RabbitMQ Consumer
│   └── out                 # DB, External API
│       ├── persistence     # Spring Data JPA Repository, JPA Entity, Mapper, QueryDSL
│       ├── ai              # OpenAI, Anthropic, Custom LLM API Clients
│       └── redis           # Redis Adapter (Cache, Token)
└── global                  # 공통 설정 (Security, Exception Handler, Config)
```

---

## 2. 코어 도메인 (Domain Layer) 상세 설계

도메인 모델은 철저하게 비즈니스 로직만을 담는 POJO로 설계합니다.

### 2.1. Root Aggregate: `AITask`
작업의 전체 상태와 점수를 관리합니다.

```java
// domain/model/AITask.java
package com.autoforge.backend.domain.model;

import java.time.LocalDateTime;
import java.util.UUID;

public class AITask {
    private final UUID id;
    private final UUID userId;
    private final String requirements;
    private TaskStatus status;
    private Double totalScore;
    private final LocalDateTime createdAt;

    // 생성자, Getter 생략...

    public void startProcessing() {
        if (this.status != TaskStatus.PENDING) {
            throw new IllegalStateException("Task must be in PENDING state to start.");
        }
        this.status = TaskStatus.PROCESSING;
    }

    public void completeTask(Double finalScore) {
        this.status = TaskStatus.COMPLETED;
        this.totalScore = finalScore;
    }

    public void failTask() {
        this.status = TaskStatus.FAILED;
    }
}
```

### 2.2. Value Object (VO): `EvaluationScore`
보안, 안정성, 성능 점수 등을 캡슐화합니다.

```java
// domain/model/EvaluationScore.java
public class EvaluationScore {
    private final int securityScore;
    private final int stabilityScore;
    private final int performanceScore;
    private final String feedback;

    // 비즈니스 룰: 합산 점수 계산 로직
    public double calculateAverage() {
        return (securityScore + stabilityScore + performanceScore) / 3.0;
    }

    public boolean isPassable() {
        return securityScore >= 80 && stabilityScore >= 70 && performanceScore >= 70;
    }
}
```

---

## 3. 멀티 에이전트 협의체 오케스트레이션



AI 로직은 어댑터 계층의 `out/ai`에 구현되며, 오케스트레이터가 하위 모델을 제어합니다.

### 3.1. 에이전트 협업 파이프라인 (The Whiteboard Pattern)
1.  **Orchestrator 시작**: 유저의 `requirements`를 받아 작업 계획 수립.
2.  **Analyzer Agent**: 요구사항 명세 및 아키텍처 설계 도출 -> `WorkflowStep`에 기록.
3.  **Drafter Agent**: 초기 코드 작성 -> `CodeSnapshot` (Iteration 1) 저장.
4.  **Feedback Loop (협의체 평가)**:
    * **Security Agent**: 보안 취약점 점검 및 피드백.
    * **Stability/Performance Agent**: 예외 처리 및 성능 점검.
    * 평가 결과(`EvaluationScore`)를 종합.
5.  **Refactoring Loop**: 합산 점수가 기준치(예: 85점)를 넘을 때까지 Orchestrator가 Drafter에게 피드백을 전달하여 코드 수정(`CodeSnapshot` 덮어쓰기).
6.  **Finalize**: 기준 점수 통과 시, 압축 파일(Zip) 생성 및 `COMPLETED` 상태 변경.

### 3.2. 포트 선언 (Application Layer)
```java
// application/port/out/ILLMOrchestratorClient.java
public interface ILLMOrchestratorClient {
    CodeSnapshot executeWhiteboardLoop(AITask task);
}
```

---

## 4. 데이터베이스 및 CQRS 최적화 (PostgreSQL)

도메인 모델과 영속성 모델(JPA Entity)을 분리합니다. 아래는 어댑터 계층에서 사용할 실제 테이블 스키마(DDL)입니다.

### 4.1. DDL (Data Definition Language)
```sql
-- 1. AI 메인 작업 테이블
CREATE TABLE tb_ai_tasks (
    task_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    requirements TEXT NOT NULL,
    status VARCHAR(30) NOT NULL,
    total_score DECIMAL(5,2),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 2. 오케스트레이터 단계별 로그 (Command 모델용)
CREATE TABLE tb_workflow_steps (
    step_id UUID PRIMARY KEY,
    task_id UUID NOT NULL REFERENCES tb_ai_tasks(task_id),
    agent_type VARCHAR(50) NOT NULL, -- ORCHESTRATOR, DRAFTER, SECURITY...
    prompt_input TEXT,
    response_output TEXT,
    execution_time_ms BIGINT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 3. 화이트보드 코드 스냅샷 (버전 관리)
CREATE TABLE tb_code_snapshots (
    snapshot_id UUID PRIMARY KEY,
    task_id UUID NOT NULL REFERENCES tb_ai_tasks(task_id),
    iteration_count INT NOT NULL,
    source_code TEXT NOT NULL,
    security_score INT,
    performance_score INT,
    feedback_notes JSONB,
    is_final BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_code_snapshots_task ON tb_code_snapshots(task_id);
```

### 4.2. Persistence Adapter 구현 전략
DB에서 데이터를 꺼내올 때, JPA 엔티티인 `JpaAITaskEntity`를 순수 도메인 객체인 `AITask`로 변환(Map)하여 Application 계층에 넘겨줍니다. 이를 통해 영속성 컨텍스트(Lazy Loading 예외 등)가 도메인을 오염시키는 것을 막습니다.

---

## 5. 비동기 이벤트 기반 처리 및 실시간 통신 (Event-Driven)

사용자가 요청을 보내면 백엔드는 즉시 응답하고, 무거운 AI 작업은 백그라운드에서 처리합니다.

### 5.1. 이벤트 흐름 (Spring ApplicationEventPublisher)
1.  **Web Adapter**: `POST /api/v1/tasks` 요청 수신.
2.  **UseCase**: `AITask` 생성 후 DB에 `PENDING`으로 저장. `TaskRequestedEvent` 발행.
3.  **Web Adapter**: 클라이언트에게 `202 Accepted` 및 `taskId` 반환.
4.  **Event Listener (Worker)**: 이벤트를 비동기로 수신(`@Async`)하여 `ILLMOrchestratorClient` 호출.

### 5.2. Server-Sent Events (SSE) 실시간 푸시
작업이 진행되는 동안 클라이언트는 SSE 포트를 열어두고 진행률을 수신합니다.

```java
// adapter/in/web/SSENotificationController.java
@GetMapping("/api/v1/tasks/{taskId}/stream")
public SseEmitter streamTaskProgress(@PathVariable UUID taskId) {
    SseEmitter emitter = new SseEmitter(60 * 1000 * 30L); // 30분 타임아웃
    sseConnectionManager.add(taskId, emitter);
    return emitter;
}
```
AI 워커에서 새로운 `WorkflowStep`이 완료될 때마다 Redis Pub/Sub을 거쳐 SSE Emitter로 클라이언트에게 JSON 데이터를 Push합니다.

---

## 6. 자체 인증 및 인가 파이프라인 (Custom OAuth2 & JWT)

Spring Security의 무거운 기본 FilterChain을 최소화하고, 직접 토큰을 검증하는 Stateless 필터를 구현합니다.

### 6.1. Custom JWT Filter Logic
```java
// global/security/JwtAuthenticationFilter.java
// 로직 흐름 요약:
// 1. HTTP 헤더에서 "Bearer [Access Token]" 추출.
// 2. Token Provider를 통해 서명(Signature)만 검증. (DB 조회 X)
// 3. 만료되었을 경우 401 Unauthorized 반환 -> 클라이언트가 RTR(Refresh Token Rotation) 엔드포인트 호출.
```

### 6.2. Refresh Token Rotation (Redis)
탈취 방지를 위해 Refresh Token은 1회용으로 사용됩니다.
* **저장**: 로그인 시 생성된 Refresh Token의 Hash값을 Redis에 저장.
* **사용**: 클라이언트가 갱신 요청 시 쿠키에서 Refresh Token 확인. 일치하면 새로운 Access & Refresh Token 발급 후 기존 Redis 데이터 파기.

---

## 7. 인프라스트럭처 및 배포 아키텍처 (Rocky Linux + Docker)

엔터프라이즈 보안 기준을 충족하기 위한 배포 스크립트입니다.

### 7.1. Multi-Stage Dockerfile (Non-Root 권한)
공격 표면을 최소화하기 위해 컴파일 환경과 실행 환경을 완벽히 분리합니다.

```dockerfile
# Stage 1: Build (JDK 17)
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
COPY . .
RUN ./gradlew clean bootJar --no-daemon

# Stage 2: Production (JRE 17 - 초경량)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# 보안: Non-Root 사용자 생성 및 권한 부여
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

COPY --from=builder /app/build/libs/autoforge-backend-*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 7.2. Nginx Reverse Proxy (SSL 오프로딩)
Tomcat을 외부망에 노출하지 않고 Nginx를 통해 라우팅합니다.

```nginx
# nginx.conf (컨테이너 내부)
server {
    listen 443 ssl http2;
    server_name api.autoforge.com;

    ssl_certificate /etc/nginx/ssl/autoforge.crt;
    ssl_certificate_key /etc/nginx/ssl/autoforge.key;

    # SSE(Server-Sent Events)를 위한 헤더 설정 유지
    location /api/ {
        proxy_pass http://backend-spring:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # SSE Timeout 방지
        proxy_buffering off;
        proxy_cache off;
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
    }
}
```