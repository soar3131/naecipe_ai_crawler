# SPECKIT_TODO.md - 레시피 크롤링 → DB 저장 스펙

이 문서는 `naecipe_ai_crawler` 프로젝트의 크롤링 데이터를 `naecipe_backend` DB에 저장하기 위한 단계별 스펙을 정의합니다.

---

## 1. 프로젝트 개요

### 1.1 목적
- 외부 플랫폼(YouTube, Instagram, 네이버 블로그, 레시피 사이트)에서 레시피를 크롤링
- LLM을 통해 정형화된 레시피 데이터로 변환
- `naecipe_backend` DB에 저장
- **생성형 AI를 통한 저작권 준수 이미지 생성**

### 1.2 아키텍처 위치
```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           naecipe_ai_crawler                                  │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                     1. 사전 검증 레이어                                    │ │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────────────┐ │ │
│  │  │ URL 수집/검증  │ → │ 사전 중복검사  │ → │  크롤링 대상 필터링           │ │ │
│  │  │              │   │ (DB 조회)     │   │  (이미 존재하면 스킵)          │ │ │
│  │  └──────────────┘   └──────────────┘   └──────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                     2. 크롤링 인프라 레이어                               │ │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────────────┐ │ │
│  │  │ 프록시 매니저  │ → │ 도메인별 전략  │ → │  크롤링 실행                  │ │ │
│  │  │ (IP 로테이션)  │   │ (SSR/iframe) │   │  (재시도/백오프)              │ │ │
│  │  └──────────────┘   └──────────────┘   └──────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                     3. 데이터 처리 레이어                                 │ │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────────────┐ │ │
│  │  │  LLM 파서     │ → │  정규화기     │ → │  이미지 생성                   │ │ │
│  │  │ (GPT/Claude)  │   │              │   │  (나노바나나 등 생성AI)        │ │ │
│  │  └──────────────┘   └──────────────┘   └──────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                        Ingestion API 호출 또는 직접 DB 접근
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           naecipe_backend                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                   app/ingestion/ 모듈                                    │ │
│  │  - 레시피 수신 및 검증                                                    │ │
│  │  - 최종 중복 검사 (Hash + Embedding 유사도)                               │ │
│  │  - 점수 계산 및 저장                                                     │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                              │                                                │
│                              ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                    PostgreSQL                                            │ │
│  │  - chefs, chef_platforms                                                 │ │
│  │  - recipes, recipe_ingredients, cooking_steps                            │ │
│  │  - tags, recipe_tags                                                     │ │
│  │  - crawl_logs (크롤링 이력 추적)                                          │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 크롤링 대상 인덱싱 전략

### 2.1 문제 정의

크롤링을 시작하기 전에 **"무엇을 크롤링할 것인가?"**를 먼저 정의해야 합니다.

#### 키워드 기반 접근의 한계
```
"제육볶음" 검색 → 수만 개의 결과
- 모든 레시피를 크롤링하는 것은 불가능
- 품질 불균일 (개인 블로그부터 전문 셰프까지)
- 중복 콘텐츠 범람
- 리소스 낭비 (LLM 파싱 비용, 스토리지)
```

#### 해결책: 요리사(Chef) 기준 인덱싱
DB 구조가 **요리사 중심**으로 설계되어 있으므로, 크롤링도 요리사 기준으로 접근합니다.

```
요리사 인덱스 구축 → 요리사별 콘텐츠 크롤링 → 레시피 저장
```

### 2.2 DB 구조 확인 (요리사 중심)

```sql
-- chefs 테이블: 요리사 정보
chefs (
    id UUID PRIMARY KEY,
    name VARCHAR(100),              -- "백종원", "승우아빠"
    name_normalized VARCHAR(100),   -- 검색용 정규화
    specialty VARCHAR(50),          -- 전문 분야 (한식, 양식)
    is_verified BOOLEAN,            -- 인증 여부
    recipe_count INTEGER,           -- 레시피 수
    ...
)

-- chef_platforms 테이블: 멀티 플랫폼 지원
chef_platforms (
    id UUID PRIMARY KEY,
    chef_id UUID FK,
    platform VARCHAR(30),           -- youtube, instagram, blog, naver
    platform_id VARCHAR(100),       -- 채널/계정 ID
    platform_url VARCHAR(500),      -- 채널 URL
    subscriber_count INTEGER,       -- 구독자 수
    ...
)

-- recipes 테이블: 요리사와 연결
recipes (
    id UUID PRIMARY KEY,
    chef_id UUID FK,                -- 요리사 참조
    title VARCHAR(200),
    source_url VARCHAR(500),
    source_platform VARCHAR(30),
    ...
)
```

### 2.3 크롤링 대상 인덱스 테이블 (신규)

```sql
-- 크롤링 대상 인덱스 테이블 (naecipe_backend에 추가)
CREATE TABLE crawl_targets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- 대상 정보
    target_type VARCHAR(20) NOT NULL,  -- chef, channel, keyword
    target_name VARCHAR(200) NOT NULL, -- "백종원", "요리왕비룡"
    target_name_normalized VARCHAR(200) NOT NULL,

    -- 플랫폼 정보
    platform VARCHAR(30) NOT NULL,     -- youtube, instagram, naver_blog, tiktok
    platform_id VARCHAR(100),          -- 채널/계정 ID
    platform_url VARCHAR(500),         -- 채널/프로필 URL

    -- 우선순위
    priority INTEGER DEFAULT 50,       -- 1(최고) ~ 100(최저)
    tier VARCHAR(10) NOT NULL,         -- S, A, B, C

    -- 메타데이터
    subscriber_count INTEGER,          -- 구독자/팔로워 수
    specialty VARCHAR(50),             -- 전문 분야
    description TEXT,                  -- 설명

    -- 상태
    status VARCHAR(20) DEFAULT 'active',  -- active, paused, completed, disabled
    last_crawled_at TIMESTAMP WITH TIME ZONE,
    crawl_frequency_hours INTEGER DEFAULT 24,  -- 크롤링 주기

    -- 연결
    chef_id UUID REFERENCES chefs(id), -- 매칭된 요리사 (있는 경우)

    -- 통계
    total_recipes_found INTEGER DEFAULT 0,
    total_recipes_saved INTEGER DEFAULT 0,
    success_rate FLOAT DEFAULT 0.0,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    UNIQUE(platform, platform_id)
);

CREATE INDEX idx_crawl_targets_tier ON crawl_targets(tier);
CREATE INDEX idx_crawl_targets_platform ON crawl_targets(platform);
CREATE INDEX idx_crawl_targets_status ON crawl_targets(status);
CREATE INDEX idx_crawl_targets_priority ON crawl_targets(priority);
```

### 2.4 요리사/인플루언서 티어 분류

#### 티어 기준
| 티어 | 구독자 수 | 특징 | 예시 | 크롤링 우선순위 |
|------|----------|------|------|----------------|
| **S** | 100만+ | 메가 인플루언서, TV 셰프 | 백종원, 정호영셰프 | 최우선 (매일) |
| **A** | 30만~100만 | 대형 인플루언서 | 승우아빠, 자취요리신 | 높음 (2일) |
| **B** | 10만~30만 | 중형 인플루언서 | 요리하는 직장인 | 중간 (주 2회) |
| **C** | 1만~10만 | 소형 인플루언서 | 인스타그램 요리 계정 | 낮음 (주 1회) |

#### 플랫폼별 티어 기준 (구독자 수 차이 반영)
```python
TIER_THRESHOLDS = {
    "youtube": {
        "S": 1_000_000,    # 100만
        "A": 300_000,      # 30만
        "B": 100_000,      # 10만
        "C": 10_000,       # 1만
    },
    "instagram": {
        "S": 500_000,      # 50만
        "A": 100_000,      # 10만
        "B": 30_000,       # 3만
        "C": 5_000,        # 5천
    },
    "tiktok": {
        "S": 1_000_000,    # 100만
        "A": 300_000,      # 30만
        "B": 50_000,       # 5만
        "C": 10_000,       # 1만
    },
    "naver_blog": {
        "S": 100_000,      # 10만 (방문자 수 기준)
        "A": 30_000,
        "B": 10_000,
        "C": 3_000,
    },
}
```

### 2.5 초기 크롤링 대상 시드 데이터

#### 플랫폼별 시드 요리사 목록

##### YouTube 요리 채널 (초기 시드)
```python
YOUTUBE_SEED_CHANNELS = [
    # S티어 - 메가 인플루언서
    {"name": "백종원", "channel_id": "UCyn-K7rZLXjGl7VXGweIlcA", "tier": "S"},
    {"name": "정호영셰프", "channel_id": "UCLPr2fR5Y2EaE6J_r_RtNHA", "tier": "S"},
    {"name": "요리왕비룡", "channel_id": "UCU2YvWH_6V9rN9IrqNdUGGw", "tier": "S"},

    # A티어 - 대형 인플루언서
    {"name": "승우아빠", "channel_id": "UC1H_wGj0iRTfg6SbvM5cWjQ", "tier": "A"},
    {"name": "자취요리신", "channel_id": "UCHJf5eLGOJa-NwYmgVwCveg", "tier": "A"},
    {"name": "요알남", "channel_id": "UCzR2h7gJ8eoT0vGQJCvpqdA", "tier": "A"},
    {"name": "딩가딩가스튜디오", "channel_id": "UCjgLRX2aWtk_cyIFjkPnvqQ", "tier": "A"},

    # B티어 - 중형 인플루언서
    {"name": "슈퍼키친", "channel_id": "...", "tier": "B"},
    {"name": "요리하는 직장인", "channel_id": "...", "tier": "B"},
    # ... 추가 채널
]
```

##### Instagram 요리 계정 (초기 시드)
```python
INSTAGRAM_SEED_ACCOUNTS = [
    {"name": "만개의레시피", "username": "10000recipe", "tier": "S"},
    {"name": "해먹남녀", "username": "haemuknamnyeo", "tier": "A"},
    # ... 추가 계정
]
```

##### 네이버 블로그 (초기 시드)
```python
NAVER_BLOG_SEED = [
    {"name": "만개의레시피", "blog_id": "10000recipe", "tier": "S"},
    {"name": "심방골주부", "blog_id": "simbg", "tier": "A"},
    # ... 추가 블로그
]
```

##### TikTok (초기 시드)
```python
TIKTOK_SEED_ACCOUNTS = [
    {"name": "요리하는여자", "username": "cooking_woman", "tier": "A"},
    # ... 추가 계정
]
```

### 2.6 크롤링 대상 발견 전략

#### 1단계: 시드 데이터 등록
```python
async def register_seed_targets():
    """초기 시드 데이터를 crawl_targets에 등록"""
    for channel in YOUTUBE_SEED_CHANNELS:
        await create_crawl_target(
            target_type="chef",
            target_name=channel["name"],
            platform="youtube",
            platform_id=channel["channel_id"],
            tier=channel["tier"]
        )
```

#### 2단계: 관련 채널 자동 발견
```python
async def discover_related_channels(seed_channel_id: str) -> list[dict]:
    """
    YouTube API를 통해 관련 채널 발견
    - 추천 채널
    - 협업 영상 출연자
    - 댓글에서 언급된 채널
    """
    # YouTube Data API - Related Channels
    related = await youtube_api.get_related_channels(seed_channel_id)

    # 요리 관련 채널 필터링
    cooking_channels = [
        ch for ch in related
        if is_cooking_channel(ch)  # 카테고리, 키워드 기반 판별
    ]

    return cooking_channels
```

#### 3단계: 신규 인플루언서 탐지
```python
async def discover_new_influencers():
    """
    새로운 요리 인플루언서 자동 탐지
    - 트렌딩 영상에서 새 채널 발견
    - 소셜 미디어 언급 모니터링
    - 레시피 사이트 인기 작성자
    """
    # YouTube Trending (요리 카테고리)
    trending = await youtube_api.get_trending(category="cooking")

    for video in trending:
        channel_id = video["channel_id"]
        if not await target_exists(channel_id):
            # 신규 채널 평가 후 등록
            await evaluate_and_register(channel_id)
```

### 2.7 크롤링 대상 관리 서비스

```python
class CrawlTargetManager:
    """크롤링 대상 인덱스 관리 서비스"""

    def __init__(self, db_session: AsyncSession):
        self.db = db_session

    async def get_targets_to_crawl(
        self,
        platform: str | None = None,
        tier: str | None = None,
        limit: int = 100
    ) -> list[CrawlTarget]:
        """
        크롤링할 대상 조회 (우선순위 순)
        - 마지막 크롤링 시간 + 주기 경과한 대상
        - 우선순위 높은 순
        """
        query = select(CrawlTarget).where(
            CrawlTarget.status == "active",
            CrawlTarget.last_crawled_at + timedelta(
                hours=CrawlTarget.crawl_frequency_hours
            ) < datetime.utcnow()
        )

        if platform:
            query = query.where(CrawlTarget.platform == platform)
        if tier:
            query = query.where(CrawlTarget.tier == tier)

        query = query.order_by(
            CrawlTarget.priority.asc(),
            CrawlTarget.last_crawled_at.asc()
        ).limit(limit)

        result = await self.db.execute(query)
        return result.scalars().all()

    async def register_target(
        self,
        target_name: str,
        platform: str,
        platform_id: str,
        platform_url: str | None = None,
        subscriber_count: int | None = None,
        specialty: str | None = None,
    ) -> CrawlTarget:
        """새 크롤링 대상 등록"""
        # 티어 자동 계산
        tier = self._calculate_tier(platform, subscriber_count)
        priority = self._calculate_priority(tier)
        crawl_frequency = self._calculate_frequency(tier)

        target = CrawlTarget(
            target_type="chef",
            target_name=target_name,
            target_name_normalized=self._normalize_name(target_name),
            platform=platform,
            platform_id=platform_id,
            platform_url=platform_url,
            subscriber_count=subscriber_count,
            specialty=specialty,
            tier=tier,
            priority=priority,
            crawl_frequency_hours=crawl_frequency,
        )

        self.db.add(target)
        await self.db.commit()
        return target

    async def link_to_chef(
        self,
        target_id: UUID,
        chef_id: UUID
    ) -> None:
        """크롤링 대상을 요리사 레코드와 연결"""
        target = await self.db.get(CrawlTarget, target_id)
        target.chef_id = chef_id
        await self.db.commit()

    async def update_stats(
        self,
        target_id: UUID,
        recipes_found: int,
        recipes_saved: int
    ) -> None:
        """크롤링 통계 업데이트"""
        target = await self.db.get(CrawlTarget, target_id)
        target.total_recipes_found += recipes_found
        target.total_recipes_saved += recipes_saved
        target.success_rate = (
            target.total_recipes_saved / target.total_recipes_found
            if target.total_recipes_found > 0 else 0.0
        )
        target.last_crawled_at = datetime.utcnow()
        await self.db.commit()

    def _calculate_tier(
        self,
        platform: str,
        subscriber_count: int | None
    ) -> str:
        """구독자 수 기반 티어 계산"""
        if subscriber_count is None:
            return "C"

        thresholds = TIER_THRESHOLDS.get(platform, TIER_THRESHOLDS["youtube"])

        if subscriber_count >= thresholds["S"]:
            return "S"
        elif subscriber_count >= thresholds["A"]:
            return "A"
        elif subscriber_count >= thresholds["B"]:
            return "B"
        else:
            return "C"

    def _calculate_priority(self, tier: str) -> int:
        """티어 기반 우선순위"""
        return {"S": 10, "A": 30, "B": 50, "C": 70}.get(tier, 50)

    def _calculate_frequency(self, tier: str) -> int:
        """티어 기반 크롤링 주기 (시간)"""
        return {"S": 24, "A": 48, "B": 84, "C": 168}.get(tier, 168)
```

### 2.8 크롤링 워크플로우 (인덱싱 포함)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       전체 크롤링 워크플로우                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 크롤링 대상 인덱싱 (신규)                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  시드 데이터 등록 → 관련 채널 발견 → 신규 인플루언서 탐지               │   │
│  │        ↓                ↓                  ↓                         │   │
│  │  crawl_targets 테이블에 저장 (티어, 우선순위, 크롤링 주기 설정)        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  2. 크롤링 대상 선택                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  get_targets_to_crawl() → 우선순위 + 마지막 크롤링 시간 기준 선택     │   │
│  │  - S티어: 매일                                                       │   │
│  │  - A티어: 2일마다                                                    │   │
│  │  - B티어: 주 2회                                                     │   │
│  │  - C티어: 주 1회                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  3. 콘텐츠 URL 수집 (대상별)                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  YouTube: 채널의 최신 영상 목록 조회                                   │   │
│  │  Instagram: 계정의 최신 게시물 조회                                   │   │
│  │  Blog: 블로그의 최신 포스트 조회                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  4. 사전 중복 검사 → 5. 크롤링 → 6. 파싱 → 7. 저장                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.9 키워드 기반 보조 인덱싱 (선택적)

요리사 기반 인덱싱으로 커버되지 않는 레시피를 위한 보조 전략:

```python
class KeywordCrawlTarget(BaseModel):
    """키워드 기반 크롤링 대상 (보조)"""
    keyword: str              # "제육볶음", "김치찌개"
    platforms: list[str]      # 검색할 플랫폼
    max_results: int          # 최대 수집 개수
    min_quality_score: float  # 최소 품질 점수 (조회수, 좋아요 기반)
    last_searched_at: datetime | None

    # 키워드 검색 시에도 품질 필터링
    # - 조회수 10,000 이상
    # - 좋아요 1,000 이상 또는 좋아요 비율 5% 이상
    # - 구독자 1,000 이상인 채널
```

```python
async def discover_recipes_by_keyword(
    keyword: str,
    platform: str,
    max_results: int = 50
) -> list[str]:
    """
    키워드로 레시피 검색 (품질 필터링 적용)
    - 이미 등록된 요리사의 콘텐츠 우선
    - 신규 채널은 품질 기준 충족 시에만 수집
    """
    results = await search_platform(platform, keyword, max_results * 3)

    filtered = []
    for item in results:
        # 이미 인덱싱된 요리사인지 확인
        if await is_indexed_chef(item.channel_id):
            filtered.append(item.url)
            continue

        # 품질 기준 체크
        if meets_quality_threshold(item):
            filtered.append(item.url)
            # 신규 채널 자동 등록 고려
            await maybe_register_new_target(item.channel_id)

    return filtered[:max_results]
```

---

## 3. 크롤링 인프라 전략

### 3.1 프록시 서비스 아키텍처

크롤링 시 IP 차단을 방지하기 위해 프록시 로테이션이 **필수**입니다.

#### 프록시 서비스 옵션
| 구분 | 서비스명 | 특징 | 가격대 |
|------|---------|------|--------|
| **무료** | Free-Proxy-List | 불안정, 속도 느림, 테스트용 | 무료 |
| **무료** | ProxyScrape | 주기적 갱신, 품질 불균일 | 무료 |
| **유료 (저가)** | Webshare | 10개 무료, 저가형 | $5/월 ~ |
| **유료 (중급)** | Bright Data | 주거용 IP 풀, 안정적 | $15/GB ~ |
| **유료 (고급)** | Oxylabs | 대용량, 고품질 | $15/GB ~ |
| **유료 (추천)** | ScraperAPI | 자동 로테이션, JS 렌더링 | $49/월 ~ |

#### 프록시 매니저 설계
```python
class ProxyManager:
    """프록시 풀 관리 및 자동 로테이션"""

    def __init__(self):
        self.proxy_pool: list[ProxyConfig] = []
        self.current_index: int = 0
        self.failed_proxies: set[str] = set()
        self.cooldown_map: dict[str, datetime] = {}

    async def get_proxy(self, domain: str) -> ProxyConfig:
        """도메인별 최적 프록시 반환"""
        pass

    async def mark_failed(self, proxy: ProxyConfig, reason: str):
        """실패한 프록시 마킹 및 쿨다운 설정"""
        pass

    async def rotate(self) -> ProxyConfig:
        """다음 프록시로 로테이션"""
        pass

    async def refresh_pool(self):
        """프록시 풀 갱신 (무료 프록시의 경우)"""
        pass


class ProxyConfig(BaseModel):
    """프록시 설정"""
    host: str
    port: int
    username: str | None = None
    password: str | None = None
    protocol: str = "http"  # http, https, socks5
    country: str | None = None
    is_residential: bool = False
    last_used: datetime | None = None
    fail_count: int = 0
```

### 3.2 크롤링 전략 분석

#### 크롤링 변수 및 대응 전략
| 변수 | 문제점 | 대응 전략 |
|------|--------|----------|
| **IP 차단** | 동일 IP 반복 요청 시 차단 | 프록시 로테이션, 요청 간격 조절 |
| **JavaScript 렌더링 (SSR)** | HTML만으로 콘텐츠 접근 불가 | Playwright/Selenium 사용 |
| **iframe 콘텐츠** | 메인 HTML에 콘텐츠 미포함 | iframe src 별도 요청, 동적 로드 대기 |
| **Captcha** | 봇 감지 시 Captcha 표시 | 2Captcha 서비스, 요청 패턴 자연화 |
| **Rate Limiting** | 과도한 요청 시 429 에러 | Exponential backoff, 요청 간격 준수 |
| **로그인 필요** | 인증 없이 접근 불가 | OAuth 토큰, 세션 쿠키 관리 |
| **동적 콘텐츠** | 스크롤/클릭으로 로드 | Playwright 스크롤 시뮬레이션 |
| **Anti-Bot 시스템** | Cloudflare, PerimeterX | undetected-chromedriver, 헤더 위장 |

#### 렌더링 전략 결정 트리
```
URL 분석
    │
    ├─ API 엔드포인트 존재? ──YES──→ API 직접 호출 (최우선)
    │
    ├─ 정적 HTML? ──YES──→ httpx 사용 (가장 빠름)
    │
    ├─ JavaScript 렌더링 필요? ──YES──→ Playwright 사용
    │
    └─ iframe 콘텐츠? ──YES──→ iframe src 추출 후 별도 요청
```

### 3.3 도메인별 크롤링 전략

#### 전략 아키텍처
```python
class DomainStrategy(ABC):
    """도메인별 크롤링 전략 인터페이스"""

    @property
    @abstractmethod
    def domain(self) -> str:
        """대상 도메인"""
        pass

    @property
    @abstractmethod
    def rendering_mode(self) -> RenderingMode:
        """렌더링 모드: STATIC, JAVASCRIPT, API"""
        pass

    @property
    @abstractmethod
    def rate_limit(self) -> RateLimit:
        """요청 제한 설정"""
        pass

    @abstractmethod
    async def extract_content(self, response: Response) -> CrawledContent:
        """콘텐츠 추출 로직"""
        pass

    @abstractmethod
    def get_headers(self) -> dict:
        """도메인별 커스텀 헤더"""
        pass


class RenderingMode(Enum):
    STATIC = "static"          # httpx로 충분
    JAVASCRIPT = "javascript"  # Playwright 필요
    API = "api"               # API 직접 호출


class RateLimit(BaseModel):
    requests_per_second: float = 1.0
    requests_per_minute: int = 30
    cooldown_on_429: int = 60  # 초
```

#### 플랫폼별 상세 전략

##### 1. YouTube
```python
class YouTubeStrategy(DomainStrategy):
    domain = "youtube.com"
    rendering_mode = RenderingMode.API  # YouTube Data API 우선
    rate_limit = RateLimit(
        requests_per_second=5.0,  # API 쿼터 고려
        requests_per_minute=100
    )

    특징:
    - YouTube Data API v3 사용 (가장 안정적)
    - API 쿼터: 10,000 units/day
    - 자막(caption) 별도 API 호출 필요
    - 채널 정보 별도 조회

    필요 데이터:
    - snippet: 제목, 설명, 썸네일, 채널 정보
    - contentDetails: 영상 길이
    - statistics: 조회수, 좋아요
    - captions: 자막 텍스트 (별도 API)

    주의사항:
    - API 쿼터 소진 시 대기 또는 여러 API 키 로테이션
    - 자막이 없는 영상은 설명(description)에서 레시피 추출
```

##### 2. 네이버 블로그
```python
class NaverBlogStrategy(DomainStrategy):
    domain = "blog.naver.com"
    rendering_mode = RenderingMode.JAVASCRIPT  # iframe 콘텐츠
    rate_limit = RateLimit(
        requests_per_second=0.5,  # 보수적 접근
        requests_per_minute=20
    )

    특징:
    - 블로그 본문이 iframe 내부에 존재
    - 네이버 검색 API로 URL 수집 가능
    - 모바일 버전(m.blog.naver.com)이 더 단순

    iframe 처리:
    1. 메인 페이지 로드
    2. iframe#mainFrame의 src 추출
    3. iframe 내부 콘텐츠 별도 요청
    또는
    1. Playwright로 전체 페이지 로드
    2. iframe 내용이 DOM에 병합될 때까지 대기

    주의사항:
    - 네이버 로그인 없이 일부 블로그 접근 제한
    - 이미지 지연 로딩(lazy loading) 처리 필요
    - SmartEditor 사용 블로그는 HTML 구조 복잡
```

##### 3. 만개의레시피
```python
class ManGaeStrategy(DomainStrategy):
    domain = "10000recipe.com"
    rendering_mode = RenderingMode.STATIC  # 정적 HTML
    rate_limit = RateLimit(
        requests_per_second=1.0,
        requests_per_minute=30
    )

    특징:
    - 구조화된 HTML (크롤링 친화적)
    - 레시피 데이터가 명확하게 마크업
    - 재료, 조리순서가 분리되어 있음

    데이터 추출 포인트:
    - 재료: div.ready_ingre3 > ul > li
    - 조리순서: div.view_step > div.view_step_cont
    - 작성자: div.user_info2

    주의사항:
    - robots.txt 준수 필요
    - 과도한 크롤링 시 IP 차단 가능
```

##### 4. 티스토리 (Tistory)
```python
class TistoryStrategy(DomainStrategy):
    domain = "tistory.com"
    rendering_mode = RenderingMode.STATIC  # 대부분 정적
    rate_limit = RateLimit(
        requests_per_second=1.0,
        requests_per_minute=30
    )

    특징:
    - 각 블로그마다 스킨이 다름 → HTML 구조 일관성 없음
    - 본문 영역: div.contents_style 또는 article
    - API 미제공

    대응 전략:
    - LLM 파싱 의존도 높음 (HTML 구조 무관하게 텍스트 추출)
    - 이미지 URL은 img 태그에서 직접 추출

    주의사항:
    - 성인 인증 필요 블로그 존재
    - 비공개 블로그 접근 불가
```

##### 5. Instagram
```python
class InstagramStrategy(DomainStrategy):
    domain = "instagram.com"
    rendering_mode = RenderingMode.API  # Graph API 또는 비공식
    rate_limit = RateLimit(
        requests_per_second=0.5,  # 매우 엄격
        requests_per_minute=15
    )

    특징:
    - 공식 Graph API 사용 권장 (비즈니스 계정 필요)
    - 웹 스크래핑 매우 어려움 (Anti-bot 강력)
    - 이미지/영상 중심 → 텍스트 정보 부족

    대응 전략:
    - Graph API: 인증된 비즈니스 계정으로 접근
    - 비공식: instaloader 라이브러리 (위험)
    - 캡션에서 레시피 정보 추출

    주의사항:
    - 무단 스크래핑 시 계정 영구 차단
    - API 사용량 제한 매우 엄격
    - 저작권 문제 가장 민감
```

#### 전략 레지스트리
```python
class StrategyRegistry:
    """도메인별 전략 레지스트리"""

    _strategies: dict[str, type[DomainStrategy]] = {
        "youtube.com": YouTubeStrategy,
        "blog.naver.com": NaverBlogStrategy,
        "m.blog.naver.com": NaverBlogStrategy,
        "10000recipe.com": ManGaeStrategy,
        "tistory.com": TistoryStrategy,
        "instagram.com": InstagramStrategy,
    }

    @classmethod
    def get_strategy(cls, url: str) -> DomainStrategy:
        """URL에서 도메인 추출 후 적절한 전략 반환"""
        domain = extract_domain(url)
        strategy_class = cls._strategies.get(domain)
        if not strategy_class:
            return DefaultStrategy()  # 기본 전략
        return strategy_class()
```

### 3.4 재시도 전략

```python
class RetryConfig(BaseModel):
    """재시도 설정"""
    max_retries: int = 3
    initial_delay: float = 1.0  # 초
    max_delay: float = 60.0  # 초
    exponential_base: float = 2.0
    jitter: bool = True  # 랜덤 지터 추가

    # 재시도할 HTTP 상태 코드
    retry_status_codes: list[int] = [429, 500, 502, 503, 504]

    # 재시도할 예외 타입
    retry_exceptions: list[type] = [
        ConnectionError,
        TimeoutError,
        ProxyError,
    ]


class RetryHandler:
    """Exponential Backoff 재시도 핸들러"""

    def __init__(self, config: RetryConfig):
        self.config = config

    async def execute_with_retry(
        self,
        func: Callable,
        *args,
        **kwargs
    ) -> Any:
        """재시도 로직으로 함수 실행"""
        last_exception = None

        for attempt in range(self.config.max_retries + 1):
            try:
                return await func(*args, **kwargs)
            except Exception as e:
                last_exception = e

                if not self._should_retry(e, attempt):
                    raise

                delay = self._calculate_delay(attempt)
                logger.warning(
                    f"재시도 {attempt + 1}/{self.config.max_retries}, "
                    f"대기 {delay:.2f}초: {e}"
                )
                await asyncio.sleep(delay)

        raise last_exception

    def _calculate_delay(self, attempt: int) -> float:
        """Exponential backoff 지연 시간 계산"""
        delay = self.config.initial_delay * (
            self.config.exponential_base ** attempt
        )
        delay = min(delay, self.config.max_delay)

        if self.config.jitter:
            delay *= (0.5 + random.random())  # 0.5~1.5x 랜덤

        return delay

    def _should_retry(self, exception: Exception, attempt: int) -> bool:
        """재시도 여부 결정"""
        if attempt >= self.config.max_retries:
            return False

        # HTTP 상태 코드 체크
        if isinstance(exception, HTTPStatusError):
            return exception.status_code in self.config.retry_status_codes

        # 예외 타입 체크
        return type(exception) in self.config.retry_exceptions
```

---

## 3. 사전 중복 검사 전략 (Pre-crawl Deduplication)

### 3.1 설계 원칙
- **크롤링 전에 중복 검사**: 리소스 낭비 방지 (네트워크, LLM 비용)
- **빠른 검사**: 해시/URL 기반 O(1) 조회
- **2단계 검증**: 사전 검사 → 크롤링 → 사후 검사

### 3.2 사전 중복 검사 서비스

```python
class PreCrawlDuplicateChecker:
    """크롤링 전 중복 검사 서비스"""

    def __init__(self, db_session: AsyncSession):
        self.db = db_session

    async def check_url_exists(self, url: str) -> DuplicateCheckResult:
        """URL 기반 중복 검사 (가장 빠름)"""
        normalized_url = self._normalize_url(url)

        # 1. source_url 정확 매칭
        existing = await self.db.execute(
            select(Recipe.id, Recipe.title, Recipe.updated_at)
            .where(Recipe.source_url == normalized_url)
        )
        result = existing.first()

        if result:
            return DuplicateCheckResult(
                is_duplicate=True,
                duplicate_type="exact_url",
                existing_recipe_id=result.id,
                existing_title=result.title,
                should_update=self._should_update(result.updated_at)
            )

        return DuplicateCheckResult(is_duplicate=False)

    async def check_url_hash(self, url: str) -> DuplicateCheckResult:
        """URL 해시 기반 검사"""
        url_hash = hashlib.sha256(url.encode()).hexdigest()

        existing = await self.db.execute(
            select(CrawlLog.recipe_id, CrawlLog.status)
            .where(CrawlLog.url_hash == url_hash)
        )
        return existing.first()

    async def check_title_similarity(
        self,
        title: str,
        author: str | None = None
    ) -> DuplicateCheckResult:
        """제목 유사도 검사 (옵션 - API 응답에 제목 포함 시)"""
        # 제목 정규화
        normalized_title = self._normalize_title(title)

        # 제목 해시로 빠른 검사
        title_hash = hashlib.md5(normalized_title.encode()).hexdigest()

        existing = await self.db.execute(
            select(Recipe.id, Recipe.title, Recipe.chef_id)
            .where(Recipe.title_hash == title_hash)
        )
        results = existing.all()

        if results:
            # 동일 작성자의 동일 제목 레시피
            if author:
                for r in results:
                    if await self._is_same_author(r.chef_id, author):
                        return DuplicateCheckResult(
                            is_duplicate=True,
                            duplicate_type="same_author_title",
                            existing_recipe_id=r.id
                        )

            return DuplicateCheckResult(
                is_duplicate=True,
                duplicate_type="same_title",
                existing_recipe_id=results[0].id,
                confidence=0.8  # 작성자 확인 못함
            )

        return DuplicateCheckResult(is_duplicate=False)

    def _normalize_url(self, url: str) -> str:
        """URL 정규화 (쿼리 파라미터 정렬, 프로토콜 통일 등)"""
        parsed = urlparse(url)
        # 불필요한 쿼리 파라미터 제거 (utm_ 등)
        query_params = parse_qs(parsed.query)
        filtered_params = {
            k: v for k, v in query_params.items()
            if not k.startswith('utm_')
        }
        normalized_query = urlencode(filtered_params, doseq=True)
        return urlunparse(parsed._replace(query=normalized_query))

    def _normalize_title(self, title: str) -> str:
        """제목 정규화"""
        # 공백 정규화, 특수문자 제거, 소문자화
        title = re.sub(r'\s+', ' ', title).strip().lower()
        title = re.sub(r'[^\w\s가-힣]', '', title)
        return title

    def _should_update(self, last_updated: datetime) -> bool:
        """업데이트 필요 여부 (30일 이상 경과)"""
        return (datetime.utcnow() - last_updated).days > 30


class DuplicateCheckResult(BaseModel):
    """중복 검사 결과"""
    is_duplicate: bool
    duplicate_type: str | None = None  # exact_url, same_title, same_author_title
    existing_recipe_id: UUID | None = None
    existing_title: str | None = None
    should_update: bool = False  # 업데이트 필요 여부
    confidence: float = 1.0
```

### 3.3 크롤링 로그 테이블 (추가 필요)

```sql
-- naecipe_backend에 추가할 테이블
CREATE TABLE crawl_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    url_hash VARCHAR(64) NOT NULL,  -- SHA256 해시
    source_url TEXT NOT NULL,
    platform VARCHAR(30) NOT NULL,
    status VARCHAR(20) NOT NULL,  -- pending, success, failed, duplicate, skipped
    recipe_id UUID REFERENCES recipes(id),
    error_message TEXT,
    retry_count INT DEFAULT 0,
    crawled_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    UNIQUE(url_hash)
);

CREATE INDEX idx_crawl_logs_url_hash ON crawl_logs(url_hash);
CREATE INDEX idx_crawl_logs_status ON crawl_logs(status);
CREATE INDEX idx_crawl_logs_platform ON crawl_logs(platform);
```

### 3.4 전체 파이프라인에서의 위치

```
1. URL 수집 (discover_urls)
        │
        ▼
2. ★ 사전 중복 검사 (Pre-crawl) ★
   ├─ URL 해시 검사 (CrawlLog 테이블)
   ├─ source_url 검사 (Recipe 테이블)
   └─ 중복이면 → SKIP (크롤링 안 함)
        │
        ▼ (중복 아닌 URL만)
3. 크롤링 실행
        │
        ▼
4. LLM 파싱
        │
        ▼
5. 정규화
        │
        ▼
6. ★ 사후 중복 검사 (Post-crawl) ★
   ├─ 제목+작성자 해시 검사
   └─ 임베딩 유사도 검사 (옵션)
        │
        ▼
7. DB 저장
```

---

## 4. 이미지 처리 전략 (저작권 준수)

### 4.1 문제점
- 원본 이미지 직접 저장 시 **저작권 침해** 위험
- 핫링크(hotlink) 시 원본 서버 부하 및 이미지 삭제 위험
- 법적 리스크 최소화 필요

### 4.2 해결 전략: 생성형 AI 이미지

#### 생성형 이미지 서비스 옵션
| 서비스 | 특징 | 가격 | 적합도 |
|--------|------|------|--------|
| **나노바나나 (추천)** | 한국 음식 특화, 자연스러운 결과 | 유료 | ★★★★★ |
| **DALL-E 3** | 범용 고품질, API 지원 | $0.04/이미지 | ★★★★☆ |
| **Midjourney** | 고품질, API 미지원 | $10/월~ | ★★★☆☆ |
| **Stable Diffusion** | 자체 호스팅 가능, 무료 | 인프라 비용 | ★★★☆☆ |
| **Ideogram** | 텍스트 렌더링 우수 | $7/월~ | ★★★☆☆ |

#### 이미지 생성 파이프라인

```python
class ImageGenerationService:
    """레시피 이미지 생성 서비스"""

    def __init__(self, provider: ImageProvider = "nanobanana"):
        self.provider = self._get_provider(provider)

    async def generate_recipe_image(
        self,
        recipe: NormalizedRecipe,
        original_image_url: str | None = None
    ) -> GeneratedImage:
        """레시피 정보 기반 이미지 생성"""

        # 1. 프롬프트 생성
        prompt = self._build_prompt(recipe, original_image_url)

        # 2. 이미지 생성
        generated = await self.provider.generate(prompt)

        # 3. 품질 검증
        if not self._validate_quality(generated):
            # 재생성 또는 대체 이미지
            generated = await self._fallback_generation(recipe)

        # 4. 저장소 업로드
        image_url = await self._upload_to_storage(generated)

        return GeneratedImage(
            url=image_url,
            prompt_used=prompt,
            provider=self.provider.name,
            generated_at=datetime.utcnow()
        )

    def _build_prompt(
        self,
        recipe: NormalizedRecipe,
        reference_url: str | None
    ) -> str:
        """레시피 정보로 이미지 생성 프롬프트 구성"""

        # 기본 구성요소
        dish_name = recipe.title
        main_ingredients = [i.name for i in recipe.ingredients[:5]]
        cuisine_type = self._detect_cuisine(recipe.tags)

        prompt = f"""
        음식 사진, {dish_name},
        주요 재료: {', '.join(main_ingredients)},
        스타일: {cuisine_type} 요리,
        조명: 자연광,
        구도: 위에서 45도 각도,
        배경: 깔끔한 테이블,
        품질: 고해상도, 음식 전문 사진
        """

        # 참조 이미지가 있으면 스타일 힌트 추가
        if reference_url:
            prompt += ", 스타일 참조: 실제 요리 완성 사진"

        return prompt.strip()

    async def generate_step_images(
        self,
        recipe: NormalizedRecipe,
        steps: list[StepInfo]
    ) -> list[GeneratedImage]:
        """조리 단계별 이미지 생성 (옵션)"""
        step_images = []

        for step in steps:
            if step.image_url:  # 원본에 이미지가 있는 단계만
                prompt = self._build_step_prompt(recipe, step)
                image = await self.provider.generate(prompt)
                step_images.append(image)

        return step_images


class ImageProvider(ABC):
    """이미지 생성 프로바이더 인터페이스"""

    @abstractmethod
    async def generate(self, prompt: str) -> bytes:
        pass

    @property
    @abstractmethod
    def name(self) -> str:
        pass


class NanoBananaProvider(ImageProvider):
    """나노바나나 이미지 생성"""
    name = "nanobanana"

    async def generate(self, prompt: str) -> bytes:
        # 나노바나나 API 호출
        pass


class DallE3Provider(ImageProvider):
    """DALL-E 3 이미지 생성"""
    name = "dalle3"

    async def generate(self, prompt: str) -> bytes:
        response = await openai.images.generate(
            model="dall-e-3",
            prompt=prompt,
            size="1024x1024",
            quality="standard",
            n=1,
        )
        return response.data[0].url
```

### 4.3 이미지 처리 정책

```python
class ImagePolicy:
    """이미지 처리 정책"""

    # 썸네일: 항상 생성형 AI로 생성
    THUMBNAIL_POLICY = "generate"

    # 조리 단계 이미지: 선택적 생성
    STEP_IMAGE_POLICY = "optional_generate"

    # 원본 이미지 URL: 메타데이터로만 보관 (표시 안 함)
    ORIGINAL_URL_POLICY = "metadata_only"

    # 이미지 생성 실패 시: 기본 이미지 사용
    FALLBACK_POLICY = "default_placeholder"
```

---

## 5. 타겟 DB 스키마 (naecipe_backend)

### 5.1 테이블 구조

#### chefs (요리사)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK |
| name | VARCHAR(100) | 요리사 이름 (예: 백종원) |
| name_normalized | VARCHAR(100) | 검색용 정규화 이름 |
| profile_image_url | VARCHAR(500) | 프로필 이미지 |
| bio | TEXT | 소개 |
| specialty | VARCHAR(50) | 전문 분야 (한식, 양식 등) |
| recipe_count | INTEGER | 레시피 수 (캐싱) |
| total_views | INTEGER | 총 조회수 |
| avg_rating | FLOAT | 평균 평점 |
| is_verified | BOOLEAN | 인증 여부 |
| created_at, updated_at | TIMESTAMP | 생성/수정 시각 |

#### chef_platforms (요리사 플랫폼)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK |
| chef_id | UUID | FK → chefs.id |
| platform | VARCHAR(30) | 플랫폼 (youtube, instagram, blog, naver) |
| platform_id | VARCHAR(100) | 플랫폼 내 채널/계정 ID |
| platform_url | VARCHAR(500) | 플랫폼 URL |
| subscriber_count | INTEGER | 구독자 수 |
| last_synced_at | TIMESTAMP | 마지막 동기화 시각 |

#### recipes (레시피)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK |
| chef_id | UUID | FK → chefs.id (nullable) |
| title | VARCHAR(200) | 레시피 제목 |
| title_hash | VARCHAR(32) | 제목 해시 (중복 검사용) |
| description | TEXT | 레시피 설명 |
| thumbnail_url | VARCHAR(500) | **생성된** 썸네일 이미지 URL |
| original_thumbnail_url | VARCHAR(500) | 원본 썸네일 (메타데이터) |
| video_url | VARCHAR(500) | 영상 URL |
| prep_time_minutes | INTEGER | 준비 시간 (분) |
| cook_time_minutes | INTEGER | 조리 시간 (분) |
| servings | INTEGER | 인분 |
| difficulty | VARCHAR(20) | 난이도 (easy, medium, hard) |
| source_url | VARCHAR(500) | 원본 소스 URL |
| source_platform | VARCHAR(30) | 출처 플랫폼 |
| exposure_score | FLOAT | 노출 점수 |
| view_count | INTEGER | 조회수 |
| is_active | BOOLEAN | 활성 상태 |
| created_at, updated_at | TIMESTAMP | 생성/수정 시각 |

#### recipe_ingredients (레시피 재료)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK |
| recipe_id | UUID | FK → recipes.id |
| name | VARCHAR(100) | 재료명 |
| amount | VARCHAR(50) | 양 (예: 300, 1/2, 약간) |
| unit | VARCHAR(30) | 단위 (g, ml, 개, 큰술) |
| note | TEXT | 부가 설명 |
| order_index | INTEGER | 표시 순서 |

#### cooking_steps (조리 단계)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK |
| recipe_id | UUID | FK → recipes.id |
| step_number | INTEGER | 단계 번호 (1부터) |
| description | TEXT | 단계 설명 |
| image_url | VARCHAR(500) | **생성된** 단계별 이미지 |
| original_image_url | VARCHAR(500) | 원본 이미지 (메타데이터) |
| duration_seconds | INTEGER | 단계 소요 시간 (초) |
| tip | TEXT | 조리 팁 |

#### tags (태그)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK |
| name | VARCHAR(50) | 태그명 (unique) |
| category | VARCHAR(30) | 카테고리 (dish_type, cuisine, meal_type, cooking_method, dietary) |
| display_order | INTEGER | 표시 순서 |

#### recipe_tags (레시피-태그 연결)
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK |
| recipe_id | UUID | FK → recipes.id |
| tag_id | UUID | FK → tags.id |

#### crawl_logs (크롤링 로그) - **신규**
| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID | PK |
| url_hash | VARCHAR(64) | URL SHA256 해시 (unique) |
| source_url | TEXT | 원본 URL |
| platform | VARCHAR(30) | 플랫폼 |
| status | VARCHAR(20) | 상태 (pending, success, failed, duplicate, skipped) |
| recipe_id | UUID | 성공 시 생성된 레시피 ID |
| error_message | TEXT | 실패 시 에러 메시지 |
| retry_count | INTEGER | 재시도 횟수 |
| crawled_at | TIMESTAMP | 크롤링 완료 시각 |
| created_at | TIMESTAMP | 레코드 생성 시각 |

### 5.2 태그 카테고리
- `dish_type`: 요리 종류 (찌개, 볶음, 구이, 튀김)
- `cuisine`: 요리 스타일 (한식, 양식, 중식, 일식)
- `meal_type`: 식사 유형 (아침, 점심, 저녁, 간식)
- `cooking_method`: 조리 방법 (에어프라이어, 오븐, 전자레인지)
- `dietary`: 식이 제한 (비건, 저탄수화물, 글루텐프리)

---

## 6. 크롤링 데이터 스키마

### 6.1 크롤링 원본 데이터 (CrawledContent)
```python
class CrawledContent(BaseModel):
    """크롤링된 원본 콘텐츠"""
    url: str                          # 원본 URL
    url_hash: str                     # URL SHA256 해시
    platform: str                     # youtube, instagram, naver_blog, etc.
    raw_content: str                  # 원본 텍스트/HTML
    raw_metadata: dict                # 플랫폼별 메타데이터
    crawled_at: datetime              # 크롤링 시각

    # 프록시 정보
    proxy_used: str | None            # 사용된 프록시
    response_time_ms: int             # 응답 시간

    # 플랫폼별 추가 정보
    author_name: str | None           # 작성자 이름
    author_channel_id: str | None     # 채널/계정 ID
    author_channel_url: str | None    # 채널/계정 URL
    thumbnail_url: str | None         # 원본 썸네일 (메타데이터용)
    video_url: str | None             # 영상 URL (YouTube 등)
    published_at: datetime | None     # 원본 게시 시각
    view_count: int | None            # 조회수
    like_count: int | None            # 좋아요 수
    subscriber_count: int | None      # 채널 구독자 수
```

### 6.2 LLM 파싱 결과 (ParsedRecipe)
```python
class ParsedRecipe(BaseModel):
    """LLM으로 파싱된 레시피"""
    title: str                        # 레시피 제목
    description: str | None           # 레시피 설명

    # 재료 목록
    ingredients: list[ParsedIngredient]

    # 조리 단계
    steps: list[ParsedStep]

    # 조리 정보
    prep_time_minutes: int | None
    cook_time_minutes: int | None
    servings: int | None
    difficulty: str | None            # easy, medium, hard

    # 태그 (LLM 추출)
    tags: list[str]                   # 태그명 리스트


class ParsedIngredient(BaseModel):
    """파싱된 재료"""
    name: str                         # 재료명
    amount: str | None                # 양
    unit: str | None                  # 단위
    note: str | None                  # 부가 설명
    order_index: int                  # 순서


class ParsedStep(BaseModel):
    """파싱된 조리 단계"""
    step_number: int                  # 단계 번호
    description: str                  # 설명
    original_image_url: str | None    # 원본 이미지 (메타데이터)
    duration_seconds: int | None      # 소요 시간
    tip: str | None                   # 팁
```

### 6.3 정규화된 레시피 (NormalizedRecipe)
```python
class NormalizedRecipe(BaseModel):
    """DB 저장용 정규화된 레시피"""
    # 레시피 기본 정보
    title: str
    title_hash: str                   # 중복 검사용 해시
    description: str | None
    thumbnail_url: str | None         # 생성된 이미지 URL
    original_thumbnail_url: str | None  # 원본 이미지 (메타데이터)
    video_url: str | None

    # 조리 정보
    prep_time_minutes: int | None
    cook_time_minutes: int | None
    servings: int | None
    difficulty: str | None

    # 출처 정보
    source_url: str
    source_url_hash: str              # URL 해시
    source_platform: str

    # 요리사 정보
    chef: ChefInfo | None

    # 재료 및 조리 단계
    ingredients: list[IngredientInfo]
    steps: list[StepInfo]

    # 태그
    tags: list[str]

    # 메타데이터
    crawled_at: datetime
    published_at: datetime | None
    original_view_count: int | None
    original_like_count: int | None


class ChefInfo(BaseModel):
    """요리사 정보"""
    name: str
    name_normalized: str              # 정규화된 이름 (공백제거, 소문자)
    profile_image_url: str | None
    platform: str
    platform_id: str | None
    platform_url: str | None
    subscriber_count: int | None


class IngredientInfo(BaseModel):
    """재료 정보"""
    name: str
    amount: str | None
    unit: str | None
    note: str | None
    order_index: int


class StepInfo(BaseModel):
    """조리 단계 정보"""
    step_number: int
    description: str
    image_url: str | None             # 생성된 이미지 URL
    original_image_url: str | None    # 원본 이미지 (메타데이터)
    duration_seconds: int | None
    tip: str | None
```

---

## 7. 구현 단계별 스펙

> **스펙 번호 체계**: 우선순위 순서대로 SPEC-001부터 순차 번호 부여
> - Phase 1 (MVP 인프라): SPEC-001 ~ SPEC-005
> - Phase 2 (크롤링 코어): SPEC-006 ~ SPEC-013
> - Phase 3 (이미지 생성): SPEC-014 ~ SPEC-015
> - Phase 4 (플랫폼 확장): SPEC-016 ~ SPEC-021
> - Phase 5 (운영): SPEC-022 ~ SPEC-027
> - Phase 6+ (후순위): SPEC-028 ~

---

### Phase 1: MVP 인프라 (1주)

#### SPEC-001: 크롤링 대상 인덱스 시스템 구현 ⭐ 최우선
- **목표**: 요리사/인플루언서 기반 크롤링 대상 관리
- **산출물**: `indexing/crawl_target_manager.py`, `indexing/models.py`
- **우선순위**: P0 (MVP 필수)
- **작업 내용**:
  - [ ] CrawlTarget 데이터 모델 정의
  - [ ] CrawlTargetManager 클래스 구현
  - [ ] 티어 자동 계산 로직 (구독자 수 기반)
  - [ ] 우선순위 및 크롤링 주기 계산
  - [ ] 크롤링 대상 등록/조회/업데이트 API
  - [ ] **시드 데이터 로더 구현** (수동 큐레이션 중심)
  - [ ] crawl_targets 테이블 마이그레이션

#### SPEC-002: 프록시 매니저 구현 ⭐ 최우선
- **목표**: IP 로테이션을 위한 프록시 풀 관리
- **산출물**: `infra/proxy_manager.py`
- **우선순위**: P0 (MVP 필수)
- **작업 내용**:
  - [ ] ProxyConfig, ProxyManager 클래스 구현
  - [ ] **ScraperAPI 단일 지원** (MVP 단순화)
  - [ ] 프록시 상태 모니터링 (실패율, 응답시간)
  - [ ] 자동 로테이션 로직
  - [ ] 프록시 없이 동작 모드 (YouTube API, 공식 API용)
- **비용 분석**:
  ```
  ScraperAPI: $49/월 (100K 요청) = $0.0005/요청
  - 만개의레시피, 티스토리, 네이버블로그에 적용
  - YouTube API는 프록시 불필요
  ```

#### SPEC-003: 도메인별 크롤링 전략 구현 (YouTube + 만개의레시피) ⭐ 최우선
- **목표**: MVP 대상 플랫폼의 맞춤 크롤링 전략
- **산출물**: `strategies/base.py`, `strategies/youtube.py`, `strategies/mangae.py`
- **우선순위**: P0 (MVP 필수)
- **작업 내용**:
  - [ ] DomainStrategy 추상 클래스 정의
  - [ ] StrategyRegistry 구현
  - [ ] **YouTubeStrategy 구현** (API 우선, 프록시 불필요)
  - [ ] **ManGaeStrategy 구현** (정적 HTML, 프록시 선택적)
- **참고**: 네이버블로그, 티스토리, Instagram은 Phase 4로 이동

#### SPEC-004: 재시도 핸들러 구현
- **목표**: Exponential backoff 기반 재시도 로직
- **산출물**: `infra/retry_handler.py`
- **우선순위**: P0 (MVP 필수)
- **작업 내용**:
  - [ ] RetryConfig 설정 클래스
  - [ ] RetryHandler 구현
  - [ ] HTTP 상태 코드별 재시도 정책 (429, 500, 502, 503, 504)
  - [ ] 예외 타입별 재시도 정책
  - [ ] Jitter 기반 지연 시간 계산

#### SPEC-005: 사전 중복 검사 서비스 구현 ⭐ 최우선
- **목표**: 크롤링 전 DB 조회로 중복 필터링 (비용 절감 핵심)
- **산출물**: `dedup/pre_crawl_checker.py`
- **우선순위**: P0 (MVP 필수)
- **작업 내용**:
  - [ ] PreCrawlDuplicateChecker 구현
  - [ ] URL 해시 기반 검사 (SHA256)
  - [ ] source_url 정확 매칭
  - [ ] CrawlLog 테이블 연동
- **효과**: LLM 파싱 비용 ~50% 절감 예상

---

### Phase 2: 크롤링 코어 (1.5주)

#### SPEC-006: 크롤링 데이터 스키마 정의
- **목표**: 크롤링 원본/파싱/정규화 데이터 모델 정의
- **산출물**: `schemas/crawling.py`
- **우선순위**: P1
- **작업 내용**:
  - [ ] CrawledContent 모델 정의 (프록시 정보 포함)
  - [ ] ParsedRecipe, ParsedIngredient, ParsedStep 모델 정의
  - [ ] NormalizedRecipe, ChefInfo, IngredientInfo, StepInfo 모델 정의
  - [ ] 각 모델 검증 로직 추가
  - [ ] 해시 필드 추가 (url_hash, title_hash)

#### SPEC-007: 플랫폼별 크롤러 인터페이스 정의
- **목표**: 각 플랫폼 크롤러의 표준 인터페이스 정의
- **산출물**: `crawlers/base.py`
- **우선순위**: P1
- **작업 내용**:
  - [ ] BaseCrawler 추상 클래스 정의
  - [ ] 공통 메서드: `crawl(url)`, `crawl_batch(urls)`, `discover_urls()`
  - [ ] 프록시 매니저 연동
  - [ ] 재시도 핸들러 연동
  - [ ] 도메인 전략 연동

#### SPEC-008: YouTube 크롤러 구현 ⭐ 핵심
- **목표**: YouTube 영상에서 레시피 정보 추출
- **산출물**: `crawlers/youtube.py`
- **우선순위**: P1
- **작업 내용**:
  - [ ] YouTube Data API v3 연동
  - [ ] 영상 메타데이터 추출 (제목, 설명, 썸네일, 채널 정보)
  - [ ] 자막(caption) 추출 (가능한 경우)
  - [ ] 레시피 채널 목록 관리 및 새 영상 발견
  - [ ] API 쿼터 관리 (10,000 units/day)
  - [ ] 사전 중복 검사 연동 (SPEC-005)
- **참고**: 프록시 불필요 (공식 API 사용)

#### SPEC-009: 만개의레시피 크롤러 구현 ⭐ 핵심
- **목표**: 만개의레시피에서 레시피 정보 추출
- **산출물**: `crawlers/mangae.py`
- **우선순위**: P1
- **작업 내용**:
  - [ ] HTML 구조 분석 및 파서 구현
  - [ ] 구조화된 데이터 추출 (재료, 조리 단계)
  - [ ] 이미지 URL 추출
  - [ ] robots.txt 준수
  - [ ] Rate limiting 준수 (1 req/sec)
  - [ ] **프록시 선택적 적용** (차단 시에만)

#### SPEC-010: GPT 파서 구현 ⭐ 핵심
- **목표**: GPT-4 Turbo를 사용한 레시피 파싱
- **산출물**: `parsers/openai_parser.py`
- **우선순위**: P1
- **작업 내용**:
  - [ ] OpenAI API 연동
  - [ ] 레시피 추출 프롬프트 설계
  - [ ] 구조화된 출력 파싱 (JSON mode)
  - [ ] 토큰 사용량 최적화
  - [ ] **비용 추적 통합** (SPEC-013)
- **예상 비용**: ~$0.02/레시피 (GPT-4 Turbo 기준)

#### SPEC-011: 레시피 메타데이터 정규화
- **목표**: 레시피 전체 메타데이터 정규화
- **산출물**: `normalizers/recipe_normalizer.py`
- **우선순위**: P1
- **작업 내용**:
  - [ ] 제목 정규화 (특수문자, 이모지 처리)
  - [ ] **제목 해시 생성** (중복 검사용)
  - [ ] 난이도 추론 (조리시간, 단계 수 기반)
  - [ ] 조리시간 추론 (단계별 시간 합산)
  - [ ] 인분 기본값 설정 (2인분)

#### SPEC-012: 해시 기반 중복 검사 (사후)
- **목표**: 빠른 중복 검사를 위한 해시 매칭
- **산출물**: `dedup/hash_checker.py`
- **우선순위**: P1
- **작업 내용**:
  - [ ] content_hash 생성 로직 (제목 + 작성자)
  - [ ] source_url 기반 정확 매칭
  - [ ] 해시 인덱스 관리

#### SPEC-013: 비용 추적 시스템 구현 ⭐ 신규
- **목표**: 크롤링 비용 실시간 추적 및 예산 관리
- **산출물**: `monitoring/cost_tracker.py`
- **우선순위**: P1 (운영 필수)
- **작업 내용**:
  - [ ] CostTracker 클래스 구현
  - [ ] LLM 토큰 사용량 기록 (GPT-4, Claude)
  - [ ] 이미지 생성 비용 기록
  - [ ] 프록시 비용 기록
  - [ ] 레시피당 평균 비용 계산
  - [ ] 일일 예산 초과 알림
- **예상 비용 (레시피당)**:
  ```
  LLM 파싱: ~$0.02
  이미지 생성: ~$0.04
  프록시: ~$0.001
  총: ~$0.06/레시피
  ```

---

### Phase 3: 이미지 생성 (0.5주)

#### SPEC-014: 이미지 생성 서비스 구현 ⭐ 핵심
- **목표**: 생성형 AI를 통한 레시피 썸네일 이미지 생성
- **산출물**: `image/generation_service.py`
- **우선순위**: P2
- **작업 내용**:
  - [ ] ImageGenerationService 클래스 구현
  - [ ] **DALL-E 3 프로바이더 구현** (MVP 우선 - 검증된 API)
  - [ ] 레시피 정보 기반 프롬프트 생성
  - [ ] 이미지 품질 검증
  - [ ] 실패 시 기본 이미지 폴백
- **참고**: 나노바나나는 API 검증 후 Phase 4에서 전환 검토
- **범위 제한**: 썸네일 1장만 생성 (조리 단계별 이미지는 제외)

#### SPEC-015: 이미지 저장소 연동
- **목표**: 생성된 이미지를 스토리지에 저장
- **산출물**: `image/storage_service.py`
- **우선순위**: P2
- **작업 내용**:
  - [ ] S3 업로드 서비스 (MVP)
  - [ ] 이미지 리사이징 (썸네일 512x512)
  - [ ] CDN URL 생성
- **참고**: GCS, 이미지 캐싱은 Phase 5로 연기

---

### Phase 4: 플랫폼 확장 (2주)

#### SPEC-016: 네이버 블로그 크롤러 구현
- **목표**: 네이버 블로그에서 레시피 정보 추출
- **산출물**: `crawlers/naver_blog.py`
- **우선순위**: P3
- **작업 내용**:
  - [ ] 네이버 검색 API 연동 (URL 수집)
  - [ ] **iframe 콘텐츠 처리** (Playwright)
  - [ ] 블로그 본문 HTML 파싱
  - [ ] 이미지 URL 추출 (lazy loading 처리)
  - [ ] 작성자 정보 추출
  - [ ] 프록시 로테이션 적용

#### SPEC-017: 티스토리 크롤러 구현
- **목표**: 티스토리 블로그에서 레시피 정보 추출
- **산출물**: `crawlers/tistory.py`
- **우선순위**: P3
- **작업 내용**:
  - [ ] 다양한 스킨 대응 (LLM 의존)
  - [ ] 본문 텍스트 추출
  - [ ] 이미지 URL 추출
  - [ ] 성인 인증/비공개 블로그 예외 처리

#### SPEC-018: 품질 점수 시스템 구현 ⭐ 신규
- **목표**: 크롤링 결과 품질 평가 및 저장 여부 결정
- **산출물**: `scoring/quality_scorer.py`
- **우선순위**: P3
- **작업 내용**:
  - [ ] CrawlQualityScore 클래스 구현
  - [ ] 재료 수 평가 (최소 3개: +0.25)
  - [ ] 조리 단계 수 평가 (최소 3개: +0.25)
  - [ ] 조리 시간 정보 존재 (+0.15)
  - [ ] 설명 길이 평가 (50자 이상: +0.15)
  - [ ] 이미지 존재 (+0.20)
  - [ ] 저장 임계값 설정 (0.5 이상만 저장)

#### SPEC-019: 태그 추출 및 정규화
- **목표**: LLM을 통한 태그 자동 추출 및 정규화
- **산출물**: `parsers/tag_extractor.py`
- **우선순위**: P3
- **작업 내용**:
  - [ ] 태그 카테고리별 추출 프롬프트 설계
  - [ ] 기존 태그와의 매칭 로직
  - [ ] 새 태그 생성 기준 정의
  - [ ] 태그 정규화 (동의어 처리)

#### SPEC-020: 요리사(Chef) 정규화
- **목표**: 크롤링된 작성자 정보를 Chef 엔티티로 정규화
- **산출물**: `normalizers/chef_normalizer.py`
- **우선순위**: P3
- **작업 내용**:
  - [ ] 요리사 이름 정규화 로직
  - [ ] 동일 요리사 다중 플랫폼 매칭
  - [ ] 새 요리사 생성 vs 기존 요리사 연결 판단
  - [ ] 요리사 통계 업데이트 로직

#### SPEC-021: 재료/단계 정규화
- **목표**: 파싱된 재료 및 조리 단계 정규화
- **산출물**: `normalizers/ingredient_normalizer.py`, `normalizers/step_normalizer.py`
- **우선순위**: P3
- **작업 내용**:
  - [ ] 재료명 정규화 (띄어쓰기, 오타 보정)
  - [ ] 양(amount) 파싱 (분수 → 소수, 범위 처리)
  - [ ] 단위(unit) 표준화
  - [ ] 단계 번호 재정렬
  - [ ] 단계 설명 정제

---

### Phase 5: 운영 (1주)

#### SPEC-022: 레시피(Recipe) 저장 서비스
- **목표**: 레시피 데이터 저장
- **산출물**: `ingestion/recipe_service.py`
- **우선순위**: P4
- **작업 내용**:
  - [ ] 레시피 기본 정보 저장
  - [ ] 생성된 이미지 URL 저장
  - [ ] 재료 목록 저장
  - [ ] 조리 단계 저장
  - [ ] 태그 연결 (기존 태그 매칭 또는 새 태그 생성)
  - [ ] 트랜잭션 관리

#### SPEC-023: 크롤링 로그 저장 서비스
- **목표**: 크롤링 이력 추적
- **산출물**: `ingestion/crawl_log_service.py`
- **우선순위**: P4
- **작업 내용**:
  - [ ] CrawlLog 레코드 생성/업데이트
  - [ ] 성공/실패/중복 상태 기록
  - [ ] 재시도 횟수 추적
  - [ ] 에러 메시지 저장

#### SPEC-024: 크롤링 파이프라인 구축
- **목표**: 전체 크롤링 → 저장 파이프라인 통합
- **산출물**: `pipeline/crawling_pipeline.py`
- **우선순위**: P4
- **작업 내용**:
  - [ ] 파이프라인 단계 정의
  ```
  URL 수집 → 사전 중복검사 → 크롤링 → LLM 파싱 → 정규화 →
  품질 평가 → 이미지 생성 → 해시 중복검사 → DB 저장 → 로그 기록
  ```
  - [ ] 단계별 에러 핸들링
  - [ ] 파이프라인 상태 추적
  - [ ] 재개 가능한 체크포인트

#### SPEC-025: 스케줄러 구현
- **목표**: 주기적 크롤링 작업 스케줄링
- **산출물**: `scheduler/crawl_scheduler.py`
- **우선순위**: P4
- **작업 내용**:
  - [ ] 플랫폼별 크롤링 주기 설정
  - [ ] 새 콘텐츠 발견 스케줄
  - [ ] 동시 실행 제한
  - [ ] 티어별 크롤링 주기 (S: 매일, A: 2일, B: 주2회, C: 주1회)

#### SPEC-026: CLI 인터페이스
- **목표**: 크롤링 작업 CLI 명령어
- **산출물**: `cli/commands.py`
- **우선순위**: P4
- **작업 내용**:
  - [ ] `crawl` - 단일 URL 크롤링
  - [ ] `crawl-batch` - URL 목록 크롤링
  - [ ] `discover` - 새 레시피 발견
  - [ ] `check-dup` - 중복 검사
  - [ ] `cost-report` - 비용 리포트

#### SPEC-027: 모니터링 및 알림
- **목표**: 크롤링 현황 모니터링 및 에러 알림
- **산출물**: `monitoring/stats.py`, `monitoring/alerts.py`
- **우선순위**: P4
- **작업 내용**:
  - [ ] 플랫폼별 크롤링 현황
  - [ ] 성공/실패 비율
  - [ ] 중복 발견 비율
  - [ ] LLM 토큰 사용량
  - [ ] 프록시 성공률
  - [ ] IP 차단 감지 및 알림
  - [ ] 일일 비용 리포트

---

### Phase 6+: 후순위 (MVP 완료 후 검토)

#### SPEC-028: 노출 점수(exposure_score) 계산 🔄 연기
- **목표**: 레시피 노출 순위를 위한 점수 계산
- **산출물**: `scoring/exposure_scorer.py`
- **상태**: MVP 후 검토
- **작업 내용**:
  - [ ] 인기도 점수 (40%): 조회수, 좋아요, 구독자 수 기반
  - [ ] 품질 점수 (30%): 재료/단계 완성도, 이미지 유무
  - [ ] 신선도 점수 (20%): 게시일 기반 감소 함수
  - [ ] 출처 신뢰도 (10%): 플랫폼/채널별 가중치

#### SPEC-029: 임베딩 기반 유사도 검사 🔄 연기
- **목표**: 의미적 유사 레시피 탐지
- **산출물**: `dedup/embedding_checker.py`
- **상태**: MVP 후 검토 (해시 기반으로 충분한지 평가 후)
- **작업 내용**:
  - [ ] 레시피 임베딩 생성 (제목 + 설명 + 재료)
  - [ ] pgvector를 통한 유사도 검색
  - [ ] 유사도 임계값 설정 (기본 0.85)

#### SPEC-030: 신규 인플루언서 자동 발견 🔄 연기
- **목표**: 신규 요리사/채널 자동 발견 및 등록
- **산출물**: `indexing/discovery_service.py`
- **상태**: MVP 후 검토 (수동 큐레이션이 품질 관리에 유리)
- **작업 내용**:
  - [ ] YouTube 관련 채널 발견 (YouTube API)
  - [ ] 트렌딩 영상에서 신규 채널 탐지
  - [ ] 요리 채널 여부 판별 로직

#### SPEC-031: Instagram 크롤러 구현 ⚠️ 제한적
- **목표**: Instagram 게시물에서 레시피 정보 추출
- **산출물**: `crawlers/instagram.py`
- **상태**: **후순위 (API 제한으로 실효성 낮음)**
- **제한 사항**:
  - Graph API는 자사 비즈니스 계정 콘텐츠만 접근 가능
  - 타인 피드 크롤링 불가
  - 비공식 방법은 법적/기술적 위험
- **권장**: 협업/파트너십 확보 시에만 구현 검토

#### SPEC-032: TikTok 크롤러 구현 ⚠️ 제한적
- **목표**: TikTok 영상에서 레시피 정보 추출
- **산출물**: `crawlers/tiktok.py`
- **상태**: **후순위 (API 제한으로 실효성 낮음)**
- **제한 사항**:
  - Research API는 학술/연구 목적만 승인
  - 영상 중심 → 텍스트 레시피 추출 어려움
  - 웹 스크래핑은 Instagram보다 더 어려움
- **권장**: Phase 6+에서 재검토

#### SPEC-033: Claude 파서 구현 🔄 연기
- **목표**: Claude를 사용한 레시피 파싱 (GPT 대체/보조)
- **산출물**: `parsers/claude_parser.py`
- **상태**: MVP 후 검토 (GPT가 충분한지 평가 후)
- **작업 내용**:
  - [ ] Anthropic API 연동
  - [ ] 레시피 추출 프롬프트 설계
  - [ ] GPT와의 결과 비교 검증

---

### ❌ 삭제된 스펙

다음 스펙은 분석 결과 삭제되었습니다:

1. **키워드 기반 보조 인덱싱** - 요리사 기준 인덱싱으로 충분, 복잡도 대비 가치 낮음
2. **조리 단계별 이미지 생성** - 비용 급증 (10단계 = $0.40/레시피), 썸네일만으로 충분
3. **벌크 저장 서비스** - 기본 저장 서비스로 대체 가능

---

## 8. 구현 우선순위 요약

### Phase 1: MVP 인프라 (1주) - 5개 스펙
| 순서 | 스펙 | 핵심 목표 |
|------|------|----------|
| 1 | SPEC-001 | 크롤링 대상 인덱스 (시드 데이터 중심) |
| 2 | SPEC-002 | 프록시 매니저 (ScraperAPI 단일) |
| 3 | SPEC-003 | 도메인별 전략 (YouTube + 만개의레시피만) |
| 4 | SPEC-004 | 재시도 핸들러 |
| 5 | SPEC-005 | 사전 중복 검사 |

### Phase 2: 크롤링 코어 (1.5주) - 8개 스펙
| 순서 | 스펙 | 핵심 목표 |
|------|------|----------|
| 6 | SPEC-006 | 데이터 스키마 |
| 7 | SPEC-007 | 크롤러 인터페이스 |
| 8 | SPEC-008 | YouTube 크롤러 |
| 9 | SPEC-009 | 만개의레시피 크롤러 |
| 10 | SPEC-010 | GPT 파서 |
| 11 | SPEC-011 | 레시피 정규화 |
| 12 | SPEC-012 | 해시 중복 검사 |
| 13 | SPEC-013 | 비용 추적 시스템 ⭐신규 |

### Phase 3: 이미지 생성 (0.5주) - 2개 스펙
| 순서 | 스펙 | 핵심 목표 |
|------|------|----------|
| 14 | SPEC-014 | 이미지 생성 (DALL-E 3, 썸네일만) |
| 15 | SPEC-015 | S3 이미지 저장소 |

### Phase 4: 플랫폼 확장 (2주) - 6개 스펙
| 순서 | 스펙 | 핵심 목표 |
|------|------|----------|
| 16 | SPEC-016 | 네이버 블로그 크롤러 |
| 17 | SPEC-017 | 티스토리 크롤러 |
| 18 | SPEC-018 | 품질 점수 시스템 ⭐신규 |
| 19 | SPEC-019 | 태그 추출 |
| 20 | SPEC-020 | 요리사 정규화 |
| 21 | SPEC-021 | 재료/단계 정규화 |

### Phase 5: 운영 (1주) - 6개 스펙
| 순서 | 스펙 | 핵심 목표 |
|------|------|----------|
| 22 | SPEC-022 | 레시피 저장 |
| 23 | SPEC-023 | 크롤링 로그 |
| 24 | SPEC-024 | 파이프라인 통합 |
| 25 | SPEC-025 | 스케줄러 |
| 26 | SPEC-026 | CLI |
| 27 | SPEC-027 | 모니터링/알림 |

### Phase 6+: 후순위 (MVP 완료 후) - 6개 스펙
| 스펙 | 상태 | 비고 |
|------|------|------|
| SPEC-028 | 🔄 연기 | 노출 점수 계산 |
| SPEC-029 | 🔄 연기 | 임베딩 유사도 검사 |
| SPEC-030 | 🔄 연기 | 자동 채널 발견 |
| SPEC-031 | ⚠️ 제한적 | Instagram (API 제한) |
| SPEC-032 | ⚠️ 제한적 | TikTok (API 제한) |
| SPEC-033 | 🔄 연기 | Claude 파서 |

---

### 예상 일정 요약

| Phase | 기간 | 스펙 수 | 목표 레시피 수 |
|-------|------|--------|--------------|
| Phase 1-2 | 2.5주 | 13개 | 500개 (MVP 검증) |
| Phase 3 | 0.5주 | 2개 | 이미지 포함 |
| Phase 4 | 2주 | 6개 | 2,000개 |
| Phase 5 | 1주 | 6개 | 운영 안정화 |
| **총계** | **6주** | **27개** | **5,000개** |

---

## 9. 기술 스택

### 크롤링 인프라
- `httpx`: 비동기 HTTP 클라이언트
- `playwright`: JavaScript 렌더링, iframe 처리
- `beautifulsoup4`: HTML 파싱
- `yt-dlp`: YouTube 데이터 추출

### 프록시
- `httpx` 프록시 설정
- 유료: ScraperAPI, Bright Data
- 무료: Webshare (테스트용)

### LLM
- `openai`: GPT API
- `anthropic`: Claude API

### 이미지 생성
- 나노바나나 API (추천)
- `openai`: DALL-E 3 (대체)
- `boto3` / `google-cloud-storage`: 이미지 저장소

### 데이터 처리
- `pydantic`: 데이터 검증
- `sqlalchemy`: ORM (naecipe_backend와 공유)
- `asyncpg`: PostgreSQL 비동기 드라이버

### 벡터 검색
- `pgvector`: PostgreSQL 벡터 확장
- `sentence-transformers`: 임베딩 생성

### 스케줄링
- `apscheduler`: 작업 스케줄링
- `celery` (선택적): 분산 작업 큐

---

## 10. 환경 변수

```env
# 데이터베이스 (naecipe_backend와 공유)
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/naecipe

# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4-turbo-preview

# Claude (선택적)
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-3-sonnet-20240229

# YouTube
YOUTUBE_API_KEY=...

# Instagram (선택적)
INSTAGRAM_ACCESS_TOKEN=...

# 네이버
NAVER_CLIENT_ID=...
NAVER_CLIENT_SECRET=...

# 프록시 서비스
PROXY_SERVICE=scraperapi  # scraperapi, brightdata, webshare, none
SCRAPERAPI_KEY=...
BRIGHTDATA_USERNAME=...
BRIGHTDATA_PASSWORD=...
WEBSHARE_API_KEY=...

# 이미지 생성
IMAGE_PROVIDER=nanobanana  # nanobanana, dalle3
NANOBANANA_API_KEY=...

# 이미지 저장소
IMAGE_STORAGE=s3  # s3, gcs
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_S3_BUCKET=naecipe-images

# 크롤링 설정
CRAWL_RATE_LIMIT=10  # 초당 요청 수
CRAWL_BATCH_SIZE=100  # 배치 크기
RETRY_MAX_ATTEMPTS=3
RETRY_INITIAL_DELAY=1.0
```

---

## 11. 디렉토리 구조

```
naecipe_ai_crawler/
├── indexing/                    # 크롤링 대상 인덱싱 (신규)
│   ├── crawl_target_manager.py  # 크롤링 대상 관리
│   ├── tier_calculator.py       # 티어 계산
│   ├── discovery_service.py     # 신규 채널 발견
│   └── seed_data/               # 초기 시드 데이터
│       ├── youtube_channels.json
│       ├── instagram_accounts.json
│       ├── naver_blogs.json
│       └── tiktok_accounts.json
├── infra/                       # 인프라 레이어 (신규)
│   ├── proxy_manager.py         # 프록시 풀 관리
│   └── retry_handler.py         # 재시도 핸들러
├── strategies/                  # 도메인별 전략 (신규)
│   ├── base.py                  # 전략 인터페이스
│   ├── registry.py              # 전략 레지스트리
│   ├── youtube.py               # YouTube 전략
│   ├── naver_blog.py            # 네이버 블로그 전략
│   ├── mangae.py                # 만개의레시피 전략
│   ├── tistory.py               # 티스토리 전략
│   └── instagram.py             # Instagram 전략
├── schemas/
│   └── crawling.py              # 데이터 모델
├── crawlers/
│   ├── base.py                  # 베이스 크롤러
│   ├── youtube.py               # YouTube 크롤러
│   ├── instagram.py             # Instagram 크롤러
│   ├── naver_blog.py            # 네이버 블로그 크롤러
│   ├── tistory.py               # 티스토리 크롤러
│   └── recipe_sites.py          # 레시피 사이트 크롤러
├── parsers/
│   ├── base.py                  # 베이스 파서
│   ├── openai_parser.py         # GPT 파서
│   ├── claude_parser.py         # Claude 파서
│   └── tag_extractor.py         # 태그 추출
├── normalizers/
│   ├── chef_normalizer.py       # 요리사 정규화
│   ├── ingredient_normalizer.py
│   ├── step_normalizer.py
│   └── recipe_normalizer.py
├── image/                       # 이미지 처리 (신규)
│   ├── generation_service.py    # 이미지 생성
│   ├── providers/
│   │   ├── nanobanana.py        # 나노바나나
│   │   └── dalle.py             # DALL-E
│   └── storage_service.py       # 이미지 저장소
├── dedup/
│   ├── pre_crawl_checker.py     # 사전 중복 검사 (신규)
│   ├── hash_checker.py          # 해시 중복 검사
│   ├── embedding_checker.py     # 임베딩 유사도
│   └── duplicate_checker.py     # 통합 서비스
├── scoring/
│   ├── exposure_scorer.py       # 점수 계산
│   └── score_updater.py         # 점수 업데이트
├── ingestion/
│   ├── chef_service.py          # 요리사 저장
│   ├── recipe_service.py        # 레시피 저장
│   ├── crawl_log_service.py     # 크롤링 로그 (신규)
│   └── bulk_service.py          # 벌크 저장
├── pipeline/
│   └── crawling_pipeline.py     # 통합 파이프라인
├── scheduler/
│   └── crawl_scheduler.py       # 스케줄러
├── cli/
│   └── commands.py              # CLI 명령어
├── monitoring/
│   ├── stats.py                 # 통계
│   └── alerts.py                # 알림
├── tests/                       # 테스트
├── CLAUDE.md                    # Claude Code 가이드
├── SPECKIT_TODO.md              # 이 문서
└── pyproject.toml               # 프로젝트 설정
```

---

## 12. 참고 문서

- `naecipe_plan/5-1-1_DOMAIN.md`: 도메인 모델 정의
- `naecipe_plan/5-1-2_SYSTEM.md`: 시스템 아키텍처 (DB 스키마 포함)
- `naecipe_plan/5-1-3_AI_AGENT.md`: AI 에이전트 설계 (Crawler Agent)
- `naecipe_plan/5-1-4_API.md`: Ingestion API 설계
- `naecipe_backend/app/recipes/models.py`: 실제 DB 모델
- `naecipe_backend/app/recipes/schemas.py`: Pydantic 스키마

---

**최종 업데이트**: 2025-12-11
**작성자**: Claude Code
**버전**: 3.0 (크롤링 대상 인덱싱 전략 추가)

### 변경 이력
- v3.0 (2025-12-11): 크롤링 대상 인덱싱 전략 추가 (요리사 기준 인덱싱, 티어 분류, 시드 데이터, SPEC-000/001 추가)
- v2.0 (2025-12-11): 프록시, 크롤링 전략, 이미지 생성, 사전 중복검사 추가
- v1.0 (2025-12-11): 초기 버전
