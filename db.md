-- 확장 기능: UUID 자동 생성을 위한 pgcrypto 활성화
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ==========================================
-- 1. 사용자 계정 (Users)
-- ==========================================
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    oauth_provider VARCHAR(20) NOT NULL, -- 'GOOGLE', 'APPLE' 등
    oauth_id VARCHAR(255) NOT NULL UNIQUE, -- 제공자가 주는 고유 식별자
    email VARCHAR(255),
    nickname VARCHAR(50),
    role VARCHAR(20) DEFAULT 'USER',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_users_oauth ON users(oauth_provider, oauth_id);

-- ==========================================
-- 2. AI 작업 메인 테이블 (Aggregate Root)
-- ==========================================
-- 유저의 요청 한 건에 대한 전체 상태와 결과를 관리합니다.
CREATE TABLE ai_tasks (
    task_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    requirements TEXT NOT NULL,          -- 유저가 입력한 초기 요구사항
    status VARCHAR(30) NOT NULL,         -- PENDING, PROCESSING, COMPLETED, FAILED
    total_score DECIMAL(5,2),            -- 협의체가 매긴 최종 합산 점수
    final_artifact_url VARCHAR(512),     -- 완성된 코드의 압축 파일(S3 등) 다운로드 링크
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_ai_tasks_user_id ON ai_tasks(user_id);
CREATE INDEX idx_ai_tasks_status ON ai_tasks(status);

-- ==========================================
-- 3. 워크플로우 단계 로그 (Workflow Steps)
-- ==========================================
-- 오케스트레이터가 각 에이전트(보안, 성능 등)를 호출한 상세 이력입니다.
CREATE TABLE task_workflow_steps (
    step_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID NOT NULL REFERENCES ai_tasks(task_id) ON DELETE CASCADE,
    step_sequence INT NOT NULL,          -- 실행 순서 (1, 2, 3...)
    agent_type VARCHAR(50) NOT NULL,     -- ORCHESTRATOR, ANALYZER, DRAFTER, SECURITY...
    prompt_input TEXT NOT NULL,          -- 모델에게 전달된 프롬프트
    response_output TEXT,                -- 모델이 답변한 원문 (수정 요청사항 등)
    status VARCHAR(20) NOT NULL,         -- SUCCESS, FAILED, TIMEOUT
    execution_time_ms BIGINT,            -- AI 응답에 걸린 시간
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_workflow_steps_task ON task_workflow_steps(task_id, step_sequence);

-- ==========================================
-- 4. 화이트보드 코드 스냅샷 (Code Snapshots)
-- ==========================================
-- AI가 코드를 수정할 때마다 버전을 기록하여, 가장 점수가 높은 버전을 추적합니다.
CREATE TABLE code_snapshots (
    snapshot_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID NOT NULL REFERENCES ai_tasks(task_id) ON DELETE CASCADE,
    iteration_count INT NOT NULL,        -- 몇 번째 수정 버전인지 (1, 2, 3...)
    source_code TEXT NOT NULL,           -- 해당 시점의 전체 소스 코드
    
    -- 각 에이전트의 평가 결과 (유연성을 위해 JSONB 사용)
    -- 예: {"security": 85, "performance": 90, "stability": 75}
    evaluation_scores JSONB,             
    
    -- 피드백 상세 내용 (수정해야 할 부분 등)
    feedback_notes JSONB,                
    
    is_best_version BOOLEAN DEFAULT FALSE, -- 현재까지 가장 완성도가 높은 버전인지 여부
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_code_snapshots_task_iter ON code_snapshots(task_id, iteration_count);