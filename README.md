# README.md

# AutoForge Backend

> **자율주행 코드 생성 AI 백엔드** — 멀티 LLM 협의체(Multi-Agent System)가 사용자의 요구사항을 분석·작성·검증·수정하여 최종 코드 산출물을 자동으로 생성합니다.

---

## Tech Stack

| Category | Stack |
|---|---|
| Language | Java 21 (LTS) |
| Framework | Spring Boot 4.0.x |
| Build | Gradle 8.x |
| Database | PostgreSQL 16 |
| Cache / Queue | Redis 7.2 |
| Proxy | Nginx (Alpine) |
| Container | Docker / Docker Compose |
| Auth | OAuth2 (Google, Apple) + JWT (RTR) |
| Async | Spring ApplicationEventPublisher + @Async |
| Realtime | Server-Sent Events (SSE) + Redis Pub/Sub |
| Storage | AWS S3 (최종 Zip 아티팩트) |

---

## Architecture

### 설계 원칙 (Extreme Hexagonal + DDD)

- **Infrastructure Agnostic** — 도메인 계층에 `org.springframework.*` / `jakarta.persistence.*` import 금지. 도메인은 순수 Java POJO.
- **Rich Domain Model** — 비즈니스 규칙과 상태 변경 로직은 Service가 아닌 Domain Entity 내부에 캡슐화.
- **CQRS** — Command(쓰기)와 Query(읽기)의 책임을 분리. 복잡한 조회는 QueryDSL로 도메인 모델을 우회하여 처리.
- **의존성 방향** — 항상 바깥(Adapter) → 안(Domain) 방향만 허용.

### Package Structure

```text
com.autoforge.backend
├── domain
│   ├── model
│   │   ├── AITask.java
│   │   ├── TaskStatus.java
│   │   ├── WorkflowStep.java
│   │   ├── AgentType.java
│   │   ├── StepStatus.java
│   │   ├── CodeSnapshot.java
│   │   └── EvaluationScore.java
│   ├── exception
│   │   ├── TaskNotFoundException.java
│   │   ├── TaskStateException.java
│   │   └── MaxIterationExceededException.java
│   └── service
│       └── (필요 시 순수 도메인 서비스)
├── application
│   ├── port
│   │   ├── in
│   │   │   ├── CreateTaskUseCase.java
│   │   │   ├── GetTaskUseCase.java
│   │   │   └── RefreshTokenUseCase.java
│   │   └── out
│   │       ├── AITaskRepositoryPort.java
│   │       ├── WorkflowStepRepositoryPort.java
│   │       ├── CodeSnapshotRepositoryPort.java
│   │       ├── ILLMOrchestratorClient.java
│   │       ├── TokenStorePort.java
│   │       ├── SseNotificationPort.java
│   │       ├── FileStoragePort.java
│   │       └── OAuthProviderPort.java
│   └── service
│       ├── CreateTaskService.java
│       ├── GetTaskService.java
│       ├── AITaskWorkerService.java
│       └── AuthService.java
├── adapter
│   ├── in
│   │   ├── web
│   │   │   ├── TaskController.java
│   │   │   ├── AuthController.java
│   │   │   ├── SseController.java
│   │   │   └── dto/
│   │   └── event
│   │       └── TaskEventListener.java
│   └── out
│       ├── persistence
│       │   ├── entity/
│       │   ├── repository/
│       │   └── mapper/
│       ├── ai
│       │   ├── LLMOrchestratorAdapter.java
│       │   └── client/
│       ├── redis
│       │   ├── RedisTokenAdapter.java
│       │   └── RedisSseAdapter.java
│       └── s3
│           └── S3FileStorageAdapter.java
└── global
    ├── config
    │   ├── AsyncConfig.java
    │   ├── SecurityConfig.java
    │   └── RedisConfig.java
    ├── security
    │   ├── JwtTokenProvider.java
    │   └── JwtAuthenticationFilter.java
    └── exception
        ├── GlobalExceptionHandler.java
        └── ErrorResponse.java
```

---

## Domain Model

### `TaskStatus` Enum

```java
package com.autoforge.backend.domain.model;

public enum TaskStatus {
    PENDING,
    PROCESSING,
    COMPLETED,
    FAILED;

    public boolean isTerminal() {
        return this == COMPLETED || this == FAILED;
    }
}
```

### `AgentType` Enum

```java
package com.autoforge.backend.domain.model;

public enum AgentType {
    ORCHESTRATOR,
    ANALYZER,
    DRAFTER,
    SECURITY,
    STABILITY,
    PERFORMANCE
}
```

### `StepStatus` Enum

```java
package com.autoforge.backend.domain.model;

public enum StepStatus {
    SUCCESS,
    FAILED,
    TIMEOUT
}
```

### Root Aggregate: `AITask`

```java
package com.autoforge.backend.domain.model;

import com.autoforge.backend.domain.exception.TaskStateException;
import java.time.Instant;
import java.util.UUID;

public class AITask {
    private final UUID id;
    private final UUID userId;
    private final String requirements;
    private TaskStatus status;
    private Double totalScore;
    private String finalArtifactUrl;
    private final Instant createdAt;
    private Instant updatedAt;

    public AITask(UUID id, UUID userId, String requirements) {
        this.id = id;
        this.userId = userId;
        this.requirements = requirements;
        this.status = TaskStatus.PENDING;
        this.createdAt = Instant.now();
        this.updatedAt = Instant.now();
    }

    public void startProcessing() {
        if (this.status != TaskStatus.PENDING) {
            throw new TaskStateException("Task must be PENDING to start. current=" + this.status);
        }
        this.status = TaskStatus.PROCESSING;
        this.updatedAt = Instant.now();
    }

    public void completeTask(Double finalScore, String artifactUrl) {
        if (this.status != TaskStatus.PROCESSING) {
            throw new TaskStateException("Task must be PROCESSING to complete. current=" + this.status);
        }
        this.status = TaskStatus.COMPLETED;
        this.totalScore = finalScore;
        this.finalArtifactUrl = artifactUrl;
        this.updatedAt = Instant.now();
    }

    public void failTask() {
        this.status = TaskStatus.FAILED;
        this.updatedAt = Instant.now();
    }

    public UUID getId() { return id; }
    public UUID getUserId() { return userId; }
    public String getRequirements() { return requirements; }
    public TaskStatus getStatus() { return status; }
    public Double getTotalScore() { return totalScore; }
    public String getFinalArtifactUrl() { return finalArtifactUrl; }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }
}
```

### `WorkflowStep` Domain Model

```java
package com.autoforge.backend.domain.model;

import java.time.Instant;
import java.util.UUID;

public class WorkflowStep {
    private final UUID id;
    private final UUID taskId;
    private final int stepSequence;
    private final AgentType agentType;
    private final String promptInput;
    private String responseOutput;
    private StepStatus status;
    private Long executionTimeMs;
    private final Instant createdAt;

    public WorkflowStep(UUID taskId, int stepSequence, AgentType agentType, String promptInput) {
        this.id = UUID.randomUUID();
        this.taskId = taskId;
        this.stepSequence = stepSequence;
        this.agentType = agentType;
        this.promptInput = promptInput;
        this.createdAt = Instant.now();
    }

    public void complete(String responseOutput, long executionTimeMs) {
        this.responseOutput = responseOutput;
        this.executionTimeMs = executionTimeMs;
        this.status = StepStatus.SUCCESS;
    }

    public void fail(StepStatus failStatus) {
        this.status = failStatus;
    }

    public UUID getId() { return id; }
    public UUID getTaskId() { return taskId; }
    public int getStepSequence() { return stepSequence; }
    public AgentType getAgentType() { return agentType; }
    public String getPromptInput() { return promptInput; }
    public String getResponseOutput() { return responseOutput; }
    public StepStatus getStatus() { return status; }
    public Long getExecutionTimeMs() { return executionTimeMs; }
    public Instant getCreatedAt() { return createdAt; }
}
```

### `CodeSnapshot` Domain Model

```java
package com.autoforge.backend.domain.model;

import java.time.Instant;
import java.util.UUID;

public class CodeSnapshot {
    private final UUID id;
    private final UUID taskId;
    private final int iterationCount;
    private final String sourceCode;
    private final EvaluationScore evaluationScore;
    private boolean isBestVersion;
    private final Instant createdAt;

    public CodeSnapshot(UUID taskId, int iterationCount, String sourceCode, EvaluationScore evaluationScore) {
        this.id = UUID.randomUUID();
        this.taskId = taskId;
        this.iterationCount = iterationCount;
        this.sourceCode = sourceCode;
        this.evaluationScore = evaluationScore;
        this.isBestVersion = false;
        this.createdAt = Instant.now();
    }

    public void markAsBest() {
        this.isBestVersion = true;
    }

    public UUID getId() { return id; }
    public UUID getTaskId() { return taskId; }
    public int getIterationCount() { return iterationCount; }
    public String getSourceCode() { return sourceCode; }
    public EvaluationScore getEvaluationScore() { return evaluationScore; }
    public boolean isBestVersion() { return isBestVersion; }
    public Instant getCreatedAt() { return createdAt; }
}
```

### Value Object: `EvaluationScore`

```java
package com.autoforge.backend.domain.model;

public class EvaluationScore {
    private final int securityScore;
    private final int stabilityScore;
    private final int performanceScore;
    private final String securityFeedback;
    private final String stabilityFeedback;
    private final String performanceFeedback;

    public EvaluationScore(
        int securityScore, int stabilityScore, int performanceScore,
        String securityFeedback, String stabilityFeedback, String performanceFeedback
    ) {
        this.securityScore = securityScore;
        this.stabilityScore = stabilityScore;
        this.performanceScore = performanceScore;
        this.securityFeedback = securityFeedback;
        this.stabilityFeedback = stabilityFeedback;
        this.performanceFeedback = performanceFeedback;
    }

    public double calculateAverage() {
        return (securityScore + stabilityScore + performanceScore) / 3.0;
    }

    public boolean isPassable() {
        return securityScore >= 80 && stabilityScore >= 70 && performanceScore >= 70;
    }

    public int getSecurityScore() { return securityScore; }
    public int getStabilityScore() { return stabilityScore; }
    public int getPerformanceScore() { return performanceScore; }
    public String getSecurityFeedback() { return securityFeedback; }
    public String getStabilityFeedback() { return stabilityFeedback; }
    public String getPerformanceFeedback() { return performanceFeedback; }
}
```

---

## Domain Exceptions

```java
// domain/exception/TaskNotFoundException.java
package com.autoforge.backend.domain.exception;

import java.util.UUID;

public class TaskNotFoundException extends RuntimeException {
    public TaskNotFoundException(UUID taskId) {
        super("AITask not found. id=" + taskId);
    }
}
```

```java
// domain/exception/TaskStateException.java
package com.autoforge.backend.domain.exception;

public class TaskStateException extends RuntimeException {
    public TaskStateException(String message) {
        super(message);
    }
}
```

```java
// domain/exception/MaxIterationExceededException.java
package com.autoforge.backend.domain.exception;

import java.util.UUID;

public class MaxIterationExceededException extends RuntimeException {
    public MaxIterationExceededException(UUID taskId, int maxIteration) {
        super("Max iteration exceeded. taskId=" + taskId + ", maxIteration=" + maxIteration);
    }
}
```

---

## Application Layer — Inbound Ports (UseCase)

```java
// application/port/in/CreateTaskUseCase.java
package com.autoforge.backend.application.port.in;

import java.util.UUID;

public interface CreateTaskUseCase {
    UUID createTask(String requirements, UUID userId);
}
```

```java
// application/port/in/GetTaskUseCase.java
package com.autoforge.backend.application.port.in;

import com.autoforge.backend.application.port.in.dto.TaskDetailResult;
import com.autoforge.backend.application.port.in.dto.TaskSummaryResult;
import java.util.List;
import java.util.UUID;

public interface GetTaskUseCase {
    TaskDetailResult getTaskDetail(UUID taskId, UUID requesterId);
    List<TaskSummaryResult> getTasksByUser(UUID userId);
    List<SnapshotResult> getSnapshots(UUID taskId, UUID requesterId);
}
```

```java
// application/port/in/RefreshTokenUseCase.java
package com.autoforge.backend.application.port.in;

import com.autoforge.backend.application.port.in.dto.TokenResult;

public interface RefreshTokenUseCase {
    TokenResult refresh(String refreshToken);
}
```

### Query DTOs

```java
// application/port/in/dto/TaskDetailResult.java
package com.autoforge.backend.application.port.in.dto;

import com.autoforge.backend.domain.model.TaskStatus;
import java.time.Instant;
import java.util.UUID;

public record TaskDetailResult(
    UUID taskId,
    String requirements,
    TaskStatus status,
    Double totalScore,
    String finalArtifactUrl,
    Instant createdAt,
    Instant updatedAt
) {}
```

```java
// application/port/in/dto/TaskSummaryResult.java
package com.autoforge.backend.application.port.in.dto;

import com.autoforge.backend.domain.model.TaskStatus;
import java.time.Instant;
import java.util.UUID;

public record TaskSummaryResult(
    UUID taskId,
    TaskStatus status,
    Double totalScore,
    Instant createdAt
) {}
```

```java
// application/port/in/dto/SnapshotResult.java
package com.autoforge.backend.application.port.in.dto;

import java.time.Instant;
import java.util.UUID;

public record SnapshotResult(
    UUID snapshotId,
    int iterationCount,
    double averageScore,
    boolean isBestVersion,
    Instant createdAt
) {}
```

```java
// application/port/in/dto/TokenResult.java
package com.autoforge.backend.application.port.in.dto;

public record TokenResult(
    String accessToken,
    String refreshToken
) {}
```

---

## Application Layer — Outbound Ports

```java
// application/port/out/AITaskRepositoryPort.java
package com.autoforge.backend.application.port.out;

import com.autoforge.backend.domain.model.AITask;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface AITaskRepositoryPort {
    AITask save(AITask task);
    Optional<AITask> findById(UUID taskId);
    List<AITask> findAllByUserId(UUID userId);
}
```

```java
// application/port/out/WorkflowStepRepositoryPort.java
package com.autoforge.backend.application.port.out;

import com.autoforge.backend.domain.model.WorkflowStep;

public interface WorkflowStepRepositoryPort {
    WorkflowStep save(WorkflowStep step);
}
```

```java
// application/port/out/CodeSnapshotRepositoryPort.java
package com.autoforge.backend.application.port.out;

import com.autoforge.backend.domain.model.CodeSnapshot;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface CodeSnapshotRepositoryPort {
    CodeSnapshot save(CodeSnapshot snapshot);
    void clearBestVersion(UUID taskId);
    List<CodeSnapshot> findAllByTaskId(UUID taskId);
    Optional<CodeSnapshot> findBestByTaskId(UUID taskId);
}
```

```java
// application/port/out/ILLMOrchestratorClient.java
package com.autoforge.backend.application.port.out;

import com.autoforge.backend.domain.model.AITask;
import com.autoforge.backend.domain.model.CodeSnapshot;

public interface ILLMOrchestratorClient {
    CodeSnapshot executeWhiteboardLoop(AITask task, int maxIteration);
}
```

```java
// application/port/out/TokenStorePort.java
package com.autoforge.backend.application.port.out;

import java.util.Optional;
import java.util.UUID;

public interface TokenStorePort {
    void saveRefreshToken(UUID userId, String tokenHash, long ttlSeconds);
    Optional<String> getRefreshTokenHash(UUID userId);
    void deleteRefreshToken(UUID userId);
    void deleteAllTokens(UUID userId);
}
```

```java
// application/port/out/SseNotificationPort.java
package com.autoforge.backend.application.port.out;

import java.util.UUID;

public interface SseNotificationPort {
    void publish(UUID taskId, String eventName, Object payload);
}
```

```java
// application/port/out/FileStoragePort.java
package com.autoforge.backend.application.port.out;

import java.util.UUID;

public interface FileStoragePort {
    String uploadZip(UUID taskId, byte[] zipContent);
}
```

```java
// application/port/out/OAuthProviderPort.java
package com.autoforge.backend.application.port.out;

public interface OAuthProviderPort {
    OAuthUserInfo extractUserInfo(String provider, String idToken);

    record OAuthUserInfo(String provider, String oauthId, String email, String nickname) {}
}
```

---

## Application Services

### `CreateTaskService`

```java
package com.autoforge.backend.application.service;

import com.autoforge.backend.application.port.in.CreateTaskUseCase;
import com.autoforge.backend.application.port.out.AITaskRepositoryPort;
import com.autoforge.backend.domain.model.AITask;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.UUID;

@Service
public class CreateTaskService implements CreateTaskUseCase {

    private final AITaskRepositoryPort taskRepository;
    private final ApplicationEventPublisher eventPublisher;

    public CreateTaskService(AITaskRepositoryPort taskRepository, ApplicationEventPublisher eventPublisher) {
        this.taskRepository = taskRepository;
        this.eventPublisher = eventPublisher;
    }

    @Override
    @Transactional
    public UUID createTask(String requirements, UUID userId) {
        AITask task = new AITask(UUID.randomUUID(), userId, requirements);
        AITask saved = taskRepository.save(task);
        eventPublisher.publishEvent(new TaskRequestedEvent(saved.getId()));
        return saved.getId();
    }
}
```

### `AITaskWorkerService` (비동기 AI 작업 + Max Iteration 방어)

```java
package com.autoforge.backend.application.service;

import com.autoforge.backend.application.port.out.*;
import com.autoforge.backend.domain.exception.MaxIterationExceededException;
import com.autoforge.backend.domain.exception.TaskNotFoundException;
import com.autoforge.backend.domain.model.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.io.ByteArrayOutputStream;
import java.util.zip.ZipEntry;
import java.util.zip.ZipOutputStream;

@Service
public class AITaskWorkerService {

    private final AITaskRepositoryPort taskRepository;
    private final CodeSnapshotRepositoryPort snapshotRepository;
    private final ILLMOrchestratorClient orchestratorClient;
    private final FileStoragePort fileStorage;
    private final SseNotificationPort sseNotification;

    @Value("${agent.max-iteration:5}")
    private int maxIteration;

    public AITaskWorkerService(
        AITaskRepositoryPort taskRepository,
        CodeSnapshotRepositoryPort snapshotRepository,
        ILLMOrchestratorClient orchestratorClient,
        FileStoragePort fileStorage,
        SseNotificationPort sseNotification
    ) {
        this.taskRepository = taskRepository;
        this.snapshotRepository = snapshotRepository;
        this.orchestratorClient = orchestratorClient;
        this.fileStorage = fileStorage;
        this.sseNotification = sseNotification;
    }

    @Async("aiTaskExecutor")
    @EventListener
    @Transactional
    public void handleTaskRequested(TaskRequestedEvent event) {
        AITask task = taskRepository.findById(event.taskId())
            .orElseThrow(() -> new TaskNotFoundException(event.taskId()));

        try {
            task.startProcessing();
            taskRepository.save(task);
            sseNotification.publish(task.getId(), "PROCESSING", null);

            CodeSnapshot best = orchestratorClient.executeWhiteboardLoop(task, maxIteration);

            byte[] zip = buildZip(best.getSourceCode());
            String artifactUrl = fileStorage.uploadZip(task.getId(), zip);

            best.markAsBest();
            snapshotRepository.clearBestVersion(task.getId());
            snapshotRepository.save(best);

            task.completeTask(best.getEvaluationScore().calculateAverage(), artifactUrl);
            taskRepository.save(task);
            sseNotification.publish(task.getId(), "COMPLETED", artifactUrl);

        } catch (MaxIterationExceededException e) {
            snapshotRepository.findBestByTaskId(task.getId())
                .ifPresent(s -> sseNotification.publish(task.getId(), "MAX_ITER_EXCEEDED", s.getIterationCount()));
            task.failTask();
            taskRepository.save(task);
            sseNotification.publish(task.getId(), "FAILED", "max_iteration_exceeded");

        } catch (Exception e) {
            task.failTask();
            taskRepository.save(task);
            sseNotification.publish(task.getId(), "FAILED", e.getMessage());
        }
    }

    private byte[] buildZip(String sourceCode) throws Exception {
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        try (ZipOutputStream zos = new ZipOutputStream(bos)) {
            ZipEntry entry = new ZipEntry("generated/Main.java");
            zos.putNextEntry(entry);
            zos.write(sourceCode.getBytes());
            zos.closeEntry();
        }
        return bos.toByteArray();
    }
}
```

---

## Global — Async Config (스레드풀 명시)

```java
package com.autoforge.backend.global.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "aiTaskExecutor")
    public Executor aiTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("ai-worker-");
        executor.setRejectedExecutionHandler(new java.util.concurrent.ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

---

## Global — Exception Handler (도메인 예외 → HTTP 매핑)

```java
package com.autoforge.backend.global.exception;

import com.autoforge.backend.domain.exception.*;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import java.time.Instant;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(TaskNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(TaskNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("TASK_NOT_FOUND", e.getMessage(), Instant.now()));
    }

    @ExceptionHandler(TaskStateException.class)
    public ResponseEntity<ErrorResponse> handleConflict(TaskStateException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse("TASK_STATE_CONFLICT", e.getMessage(), Instant.now()));
    }

    @ExceptionHandler(MaxIterationExceededException.class)
    public ResponseEntity<ErrorResponse> handleMaxIter(MaxIterationExceededException e) {
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)
            .body(new ErrorResponse("MAX_ITERATION_EXCEEDED", e.getMessage(), Instant.now()));
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleBadRequest(IllegalArgumentException e) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse("BAD_REQUEST", e.getMessage(), Instant.now()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleInternal(Exception e) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("INTERNAL_ERROR", "Unexpected error occurred.", Instant.now()));
    }
}
```

```java
package com.autoforge.backend.global.exception;

import java.time.Instant;

public record ErrorResponse(
    String code,
    String message,
    Instant timestamp
) {}
```

---

## Auth Flow — OAuth2 + JWT

### 로그인 흐름

```
1. 클라이언트: POST /api/v1/auth/login  body: { provider: "GOOGLE", idToken: "..." }
2. AuthController → AuthService
3. AuthService → OAuthProviderPort.extractUserInfo(provider, idToken)
   - GOOGLE: Google TokenInfo API로 id_token 검증
   - APPLE: Apple Public Key로 JWT 서명 검증
4. users 테이블에서 oauth_id 조회 → 없으면 신규 생성 (upsert)
5. Access Token (15분) + Refresh Token (30일) 발급
6. Refresh Token Hash → Redis 저장 (TTL 30일)
7. 응답: { accessToken } + Set-Cookie: refreshToken (HttpOnly, Secure)
```

### RTR (Refresh Token Rotation)

```
1. 클라이언트: POST /api/v1/auth/refresh  (쿠키에 refreshToken 자동 포함)
2. AuthService: 쿠키에서 Refresh Token 추출 → Hash 계산
3. Redis에서 해당 userId의 저장된 Hash와 비교
4-A. 일치: 새 Access + Refresh Token 발급 → 기존 Redis 키 파기 → 새 Hash 저장
4-B. 불일치(재사용 감지): deleteAllTokens(userId) → 해당 유저 전체 세션 강제 만료 → 401
```

### JWT Filter 흐름

```
HTTP Request
  → JwtAuthenticationFilter
  → "Authorization: Bearer <token>" 추출
  → JwtTokenProvider.validateSignature(token)   ← DB 조회 없음
  → 만료 → 401 Unauthorized
  → 유효 → SecurityContextHolder에 Authentication 설정
  → Controller 진입
```

---

## SSE + Redis Pub/Sub 흐름

```
AI Worker (aiTaskExecutor 스레드)
  → SseNotificationPort.publish(taskId, event, payload)
  → RedisSseAdapter: Redis Channel "sse:{taskId}" 에 Publish

Redis Pub/Sub
  → RedisMessageListener (Main 스레드풀)
  → SseConnectionManager.getEmitter(taskId)
  → SseEmitter.send(event)
  → 클라이언트 수신
```

---

## API Endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/api/v1/auth/login` | None | OAuth2 idToken → JWT 발급 |
| `POST` | `/api/v1/auth/refresh` | Cookie | RTR — Access Token 갱신 |
| `POST` | `/api/v1/auth/logout` | Bearer | Refresh Token 파기 |
| `POST` | `/api/v1/tasks` | Bearer | AI 작업 생성 (202 Accepted) |
| `GET` | `/api/v1/tasks` | Bearer | 내 작업 목록 조회 |
| `GET` | `/api/v1/tasks/{taskId}` | Bearer | 작업 상세 조회 |
| `GET` | `/api/v1/tasks/{taskId}/snapshots` | Bearer | 코드 스냅샷 목록 조회 |
| `GET` | `/api/v1/tasks/{taskId}/stream` | Bearer | SSE 실시간 진행률 스트림 |

---

## Environment Variables

| Variable | Description | Default |
|---|---|---|
| `SPRING_PROFILES_ACTIVE` | 활성 프로파일 | `local` |
| `SPRING_DATASOURCE_URL` | PostgreSQL JDBC URL | — |
| `SPRING_DATASOURCE_USERNAME` | DB 계정명 | — |
| `SPRING_DATASOURCE_PASSWORD` | DB 비밀번호 | — |
| `SPRING_DATA_REDIS_HOST` | Redis 호스트 | `localhost` |
| `SPRING_DATA_REDIS_PORT` | Redis 포트 | `6379` |
| `JWT_SECRET` | JWT 서명 시크릿 키 | — |
| `JWT_ACCESS_TTL_MS` | Access Token 유효시간 (ms) | `900000` |
| `JWT_REFRESH_TTL_SEC` | Refresh Token Redis TTL (sec) | `2592000` |
| `LLM_OPENAI_API_KEY` | OpenAI API Key | — |
| `LLM_ANTHROPIC_API_KEY` | Anthropic API Key | — |
| `S3_BUCKET_NAME` | S3 버킷명 | — |
| `S3_REGION` | S3 리전 | `ap-northeast-2` |
| `AWS_ACCESS_KEY_ID` | AWS 자격증명 | — |
| `AWS_SECRET_ACCESS_KEY` | AWS 자격증명 | — |
| `AGENT_MAX_ITERATION` | AI 루프 최대 반복 횟수 | `5` |

---

## Getting Started

```bash
git clone https://github.com/creatorjun/AutoForge.git
cd autoforge-backend

cp .env.example .env
# .env 파일 내 시크릿 값 입력

docker compose up --build -d

docker compose logs -f api
```
