<!--
Sync Impact Report
==================
Version change: (new) → 1.0.0
Added sections:
  - I. 요리사 중심 인덱싱 (Chef-Centric Indexing)
  - II. LLM 기반 데이터 정형화 (LLM-Based Data Normalization)
  - III. 중복 검사 우선 (Deduplication-First)
  - IV. 듀얼 모드 아키텍처 (Dual-Mode Architecture)
  - V. 재시도 및 장애 대응 (Retry & Fallback)
  - 기술 제약 사항 (Technical Constraints)
  - 개발 워크플로우 (Development Workflow)
  - 거버넌스 (Governance)
Templates requiring updates:
  - .specify/templates/plan-template.md: ⚠ pending (Constitution Check 섹션 추가 필요)
  - .specify/templates/spec-template.md: ⚠ pending (원칙 정렬 필요)
  - .specify/templates/tasks-template.md: ⚠ pending (작업 분류 정렬 필요)
  - .specify/templates/checklist-template.md: ⚠ pending (품질 게이트 정렬 필요)
Follow-up TODOs:
  - 템플릿 파일들에 Constitution Check 섹션 추가 검토
-->

# Naecipe AI Crawler Constitution

## Core Principles

### I. 요리사 중심 인덱싱 (Chef-Centric Indexing)

모든 크롤링은 **요리사(Chef) 기준**으로 수행한다. 키워드 기반 무차별 수집은 금지한다.

- 크롤링 대상은 반드시 `crawl_targets` 테이블에 등록된 요리사/채널이어야 한다
- 티어 시스템(S/A/B/C)에 따른 우선순위 적용이 필수이다
- 신규 요리사 발견 시 반드시 티어 평가 후 등록해야 한다
- 키워드 기반 크롤링은 요리사 발견 목적으로만 허용되며, 레시피 직접 수집에는 사용하지 않는다

**근거**: DB 스키마가 `chefs` → `recipes` 관계로 설계되어 있으며, 품질 관리와 리소스 효율성을 위해 신뢰할 수 있는 소스에 집중한다.

### II. LLM 기반 데이터 정형화 (LLM-Based Data Normalization)

모든 비정형 콘텐츠(영상 자막, 블로그 텍스트)는 LLM을 통해 표준화된 레시피 스키마로 변환한다.

- GPT-4 Turbo를 Primary LLM으로 사용한다
- Claude 3 Sonnet을 Fallback LLM으로 유지한다
- 파싱 결과는 반드시 Pydantic v2 모델로 검증한다
- 파싱 성공률 95% 이상을 유지해야 한다

**필수 추출 필드**:
- 제목, 설명, 재료(이름/양/단위), 조리 단계(순서/지시/시간)
- 소요 시간, 인분 수, 난이도, 태그

**근거**: 다양한 소스의 비정형 데이터를 일관된 구조로 저장하여 검색 및 활용 품질을 보장한다.

### III. 중복 검사 우선 (Deduplication-First)

크롤링 전 중복 검사를 수행하여 리소스 낭비를 방지한다.

**중복 검사 단계**:
1. **로컬 캐시**: `content_hash` 매칭 (100% 일치 시 스킵)
2. **DB 정확 매칭**: 제목 + 저자명 해시 (100% 일치 시 중복)
3. **유사도 검사**: 재료 + 조리법 임베딩 코사인 유사도 (≥0.85 시 중복)
4. **URL 검사**: 동일 `source_url` 존재 여부

**근거**: 불필요한 LLM API 호출과 DB 작업을 최소화하여 비용과 시간을 절감한다.

### IV. 듀얼 모드 아키텍처 (Dual-Mode Architecture)

로컬 모드와 API 모드를 모두 지원하여 개발과 운영 환경을 분리한다.

- **로컬 모드** (`--mode local`): 로컬 PostgreSQL 직접 저장, 개발/초기 수집용
- **API 모드** (`--mode api`): Ingestion API 통신, 운영 환경용
- **스케줄러 모드** (`--mode schedule`): APScheduler 기반 자동 크롤링

모드 전환은 `STORAGE_MODE` 환경 변수 또는 CLI 플래그로 제어한다.

**근거**: 서버 구축 전 초기 데이터 수집과 운영 모드를 동일한 코드베이스로 지원한다.

### V. 재시도 및 장애 대응 (Retry & Fallback)

모든 외부 연동은 재시도 로직과 장애 대응 전략을 갖춘다.

**재시도 정책**:
- LLM API: 최대 3회, 지수 백오프 (1s, 2s, 4s)
- DB/API: 최대 3회, 지수 백오프
- 크롤링 타임아웃: 30초

**장애 대응**:
- Primary LLM 실패 → Fallback LLM 자동 전환
- 로컬 DB 장애 → JSON 파일 임시 저장 후 재시도
- Ingestion API 장애 → 로컬 큐 저장 후 재시도

**근거**: 외부 의존성 장애로 인한 데이터 손실을 방지하고 시스템 안정성을 보장한다.

## 기술 제약 사항 (Technical Constraints)

### 기술 스택

| 영역 | 기술 | 비고 |
|------|------|------|
| 언어 | Python 3.11+ | 타입 힌트 필수 |
| AI 프레임워크 | LangGraph | 워크플로우 제어, 상태 관리 |
| 크롤링 | Playwright (headless) | 동적 페이지 렌더링 |
| HTTP | httpx (async) | API 통신 |
| DB | PostgreSQL + psycopg3 | 로컬/운영 공통 |
| 검증 | Pydantic v2 | 스키마 검증 |
| 스케줄러 | APScheduler | 크롤링 주기 관리 |

### 성능 목표

| 메트릭 | 목표 |
|--------|------|
| 레시피 파싱 시간 | < 10초/개 |
| 일일 크롤링 용량 | > 1,000개 레시피 |
| LLM 파싱 성공률 | > 95% |
| 중복 판정 정확도 | > 99% |

### 코딩 컨벤션

- 모든 코드 주석, 커밋 메시지, 문서화는 **한글**로 작성
- 변수명, 함수명은 **영어**로 작성하되 의미가 명확해야 함
- async/await 패턴으로 비동기 처리
- tenacity 라이브러리로 재시도 로직 구현
- 구조화된 로깅 (structlog 권장)

## 개발 워크플로우 (Development Workflow)

### 파이프라인 단계

```
DISCOVER (소스 탐색) → EXTRACT (콘텐츠 추출) → PARSE (LLM 파싱)
→ CHECK_DUPLICATE (중복 검사) → SAVE (저장) → END
```

### 코드 리뷰 요구사항

- 중복 검사 로직 변경 시 반드시 테스트 케이스 포함
- LLM 프롬프트 변경 시 파싱 성공률 검증 필수
- 새 플랫폼 크롤러 추가 시 `base_crawler.py` 상속 준수

### 품질 게이트

- 모든 PR은 타입 체크 통과 필수 (mypy)
- 새 기능은 단위 테스트 포함 필수
- 크롤러 변경 시 통합 테스트 필수

## Governance

본 Constitution은 프로젝트의 모든 개발 관행보다 우선한다.

### 개정 절차

1. 개정 사유와 영향 범위 문서화
2. 관련 템플릿 파일 동기화 검토
3. 버전 업데이트 (Major: 원칙 삭제/재정의, Minor: 원칙 추가, Patch: 문구 수정)
4. `LAST_AMENDED_DATE` 갱신

### 준수 검토

- 모든 PR은 Constitution 원칙 준수 여부 확인
- 원칙 위반 시 정당한 사유 없이는 승인 불가
- 복잡성 추가 시 반드시 근거 명시

### 런타임 가이드

개발 시 세부 가이드는 `CLAUDE.md` 참조

**Version**: 1.0.0 | **Ratified**: 2025-12-11 | **Last Amended**: 2025-12-11
