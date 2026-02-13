# 🍜 맛플리 (MatPly) 개발 계획서 v1.0

> YouTube 좋아요 영상에서 AI가 찾아주는 나만의 맛집 리스트

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|------|------|
| **서비스명** | 맛플리 (MatPly) |
| **목적** | YouTube 좋아요/재생목록 영상에서 AI가 맛집 영상을 자동 분류하여 음식점 리스트로 정리해주는 웹 서비스 |
| **핵심 가치** | 좋아요 목록 속 묻혀있는 맛집 영상을 발굴하여 실제 방문 계획으로 연결 |
| **타겟** | 유튜브에서 맛집/카페 영상에 좋아요를 누르지만 나중에 찾지 못하는 사용자 |
| **OAuth Scope** | `youtube.readonly` (Read Only) - 심사 간편 |
| **쿼터 전략** | 서버 중앙 처리 + 큐 시스템 (쿼터 증가 신청 병행) |

---

## 2. 핵심 기능

### ① Google OAuth 2.0 로그인 + 회원가입

- `youtube.readonly` scope로 좋아요/재생목록 읽기 권한만 획득
- Google 계정으로 회원가입 동시 처리, JWT 토큰 발급
- 초기 100명까지는 테스트 모드로 OAuth 심사 없이 운영 가능

### ② 영상 소스 선택

사용자가 처음에 두 가지 중 하나를 선택:

- **"좋아요 누른 영상"** - 좋아요 목록에서 수집
- **"재생목록의 영상"** - 특정 재생목록 선택 후 수집

> 💰 **과금 기준:** 수집된 영상 50개까지 무료 / 50개 초과 시 유료 플랜 ₩3,900

### ③ AI 2단계 필터링 (핵심 로직)

**1단계: 맛집 영상 필터링 (저비용)**

- YouTube API로 영상 제목 + description 배치 수집
- AI가 전체 영상 중 음식점/카페/맛집 관련 영상만 필터링
- 예: 1,000개 중 약 200개 맛집 영상 추출

**2단계: 음식점 정보 추출 (맛집만)**

- 필터된 맛집 영상의 상단 댓글 1개 추가 수집 (`commentThreads.list`, `order=relevance`, `maxResults=1`)
- AI가 제목 + description + 상단댓글에서 음식점명, 지역, 음식 종류 추출
- 추출 불가 시 "미확인 맛집"으로 분류

### ④ 맛집 리스트 화면

- **카드 형태:** 영상 썸네일 + 음식점명 + 지역 + 음식 종류
- **상태 관리:** "가보고 싶어요" / "가봤어요" 토글
- **메모:** 개인 메모 작성 가능
- **별점:** 1~5점 별점 부여 (가봤어요 상태일 때)
- **정렬:** 최신 영상순 / 별점순 선택 가능
- **영상 바로가기:** 카드 클릭 시 YouTube 영상으로 이동

---

## 3. 서비스 프로세스 (User Journey)

```
1. 웹사이트 접속 → Google 계정으로 로그인 (회원가입 동시 처리)
2. "좋아요 영상" 또는 "재생목록" 선택
3. 영상 수집 시작 (50개 초과 시 결제 안내)
4. AI가 맛집 영상 필터링 + 음식점 정보 추출
5. 맛집 리스트 화면에서 결과 확인
6. 각 음식점에 "가보고 싶어요" / "가봤어요" 상태 설정, 메모 및 별점 부여
```

---

## 4. YouTube API 쿼터 전략

### ⓐ 쿼터 소모 분석

Read Only로 변경되어 Write 쿼터가 제거되었으며, Read 작업은 모두 1 unit으로 매우 저렴합니다.

| API 메서드 | Units | 용도 | 호출 횟수 |
|-----------|-------|------|----------|
| `playlistItems.list` | 1 | 좋아요 목록 수집 | 50개씩 페이징 |
| `videos.list` | 1 | 영상 상세정보 배치 | 50개씩 배치 |
| `commentThreads.list` | 1 | 상단 댓글 수집 | 맛집 영상만 |

### ⓑ 유저당 쿼터 계산 (1,000개 영상 기준)

| 단계 | 계산 | Units |
|------|------|-------|
| 좋아요 목록 수집 (1,000/50) | 20회 | 20 |
| 영상 상세정보 배치 (1,000/50) | 20회 | 20 |
| 맛집 영상 댓글 수집 (약 200개) | 200회 | 200 |
| **합계** | | **약 240 units** |

> ✅ **쿼터 문제 완전 해결:** 기본 쿼터 10,000 units/일 기준으로 하루 약 40명 처리 가능. 쿼터 증가 신청 (무료) 시 500,000 units면 하루 약 2,000명 처리 가능.

---

## 5. Google OAuth 전략

### Scope 분류

| 구분 | Scope | 심사 |
|------|-------|------|
| Non-sensitive | `openid`, `email`, `profile` | ❌ 불필요 |
| **Sensitive** | **`youtube.readonly`** | ⚠️ 필요 (단, 테스트 모드 가능) |
| Restricted | `gmail.readonly` 등 | 🔴 심사 + 보안감사 |

### 운영 전략

- **Phase 1 (MVP, ~100명):** 테스트 모드로 심사 없이 운영. GCP 콘솔에서 테스트 유저 이메일만 등록.
- **Phase 2 (100명 초과):** OAuth 심사 제출. 준비물: 데모 영상, 개인정보처리방침, 홈페이지, scope 사용 목적 설명서.

### 심사 준비물 (Phase 2)

- 앱 기능 데모 영상 (YouTube 비공개)
- 개인정보처리방침 페이지 (본인 소유 도메인)
- 서비스 홈페이지 (본인 소유 도메인)
- scope 사용 목적 설명: "사용자의 좋아요 목록을 읽어 맛집 영상을 분류합니다"

---

## 6. 시스템 아키텍처

### ⓐ 기술 스택

| 계층 | 기술 | 역할 |
|------|------|------|
| **Frontend** | Next.js (App Router) + TypeScript | SSR/SEO, 사용자 UI |
| **Backend** | Node.js + Express + TypeScript | API 서버, 비즈니스 로직 |
| **Database** | PostgreSQL + Prisma ORM | 사용자/음식점/상태 저장 |
| **Job Queue** | BullMQ + Redis | 비동기 영상 수집/분류 |
| **AI** | OpenAI GPT-4o-mini | 맛집 필터링 + 정보 추출 |
| **Auth** | Google OAuth 2.0 + JWT | 인증/인가 |
| **Infra** | Vercel + GCP | 배포, API 관리 |

### ⓑ 데이터 플로우

```
OAuth 로그인
  → 소스 선택 (좋아요/재생목록)
  → YouTube API Read (description 수집)
  → [1단계] AI 맛집 필터링
  → 맛집 영상만 상단댓글 수집
  → [2단계] AI 음식점 정보 추출
  → DB 저장
  → 맛집 리스트 화면 표시
```

---

## 7. DB 스키마 (PostgreSQL + Prisma)

### users

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | 기본 키 |
| `google_id` | VARCHAR(255) UNIQUE | Google 고유 ID |
| `email` | VARCHAR(255) UNIQUE | 이메일 |
| `name` | VARCHAR(100) | 사용자명 |
| `profile_image` | TEXT | 프로필 이미지 URL |
| `access_token` | TEXT | YouTube API용 (암호화) |
| `refresh_token` | TEXT | 토큰 갱신용 (암호화) |
| `plan` | VARCHAR(20) | `free` / `premium` |
| `created_at` | TIMESTAMPTZ | 가입일 |
| `updated_at` | TIMESTAMPTZ | 수정일 |

### scan_jobs (수집/분류 작업)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | 기본 키 |
| `user_id` | INTEGER FK → users | users 참조 |
| `source_type` | VARCHAR(20) | `liked` / `playlist` |
| `playlist_id` | VARCHAR(255) | 재생목록 ID (source=playlist일 때) |
| `status` | VARCHAR(20) | `pending` / `collecting` / `classifying` / `completed` / `failed` |
| `total_videos` | INTEGER | 전체 영상 수 |
| `restaurant_count` | INTEGER | 추출된 맛집 수 |
| `created_at` | TIMESTAMPTZ | 작업 생성일 |
| `updated_at` | TIMESTAMPTZ | 작업 수정일 |

### restaurants (추출된 음식점)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | 기본 키 |
| `user_id` | INTEGER FK → users | users 참조 |
| `job_id` | INTEGER FK → scan_jobs | scan_jobs 참조 |
| `video_id` | VARCHAR(50) | YouTube 영상 ID |
| `video_title` | TEXT | 영상 제목 |
| `thumbnail_url` | TEXT | 썸네일 이미지 URL |
| `channel_name` | VARCHAR(255) | 채널명 |
| `restaurant_name` | VARCHAR(255) | AI가 추출한 음식점명 |
| `region` | VARCHAR(100) | 지역 (성수동, 이태원 등) |
| `food_type` | VARCHAR(100) | 음식 종류 (라멘, 피자 등) |
| `status` | VARCHAR(20) | `none` / `want_to_go` / `visited` |
| `rating` | SMALLINT | 별점 1~5 (visited일 때) |
| `memo` | TEXT | 개인 메모 |
| `published_at` | TIMESTAMPTZ | 영상 업로드일 (정렬용) |
| `created_at` | TIMESTAMPTZ | 생성일 |
| `updated_at` | TIMESTAMPTZ | 수정일 |

### Prisma Schema

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id           Int          @id @default(autoincrement())
  googleId     String       @unique @map("google_id")
  email        String       @unique
  name         String?
  profileImage String?      @map("profile_image")
  accessToken  String?      @map("access_token")   // 암호화 저장
  refreshToken String?      @map("refresh_token")  // 암호화 저장
  plan         String       @default("free")
  createdAt    DateTime     @default(now()) @map("created_at")
  updatedAt    DateTime     @updatedAt @map("updated_at")

  scanJobs     ScanJob[]
  restaurants  Restaurant[]

  @@map("users")
}

model ScanJob {
  id              Int          @id @default(autoincrement())
  userId          Int          @map("user_id")
  sourceType      String       @map("source_type")   // liked | playlist
  playlistId      String?      @map("playlist_id")
  status          String       @default("pending")    // pending | collecting | classifying | completed | failed
  totalVideos     Int?         @map("total_videos")
  restaurantCount Int?         @map("restaurant_count")
  createdAt       DateTime     @default(now()) @map("created_at")
  updatedAt       DateTime     @updatedAt @map("updated_at")

  user            User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  restaurants     Restaurant[]

  @@map("scan_jobs")
}

model Restaurant {
  id             Int       @id @default(autoincrement())
  userId         Int       @map("user_id")
  jobId          Int       @map("job_id")
  videoId        String    @map("video_id")
  videoTitle     String    @map("video_title")
  thumbnailUrl   String?   @map("thumbnail_url")
  channelName    String?   @map("channel_name")
  restaurantName String?   @map("restaurant_name")
  region         String?                              // 지역
  foodType       String?   @map("food_type")          // 음식 종류
  status         String    @default("none")           // none | want_to_go | visited
  rating         Int?                                 // 1~5
  memo           String?
  publishedAt    DateTime? @map("published_at")       // 영상 업로드일
  createdAt      DateTime  @default(now()) @map("created_at")
  updatedAt      DateTime  @updatedAt @map("updated_at")

  user           User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  scanJob        ScanJob   @relation(fields: [jobId], references: [id], onDelete: Cascade)

  @@map("restaurants")
}
```

---

## 8. API 설계

### Auth

| Method | Endpoint | 설명 |
|--------|----------|------|
| `POST` | `/api/auth/google` | Google OAuth 콜백 (회원가입/로그인 + JWT 발급) |
| `POST` | `/api/auth/refresh` | JWT 토큰 갱신 |
| `GET` | `/api/auth/me` | 내 정보 조회 |

### Scan (영상 수집/분류)

| Method | Endpoint | 설명 |
|--------|----------|------|
| `GET` | `/api/playlists` | 사용자 재생목록 목록 조회 |
| `POST` | `/api/scan` | 스캔 작업 시작 (`{ sourceType, playlistId? }`) |
| `GET` | `/api/scan/:jobId` | 스캔 작업 상태 조회 (폴링용) |

### Restaurant (맛집 관리)

| Method | Endpoint | 설명 |
|--------|----------|------|
| `GET` | `/api/restaurants` | 내 맛집 리스트 조회 (`?sort=latest\|rating&status=want_to_go\|visited`) |
| `PATCH` | `/api/restaurants/:id/status` | 상태 변경 (`{ status: "want_to_go" \| "visited" }`) |
| `PATCH` | `/api/restaurants/:id/rating` | 별점 부여 (`{ rating: 1~5 }`) |
| `PATCH` | `/api/restaurants/:id/memo` | 메모 수정 (`{ memo: "..." }`) |

### Payment (결제)

| Method | Endpoint | 설명 |
|--------|----------|------|
| `POST` | `/api/payment/check` | 결제 필요 여부 확인 (50개 초과?) |
| `POST` | `/api/payment/process` | 결제 처리 (카카오페이/토스페이먼츠) |

---

## 9. 수익화 모델

| 구분 | 무료 플랜 | 유료 플랜 (₩3,900) |
|------|----------|-------------------|
| 수집 영상 수 | 최대 50개 | 무제한 |
| 소스 선택 | 좋아요 / 재생목록 | 좋아요 / 재생목록 |
| AI 분류 | 동일 | 동일 |
| 상태/메모/별점 | 동일 | 동일 |
| 결제 방식 | - | 카카오페이 / 토스페이먼츠 |

---

## 10. 프로젝트 디렉토리 구조

```
matply/
├── frontend/                          # Next.js App Router
│   ├── src/
│   │   ├── app/
│   │   │   ├── page.tsx               # 랜딩 페이지
│   │   │   ├── dashboard/
│   │   │   │   └── page.tsx           # 대시보드 (소스 선택)
│   │   │   ├── scan/
│   │   │   │   └── [jobId]/
│   │   │   │       └── page.tsx       # 스캔 진행 상태
│   │   │   └── restaurants/
│   │   │       └── page.tsx           # 맛집 리스트
│   │   ├── components/
│   │   │   ├── common/                # Header, Footer, Loading 등
│   │   │   ├── auth/                  # GoogleLoginButton
│   │   │   ├── scan/                  # ScanProgress, SourceSelector
│   │   │   └── restaurant/            # RestaurantCard, StatusToggle, RatingStars, MemoEditor
│   │   ├── hooks/
│   │   │   ├── useAuth.ts
│   │   │   ├── useRestaurants.ts
│   │   │   └── useScan.ts
│   │   └── lib/
│   │       ├── api.ts                 # API 클라이언트 (axios)
│   │       └── utils.ts
│   ├── public/
│   ├── next.config.js
│   ├── tailwind.config.ts
│   ├── tsconfig.json
│   └── package.json
│
├── backend/                           # Node.js + Express + TypeScript
│   ├── src/
│   │   ├── config/
│   │   │   ├── database.ts            # Prisma 클라이언트 싱글톤
│   │   │   ├── oauth.ts               # Google OAuth 설정
│   │   │   ├── ai.ts                  # OpenAI 설정
│   │   │   └── redis.ts               # Redis 연결
│   │   ├── middleware/
│   │   │   ├── auth.ts                # JWT 인증 미들웨어
│   │   │   ├── errorHandler.ts        # 글로벌 에러 핸들링
│   │   │   └── planCheck.ts           # Free/Premium 플랜 체크
│   │   ├── routes/
│   │   │   ├── auth.routes.ts
│   │   │   ├── scan.routes.ts
│   │   │   ├── restaurant.routes.ts
│   │   │   └── payment.routes.ts
│   │   ├── controllers/
│   │   │   ├── auth.controller.ts
│   │   │   ├── scan.controller.ts
│   │   │   ├── restaurant.controller.ts
│   │   │   └── payment.controller.ts
│   │   ├── services/
│   │   │   ├── auth.service.ts        # OAuth 처리, JWT 발급
│   │   │   ├── youtube.service.ts     # YouTube API 호출 (좋아요, 재생목록, 댓글)
│   │   │   ├── classifier.service.ts  # [1단계] AI 맛집 필터링
│   │   │   ├── extractor.service.ts   # [2단계] AI 음식점 정보 추출
│   │   │   └── queue.service.ts       # BullMQ 작업 큐 관리
│   │   ├── workers/
│   │   │   └── scan.worker.ts         # 스캔 작업 워커 (수집 → 분류 → 추출)
│   │   ├── utils/
│   │   │   ├── encryption.ts          # 토큰 암호화/복호화
│   │   │   └── logger.ts              # 로깅
│   │   └── app.ts                     # Express 앱 진입점
│   ├── prisma/
│   │   └── schema.prisma
│   ├── tsconfig.json
│   └── package.json
│
├── .env.example                       # 환경변수 템플릿
├── docker-compose.yml                 # PostgreSQL + Redis 로컬 개발용
└── README.md
```

---

## 11. 개발 마일스톤

| Phase | 기간 | 주요 작업 |
|-------|------|----------|
| **Phase 1** | 1~2주 | 프로젝트 세팅, DB 스키마 (Prisma), Google OAuth + JWT, 기본 UI 레이아웃 |
| **Phase 2** | 2~3주 | YouTube API Read 연동 (좋아요/재생목록 수집), AI 2단계 필터링 구현, BullMQ 작업 큐 |
| **Phase 3** | 1~2주 | 맛집 리스트 UI (카드, 정렬, 필터), 상태 관리/메모/별점 기능 |
| **Phase 4** | 1주 | Free/Premium 분기, 결제 연동 (카카오페이/토스), 테스트, 배포 |

**총 예상 기간: 약 5~8주 (MVP 기준)**

---

## 12. 주요 리스크 & 대응

| 리스크 | 대응 방안 |
|--------|----------|
| AI 음식점명 추출 실패 | "미확인 맛집"으로 분류, 영상 제목을 대신 표시. 프롬프트 튜닝으로 정확도 개선 |
| YouTube API 쿼터 부족 | Read Only로 유저당 ~240 units. 쿼터 증가 신청 (무료, 3~5일) |
| OAuth 심사 지연 | <100명은 테스트 모드로 운영, Phase 1에서 심사 선제출 |
| 댓글 비활성화 영상 | 댓글 수집 실패 시 description만으로 추출 시도 (graceful fallback) |
| AI API 비용 증가 | GPT-4o-mini 사용으로 비용 최소화. 50개씩 배치 처리 |
| 해외 맛집 영상 | MVP는 한국 맛집 우선. 추후 다국어 지원 확장 |

---

## 13. 환경변수 (.env)

```env
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/matply

# Redis
REDIS_URL=redis://localhost:6379

# Google OAuth
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_REDIRECT_URI=http://localhost:3000/api/auth/google/callback

# JWT
JWT_SECRET=your_jwt_secret
JWT_EXPIRES_IN=7d

# OpenAI
OPENAI_API_KEY=AIzaSyDd8cDxF78rg16Omnr1e25xlmhTgn0UakM

# Payment (추후 추가)
# KAKAOPAY_CID=your_kakaopay_cid
# TOSS_SECRET_KEY=your_toss_secret_key

# App
NODE_ENV=development
PORT=8000
FRONTEND_URL=http://localhost:3000
```

---

## 14. 향후 확장 로드맵

| 단계 | 조건 | 내용 |
|------|------|------|
| **Phase A (현재 MVP)** | 0~100명 | 개인 맛집 리스트 도구 + 유료 플랜 ₩3,900 |
| **Phase B (소셜 확장)** | 1,000명+ | 다른 사용자들의 "가보고 싶어요" 랭킹, 지역별/음식별 인기 맛집 차트 |
| **Phase C (광고 플랫폼)** | 10,000명+ | 사장님 우선노출 월 구독 (₩30,000~100,000), 사장님 대시보드 제공 |

> ⚠️ Phase B 진입 시 음식점 마스터 테이블 추가 마이그레이션 필요 (동일 가게 매칭 로직)