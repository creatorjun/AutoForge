# db.md

# Database Schema — AutoForge (PostgreSQL 16)

## DDL

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE users (
    user_id        UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    oauth_provider VARCHAR(20)  NOT NULL,
    oauth_id       VARCHAR(255) NOT NULL UNIQUE,
    email          VARCHAR(255),
    nickname       VARCHAR(50),
    role           VARCHAR(20)  DEFAULT 'USER',
    created_at     TIMESTAMPTZ  DEFAULT CURRENT_TIMESTAMP,
    updated_at     TIMESTAMPTZ  DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_users_oauth ON users(oauth_provider, oauth_id);

CREATE TABLE ai_tasks (
    task_id            UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id            UUID         NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    requirements       TEXT         NOT NULL,
    status             VARCHAR(30)  NOT NULL,
    total_score        DECIMAL(5,2),
    final_artifact_url VARCHAR(512),
    created_at         TIMESTAMPTZ  DEFAULT CURRENT_TIMESTAMP,
    updated_at         TIMESTAMPTZ  DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_ai_tasks_user_id ON ai_tasks(user_id);
CREATE INDEX idx_ai_tasks_status  ON ai_tasks(status);

CREATE TABLE task_workflow_steps (
    step_id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id           UUID        NOT NULL REFERENCES ai_tasks(task_id) ON DELETE CASCADE,
    step_sequence     INT         NOT NULL,
    agent_type        VARCHAR(50) NOT NULL,
    prompt_input      TEXT        NOT NULL,
    response_output   TEXT,
    status            VARCHAR(20) NOT NULL,
    execution_time_ms BIGINT,
    created_at        TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_workflow_steps_task ON task_workflow_steps(task_id, step_sequence);

CREATE TABLE code_snapshots (
    snapshot_id       UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id           UUID    NOT NULL REFERENCES ai_tasks(task_id) ON DELETE CASCADE,
    iteration_count   INT     NOT NULL,
    source_code       TEXT    NOT NULL,
    evaluation_scores JSONB,
    feedback_notes    JSONB,
    is_best_version   BOOLEAN DEFAULT FALSE,
    created_at        TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_code_snapshots_task_iter ON code_snapshots(task_id, iteration_count);
```

## Table Relationships

```
users (1) ──< ai_tasks (1) ──< task_workflow_steps
                         (1) ──< code_snapshots
```

## Column Notes

### users
| Column | Description |
|---|---|
| `oauth_provider` | `GOOGLE` \| `APPLE` |
| `oauth_id` | OAuth 제공자가 발급하는 고유 식별자 |
| `role` | `USER` \| `ADMIN` |

### ai_tasks
| Column | Description |
|---|---|
| `status` | `PENDING` → `PROCESSING` → `COMPLETED` \| `FAILED` |
| `final_artifact_url` | S3 Pre-signed URL (Zip 파일) |

### task_workflow_steps
| Column | Description |
|---|---|
| `agent_type` | `ORCHESTRATOR` \| `ANALYZER` \| `DRAFTER` \| `SECURITY` \| `STABILITY` \| `PERFORMANCE` |
| `status` | `SUCCESS` \| `FAILED` \| `TIMEOUT` |

### code_snapshots
| Column | Description |
|---|---|
| `evaluation_scores` | `{"security": 85, "stability": 90, "performance": 75}` |
| `feedback_notes` | `{"security": "SQL Injection 가능성 있음", ...}` |
| `is_best_version` | 현재까지 가장 높은 점수의 스냅샷 여부 |
