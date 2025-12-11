# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

**naecipe_ai_crawler** - 내시피(Naecipe) 서비스의 레시피 크롤러 Bot

외부 플랫폼(YouTube, Instagram, 블로그, 레시피 사이트)에서 유명 쉐프/인플루언서의 레시피를 수집하고, LLM을 활용하여 구조화된 형태로 변환한 후 로컬 DB 또는 Ingestion API를 통해 저장하는 독립 실행형 크롤러 에이전트.

### 서비스 컨텍스트

내시피(Naecipe)는 "나만의 레시피"를 의미하는 AI 기반 개인화 요리 플랫폼이다. 본 크롤러는 사용자가 검색할 수 있는 **원본 레시피 데이터베이스**를 구축하기 위한 데이터 파이프라인의 핵심 컴포넌트이다.

## 운영 모드 아키텍처

본 프로젝트는 **2가지 운영 모드**를 지원한다:

### 모드 1: 로컬 직접 수집 모드 (서버 구축 전)

Claude Code를 활용하여 로컬에서 직접 크롤링 → 정규화 → 로컬 DB 인제스트하는 파이프라인.
서버가 구축되기 전 초기 데이터 수집 용도.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        로컬 직접 수집 모드 (Phase 1)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐               │
│   │ 외부 소스     │     │ Claude Code  │     │ 로컬 DB      │               │
│   │              │     │ (에이전트)    │     │ (PostgreSQL) │               │
│   │ • YouTube    │     │              │     │              │               │
│   │ • Instagram  │ ──▶ │ • 크롤링     │ ──▶ │ • recipes    │               │
│   │ • 블로그     │     │ • LLM 파싱   │     │ • ingredients│               │
│   │ • 레시피사이트│     │ • 정규화     │     │ • steps      │               │
│   └──────────────┘     │ • 중복검사   │     │ • tags       │               │
│                        │ • DB 저장    │     │ • sources    │               │
│                        └──────────────┘     └──────────────┘               │
│                                                                             │
│   실행: python main.py --mode local --platform youtube                      │
│   또는: Claude Code 대화형으로 직접 수집 명령                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 모드 2: API 연동 모드 (서버 구축 후)

Ingestion API를 통해 백엔드 서비스와 통신하는 운영 모드.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         API 연동 모드 (Phase 2)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌───────┐ │
│   │ 외부 소스     │     │ Crawler Bot  │     │ Ingestion    │     │Recipe │ │
│   │              │     │              │     │ API          │     │DB     │ │
│   │ • YouTube    │     │ • 크롤링     │     │              │     │       │ │
│   │ • Instagram  │ ──▶ │ • LLM 파싱   │ ──▶ │ • 스키마검증 │ ──▶ │ (운영)│ │
│   │ • 블로그     │     │ • 정규화     │     │ • 중복검사   │     │       │ │
│   │ • 레시피사이트│     │ • 전송      │     │ • 스코어계산 │     │       │ │
│   └──────────────┘     └──────────────┘     └──────────────┘     └───────┘ │
│                                                                             │
│   실행: python main.py --mode api --platform youtube                        │
│   또는: python main.py --mode schedule (스케줄러)                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 전체 파이프라인 흐름도

```
                              ┌─────────────────────┐
                              │    외부 레시피 소스   │
                              │  YouTube/Instagram  │
                              │   블로그/레시피사이트  │
                              └──────────┬──────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │   1. 소스 탐색       │
                              │   (discover)        │
                              │   인기 콘텐츠 발견   │
                              └──────────┬──────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │   2. 콘텐츠 추출     │
                              │   (extract)         │
                              │   자막/텍스트/메타   │
                              └──────────┬──────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │   3. LLM 파싱       │
                              │   (parse)           │
                              │   GPT-4 / Claude    │
                              │   구조화된 레시피로  │
                              └──────────┬──────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │   4. 중복 검사       │
                              │   (deduplicate)     │
                              │   해시/유사도 검사   │
                              └──────────┬──────────┘
                                         │
                         ┌───────────────┴───────────────┐
                         │                               │
                         ▼                               ▼
              ┌─────────────────────┐       ┌─────────────────────┐
              │   로컬 모드          │       │   API 모드          │
              │   (--mode local)    │       │   (--mode api)      │
              │                     │       │                     │
              │   5a. 로컬 DB 저장   │       │   5b. API 전송      │
              │   PostgreSQL 직접   │       │   Ingestion API     │
              │   INSERT/UPDATE     │       │   POST/PATCH        │
              └─────────┬───────────┘       └─────────┬───────────┘
                        │                             │
                        ▼                             ▼
              ┌─────────────────────┐       ┌─────────────────────┐
              │   로컬 PostgreSQL    │       │   운영 Recipe DB    │
              │   (개발/초기수집)     │       │   (프로덕션)         │
              └─────────────────────┘       └─────────────────────┘
```

## 핵심 기능

### 1. 소스별 콘텐츠 크롤링
- **YouTube**: 유명 요리 채널 (백종원, 샘킴 등) 영상 + 자막 추출
- **Instagram**: 요리 인플루언서 피드 수집
- **블로그**: 네이버 블로그, 티스토리 레시피 포스트
- **레시피 사이트**: 만개의레시피, 해먹남녀 등

### 2. LLM 기반 레시피 파싱
- 비정형 콘텐츠(영상 자막, 텍스트)를 구조화된 레시피 형태로 변환
- 추출 항목: 제목, 재료(이름/양/단위), 조리 단계, 소요시간, 난이도, 태그

### 3. 중복 검사 (로컬 캐시 + DB/API)
- 제목 + 저자명 해시 매칭 (정확 매칭)
- 콘텐츠 해시로 사전 필터링 (로컬)
- DB 직접 조회 또는 Ingestion API 호출로 최종 중복 판정

### 4. 저장소 연동
- **로컬 모드**: 로컬 PostgreSQL에 직접 INSERT/UPDATE
- **API 모드**: Ingestion API를 통한 저장

## 기술 스택

| 영역 | 기술 | 비고 |
|------|------|------|
| **언어** | Python 3.11+ | 타입 힌트 필수 |
| **AI 프레임워크** | LangGraph | 워크플로우 제어, 상태 관리 |
| **LLM** | OpenAI GPT-4 Turbo (Primary) | 레시피 파싱 |
| **LLM Fallback** | Anthropic Claude 3 Sonnet | 장애 대응 |
| **크롤링** | Playwright (headless) | 동적 페이지 렌더링 |
| **HTTP Client** | httpx (async) | API 통신 |
| **로컬 DB** | PostgreSQL + psycopg3 | 로컬 직접 저장 |
| **ORM** | SQLAlchemy 2.0 (선택) | DB 추상화 |
| **스케줄러** | APScheduler | 크롤링 주기 관리 |
| **데이터 검증** | Pydantic v2 | 스키마 검증 |

## 디렉토리 구조 (권장)

```
naecipe_ai_crawler/
├── src/
│   ├── agents/
│   │   └── crawler_agent.py      # LangGraph 기반 크롤러 에이전트
│   ├── crawlers/
│   │   ├── base_crawler.py       # 크롤러 베이스 클래스
│   │   ├── youtube_crawler.py    # YouTube 크롤러
│   │   ├── instagram_crawler.py  # Instagram 크롤러
│   │   └── blog_crawler.py       # 블로그 크롤러 (네이버, 티스토리)
│   ├── parsers/
│   │   └── recipe_parser.py      # LLM 레시피 파싱 로직
│   ├── storage/
│   │   ├── base_storage.py       # 저장소 인터페이스 (추상 클래스)
│   │   ├── local_storage.py      # 로컬 PostgreSQL 직접 저장
│   │   └── api_storage.py        # Ingestion API 클라이언트
│   ├── db/
│   │   ├── connection.py         # DB 연결 관리
│   │   ├── models.py             # SQLAlchemy 모델 (선택)
│   │   └── migrations/           # DB 마이그레이션 스크립트
│   ├── models/
│   │   ├── recipe.py             # 레시피 Pydantic 모델
│   │   ├── crawler_state.py      # LangGraph 상태 모델
│   │   └── api_schemas.py        # API 요청/응답 스키마
│   ├── scheduler/
│   │   └── crawl_scheduler.py    # 크롤링 스케줄러
│   └── utils/
│       ├── config.py             # 설정 관리
│       ├── logger.py             # 로깅 설정
│       └── hash.py               # 해시 유틸리티
├── scripts/
│   ├── init_local_db.py          # 로컬 DB 초기화 스크립트
│   ├── migrate_to_api.py         # 로컬→API 마이그레이션
│   └── export_recipes.py         # 레시피 내보내기
├── tests/
│   ├── unit/
│   └── integration/
├── config/
│   ├── channels.yaml             # 크롤링 대상 채널 설정
│   └── local_db.yaml             # 로컬 DB 설정
├── data/
│   └── crawled/                  # 크롤링 임시 데이터 저장
├── main.py                       # 엔트리포인트
├── pyproject.toml
└── CLAUDE.md
```

## 데이터 모델

### 크롤링 콘텐츠 (CrawledContent)
```python
class CrawledContent(BaseModel):
    url: str
    platform: Literal["youtube", "instagram", "naver_blog", "tistory", "recipe_site"]
    title: str
    author_name: str
    author_channel: str
    raw_content: str
    thumbnail_url: Optional[str] = None
    video_url: Optional[str] = None
    metrics: dict = {}  # view_count, like_count, comment_count
```

### 파싱된 레시피 (ParsedRecipe)
```python
class ParsedRecipe(BaseModel):
    title: str
    description: str
    author_name: str
    author_channel: str
    ingredients: List[Ingredient]  # name, amount, unit, is_optional
    steps: List[CookingStep]       # step_number, instruction, duration_seconds
    cooking_time_minutes: Optional[int] = None
    servings: Optional[int] = None
    difficulty: Optional[Literal["easy", "medium", "hard"]] = None
    tags: List[str] = []
```

### Ingestion API 페이로드
```python
class RecipeIngestionPayload(BaseModel):
    title: str
    description: str
    author_name: str
    author_channel: str
    source_url: str
    source_platform: str
    ingredients: List[dict]
    steps: List[dict]
    cooking_time_minutes: Optional[int]
    servings: Optional[int]
    difficulty: Optional[str]
    tags: List[str]
    thumbnail_url: Optional[str]
    video_url: Optional[str]
    content_hash: str
    source_metrics: dict
```

## 워크플로우 (LangGraph)

```
START → DISCOVER (소스 탐색) → EXTRACT (콘텐츠 추출) → PARSE (LLM 파싱)
→ CHECK_DUPLICATE (중복 검사) → SAVE (로컬 DB 저장 또는 API 전송) → END
```

### 상태 모델 (CrawlerState)
```python
class CrawlerState(BaseModel):
    # Input
    platform: SourcePlatform
    target_channels: List[str] = []
    storage_mode: Literal["local", "api"] = "local"  # 저장 모드

    # Processing
    discovered_contents: List[CrawledContent] = []
    current_content: Optional[CrawledContent] = None
    parsed_recipe: Optional[ParsedRecipe] = None
    content_hash: Optional[str] = None

    # Output
    processed_count: int = 0
    success_count: int = 0
    duplicate_count: int = 0
    failed_count: int = 0
    results: List[dict] = []
```

## 실행 모드

### 1. 로컬 직접 수집 모드 (서버 구축 전 - Claude Code 활용)
```bash
# 로컬 DB 초기화 (최초 1회)
python scripts/init_local_db.py

# YouTube 레시피 수집 → 로컬 DB 저장
python main.py --mode local --platform youtube

# 특정 채널만 수집
python main.py --mode local --platform youtube --channels "백종원의요리비책,쿠킹로그"

# 모든 플랫폼 수집
python main.py --mode local --platform all

# 단일 URL 수집 (Claude Code 대화형)
python main.py --mode local --url "https://youtube.com/watch?v=..."
```

### 2. API 연동 모드 (서버 구축 후)
```bash
# 1회 실행
python main.py --mode api --platform youtube

# 스케줄러 모드 (운영)
python main.py --mode schedule
```
- YouTube: 매일 새벽 2시
- Instagram: 매일 새벽 3시
- 블로그: 매일 새벽 4시
- 점수 갱신: 매주 일요일 새벽 5시

### 3. 마이그레이션 (로컬 → API)
```bash
# 로컬 DB 데이터를 운영 DB로 마이그레이션
python scripts/migrate_to_api.py --batch-size 100
```

## 환경 변수

```env
# 운영 모드
STORAGE_MODE=local  # local 또는 api

# LLM 설정
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...

# 로컬 DB 설정 (로컬 모드)
LOCAL_DB_HOST=localhost
LOCAL_DB_PORT=5432
LOCAL_DB_NAME=naecipe_recipes
LOCAL_DB_USER=postgres
LOCAL_DB_PASSWORD=password

# Ingestion API (API 모드)
INGESTION_API_URL=https://api.naecipe.com/v1/ingestion
INGESTION_API_KEY=internal-api-key

# YouTube API
YOUTUBE_API_KEY=...

# 크롤링 설정
CRAWL_MAX_RESULTS_PER_CHANNEL=50
CRAWL_RETRY_COUNT=3
CRAWL_TIMEOUT_SECONDS=30

# 로깅
LOG_LEVEL=INFO
```

## 로컬 DB 스키마 (PostgreSQL)

```sql
-- 로컬 개발/초기 수집용 스키마
-- 운영 DB 스키마와 동일한 구조 유지

CREATE TABLE recipes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(200) NOT NULL,
    author_name VARCHAR(100),
    author_channel VARCHAR(200),
    source_url TEXT,
    source_platform VARCHAR(50),
    description TEXT,
    cooking_time_minutes INTEGER,
    servings INTEGER DEFAULT 2,
    difficulty VARCHAR(20),
    thumbnail_url TEXT,
    video_url TEXT,
    content_hash VARCHAR(64),
    quality_score DECIMAL(3,2) DEFAULT 0,
    popularity_score DECIMAL(3,2) DEFAULT 0,
    is_migrated BOOLEAN DEFAULT false,  -- 운영 DB로 마이그레이션 여부
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ingredients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipe_id UUID REFERENCES recipes(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    amount VARCHAR(50),
    unit VARCHAR(30),
    order_index INTEGER NOT NULL,
    is_optional BOOLEAN DEFAULT false
);

CREATE TABLE cooking_steps (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipe_id UUID REFERENCES recipes(id) ON DELETE CASCADE,
    step_number INTEGER NOT NULL,
    instruction TEXT NOT NULL,
    duration_seconds INTEGER,
    tips TEXT
);

CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) NOT NULL UNIQUE,
    category VARCHAR(30)
);

CREATE TABLE recipe_tags (
    recipe_id UUID REFERENCES recipes(id) ON DELETE CASCADE,
    tag_id UUID REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (recipe_id, tag_id)
);

CREATE TABLE recipe_sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipe_id UUID REFERENCES recipes(id) ON DELETE CASCADE,
    platform VARCHAR(50) NOT NULL,
    source_url TEXT NOT NULL,
    platform_view_count BIGINT DEFAULT 0,
    platform_like_count INTEGER DEFAULT 0,
    platform_comment_count INTEGER DEFAULT 0,
    crawled_at TIMESTAMPTZ DEFAULT NOW()
);

-- 인덱스
CREATE INDEX idx_recipes_content_hash ON recipes(content_hash);
CREATE INDEX idx_recipes_author_title ON recipes(author_name, title);
CREATE INDEX idx_recipes_source_url ON recipes(source_url);
CREATE INDEX idx_recipes_is_migrated ON recipes(is_migrated);
```

## 중복 검사 로직

| 단계 | 방식 | 임계값 |
|------|------|--------|
| 1. 로컬 캐시 | content_hash 매칭 | 100% 일치 시 스킵 |
| 2. DB/API 정확 매칭 | 제목 + 저자명 해시 | 100% 일치 시 중복 |
| 3. DB/API 유사도 검사 | 재료 + 조리법 임베딩 코사인 유사도 | ≥0.85 시 중복 |
| 4. URL 검사 | 동일 source_url 존재 여부 | URL 일치 시 중복 |

## 스코어링 시스템

크롤링 시 수집된 플랫폼 메트릭스로 레시피 점수 계산:

```
종합 점수 = (인기도 * 0.4) + (품질 * 0.3) + (신선도 * 0.2) + (소스 신뢰도 * 0.1)
```

- **인기도**: 조회수, 좋아요, 댓글 수 기반
- **품질**: 재료 수, 조리 단계 수, 이미지 유무, 시간 정보 유무
- **신선도**: 크롤링 일자 기준 감쇠
- **소스 신뢰도**: 플랫폼별 가중치 (YouTube > 레시피사이트 > 블로그)

## Claude Code 대화형 수집 가이드

Claude Code를 통해 대화형으로 레시피를 수집할 수 있다:

```
사용자: "백종원 유튜브에서 김치찌개 레시피 수집해줘"

Claude Code:
1. YouTube에서 "백종원 김치찌개" 검색
2. 영상 URL에서 자막 추출
3. LLM으로 레시피 파싱
4. 중복 검사 (로컬 DB 조회)
5. 로컬 PostgreSQL에 저장
6. 결과 보고

사용자: "만개의레시피에서 인기 레시피 10개 수집해줘"

Claude Code:
1. 만개의레시피 인기 순위 크롤링
2. 상위 10개 레시피 페이지 추출
3. LLM으로 각각 파싱
4. 중복 검사 후 저장
5. 결과 보고
```

## 언어 규칙

- **모든 코드 주석은 한글로 작성**
- **커밋 메시지는 한글로 작성**
- **문서화(docstring, README 등)는 한글로 작성**
- **변수명, 함수명은 영어로 작성하되 의미가 명확해야 함**

## 코딩 컨벤션

- Python 3.11+ 기능 적극 활용 (match-case, type hints)
- Pydantic v2 모델로 데이터 검증
- async/await 패턴으로 비동기 처리
- tenacity 라이브러리로 재시도 로직 구현
- 구조화된 로깅 (structlog 권장)

## 에러 처리

### 재시도 정책
- LLM API 호출: 최대 3회, 지수 백오프 (1s, 2s, 4s)
- DB/API 호출: 최대 3회, 지수 백오프
- 크롤링 타임아웃: 30초

### 장애 대응
- Primary LLM (GPT-4) 실패 시 → Fallback LLM (Claude 3) 자동 전환
- 로컬 DB 장애 시 → JSON 파일로 임시 저장 후 재시도
- Ingestion API 장애 시 → 로컬 큐에 저장 후 재시도

## 성능 목표

| 메트릭 | 목표 |
|--------|------|
| 레시피 파싱 시간 | < 10초/개 |
| 일일 크롤링 용량 | > 1,000개 레시피 |
| LLM 파싱 성공률 | > 95% |
| 중복 판정 정확도 | > 99% |

## 개발 로드맵

```
Phase 1: 로컬 직접 수집 (현재)
├── 로컬 PostgreSQL 설정
├── Claude Code 대화형 수집
├── 기본 크롤러 구현 (YouTube, 만개의레시피)
└── 초기 데이터 1,000개 수집

Phase 2: API 연동 준비
├── Ingestion API 클라이언트 구현
├── 로컬→API 마이그레이션 스크립트
└── 중복 검사 API 연동

Phase 3: 운영 모드
├── 스케줄러 모드 활성화
├── 추가 플랫폼 크롤러 (Instagram, 블로그)
└── 모니터링 및 알림
```

## 관련 문서

- [5-1-3_AI_AGENT.md](../naecipe_plan/5-1-3_AI_AGENT.md) - AI 에이전트 아키텍처 (Crawler Agent 섹션)
- [5-1-2_SYSTEM.md](../naecipe_plan/5-1-2_SYSTEM.md) - 시스템 아키텍처 (원본 레시피 수집 파이프라인)
- [5-1-4_API.md](../naecipe_plan/5-1-4_API.md) - Ingestion API 명세
- [5-1-1_DOMAIN.md](../naecipe_plan/5-1-1_DOMAIN.md) - 도메인 분석 (Recipe Domain)
