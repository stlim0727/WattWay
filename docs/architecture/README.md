# WattWay 아키텍처 설계

> EV 충전/경로 플래너 — 전체 시스템 아키텍처 (v0.1)
>
> 작성일: 2026-06-18 · 상태: Draft

---

## 1. 제품 개요

WattWay는 전기차(EV) 운전자가 출발지 → 목적지 경로를 계획할 때, **배터리 잔량(SoC)** 과
**차량 주행거리**를 고려해 **최적의 충전 정차 지점을 자동으로 삽입**해 주는 서비스다.

핵심 가치:

- "여기서 출발해서 저기까지, 어디서 충전해야 하나?"를 한 번에 답한다.
- 충전소 커넥터 타입 / 충전 속도 / 실시간 가용 여부 / 요금을 종합해 추천한다.
- 웹과 모바일에서 동일한 경험을 제공한다.

### 1.1 MVP 범위 (Phase 1)

| 영역 | 포함 | 비고 |
|------|------|------|
| 사용자 인증 | ✅ | 이메일/소셜 로그인 |
| 차량 프로필 | ✅ | 배터리 용량, 커넥터, 전비(효율) |
| 충전소 검색 | ✅ | 위치/반경/필터(커넥터·속도·사업자) |
| 충전소 상세 | ✅ | 커넥터, 출력, 요금, 실시간 가용성 |
| **경로 계획** | ✅ | **잔량 기반 충전 정차 삽입 (핵심)** |
| 저장/즐겨찾기 | ✅ | 경로·충전소 즐겨찾기 |
| 충전 세션 기록 | ⏳ Phase 2 | 결제 연동 이후 |
| 결제/충전 개시 | ⏳ Phase 3 | 사업자 OCPI 연동 필요 |

---

## 2. 아키텍처 개요

### 2.1 스타일

- **모노레포 (Turborepo + pnpm workspaces)** — 웹/모바일/서버가 타입과 도메인 로직을 공유한다.
- **클라이언트-BFF-서비스** 3계층. 무거운 경로 탐색은 전용 라우팅 서비스로 분리한다.
- **Supabase**(Postgres + PostGIS, Auth, Realtime, Storage)를 기본 백엔드 플랫폼으로 사용하고,
  CPU 집약적인 경로 탐색만 별도 서비스로 둔다.

### 2.2 컨텍스트 다이어그램

```
                ┌───────────────┐      ┌───────────────┐
                │   Web (Next)  │      │ Mobile (Expo) │
                └───────┬───────┘      └───────┬───────┘
                        │      HTTPS / WSS      │
                        └───────────┬───────────┘
                                    ▼
                        ┌───────────────────────┐
                        │   API Gateway / BFF    │  (Next.js Route Handlers
                        │   - 인증 위임          │   또는 Supabase Edge Fn)
                        │   - 요청 검증/조합     │
                        └───┬───────────────┬────┘
                            │               │
              ┌─────────────▼───┐     ┌─────▼───────────────┐
              │  Supabase        │     │  Routing Service     │
              │  - Postgres+PostGIS    │  - 잔량 기반 경로    │
              │  - Auth (JWT)    │     │  - 충전 정차 최적화  │
              │  - Realtime      │     │  (Node/TS or Go)     │
              │  - Storage       │     └─────┬───────────────┘
              └───┬──────────────┘           │
                  │                          ▼
                  │              ┌──────────────────────────┐
                  │              │ External Routing Engine   │
                  │              │ (OSRM / Valhalla / Mapbox)│
                  │              └──────────────────────────┘
                  ▼
       ┌──────────────────────────┐
       │ Data Ingestion (Cron)     │
       │ - Open Charge Map / OCPI  │
       │ - 충전소·요금·가용성 동기화 │
       └──────────────────────────┘
```

---

## 3. 기술 스택

| 레이어 | 선택 | 이유 |
|--------|------|------|
| 모노레포 | Turborepo + pnpm | 빌드 캐시, 워크스페이스 공유 |
| 웹 | Next.js (App Router), React, TypeScript | SSR/SEO, BFF 겸용 |
| 모바일 | Expo (React Native), TypeScript | 웹과 로직/타입 공유, 단일 코드베이스 |
| UI 공유 | 디자인 토큰 + 플랫폼별 컴포넌트 | RN/웹 완전 공유는 지양, 토큰만 공유 |
| 지도 | Mapbox GL (웹) / Mapbox Maps SDK (모바일) | 경로·벡터타일·EV 라우팅 친화 |
| 상태/데이터 | TanStack Query + Zustand | 서버 캐시 + 클라이언트 상태 분리 |
| 백엔드 | Supabase (Postgres 15 + PostGIS) | Auth/Realtime/Geo 일괄 제공 |
| 인증 | Supabase Auth (JWT, RLS) | 행 수준 보안 일관 적용 |
| 경로 서비스 | Node.js/TypeScript 서비스 + OSRM/Valhalla | 도로망 라우팅 위임, 정차 최적화 자체 구현 |
| 충전소 데이터 | Open Charge Map → (확장) OCPI | 공개 데이터로 시작 |
| 배포 | Vercel(웹) / EAS(모바일) / Fly.io·Render(라우팅 서비스) | |

> 라우팅 서비스 언어는 TypeScript로 시작(팀 공유 용이)하고, 성능 병목 시 Go로 이관 가능하도록
> 알고리즘과 I/O를 분리한다. (ADR-0003 참고 예정)

---

## 4. 모노레포 구조

```
wattway/
├── apps/
│   ├── web/                  # Next.js (웹 + BFF Route Handlers)
│   ├── mobile/               # Expo (React Native)
│   └── routing-service/      # 경로 탐색 서비스
├── packages/
│   ├── core/                 # 도메인 로직 (순수 TS): 잔량 모델, 정차 최적화
│   ├── types/                # 공유 타입 (DB 스키마 → 생성된 타입 포함)
│   ├── api-client/           # 타입 안전 API 클라이언트 (web/mobile 공용)
│   ├── ui-tokens/            # 색상/타이포/스페이싱 토큰
│   └── config/               # eslint, tsconfig, tailwind 프리셋
├── supabase/
│   ├── migrations/           # SQL 마이그레이션
│   └── functions/            # Edge Functions (데이터 인제스트 등)
└── docs/
    └── architecture/
```

핵심 원칙: **도메인 로직(`packages/core`)은 프레임워크/플랫폼 비의존**.
경로 정차 최적화 같은 비즈니스 규칙을 순수 함수로 두어 웹·모바일·서버가 모두 재사용한다.

---

## 5. 데이터 모델 (개요)

PostGIS의 `geography(Point)` 타입으로 위치를 저장하고 GiST 인덱스로 반경 검색을 가속한다.

```
users (Supabase auth.users 확장: profiles)
  └─ vehicles            # 사용자 차량 (model_id, 현재 SoC 기본값)
        └─ vehicle_models  # 카탈로그: battery_kwh, 전비(Wh/km), 지원 커넥터, 최대 충전출력

charging_stations          # location(geography), operator, address
  └─ connectors            # type(CCS2/CHAdeMO/Type2), power_kw, price_model
        └─ availability     # 실시간 상태(available/occupied/offline), updated_at

routes                     # 사용자가 계획/저장한 경로 (origin, destination, vehicle_id)
  └─ route_stops           # 정차 지점 (station_id 또는 경유지, arrival_soc, charge_to_soc, 순서)

favorites                  # 즐겨찾기 (대상: station | route)
station_reviews            # 리뷰/평점 (Phase 2)
```

> 상세 스키마와 DDL은 별도 작업(데이터 모델/DB 스키마)에서 마이그레이션으로 확정한다.
> 본 문서는 엔티티 관계와 경계를 정의하는 수준까지만 다룬다.

---

## 6. 핵심 알고리즘 — 잔량 기반 경로 + 충전 정차 최적화

이 서비스의 차별점이자 가장 복잡한 부분.

### 6.1 입력

- 출발지, 목적지 (좌표)
- 차량: 배터리 용량(kWh), 전비(Wh/km), 지원 커넥터, 최대 충전출력(kW)
- 현재 SoC(%), 도착 시 목표 최소 SoC(안전 버퍼, 예: 10%)

### 6.2 처리 단계

1. **기본 경로 산출**: 외부 라우팅 엔진(OSRM/Valhalla)으로 도로 경로 + 구간별 거리/표고 획득.
2. **에너지 모델**: 구간별 소비 에너지 추정 (거리·표고·속도·기본 전비, Phase 2에서 기상/온도 보정).
3. **도달 가능성 검사**: 무충전 도달 가능하면 정차 없음.
4. **후보 충전소 수집**: 경로 회랑(corridor) 내 충전소를 PostGIS 반경 질의로 수집,
   차량 커넥터 호환 필터.
5. **정차 최적화**: 경로를 충전 그래프로 모델링.
   - 노드 = 충전소(도착 SoC 상태 포함), 엣지 = 충전소 간 주행(에너지 소비).
   - 제약: 모든 구간에서 SoC ≥ 안전 버퍼.
   - 목적함수: **총 소요시간 최소화**(주행 + 충전), 부가로 비용/정차 횟수 가중.
   - 알고리즘: 라벨링 기반 최단경로(자원 제약 최단경로, RCSP) 또는
     실용적 그리디 + 지역 탐색으로 시작.
6. **결과**: 정차 목록(어디서·몇 %까지·예상 충전시간) + 구간별 ETA + 총 비용 추정.

### 6.3 설계 결정

- 알고리즘은 `packages/core`에 **순수 함수**로 두고, 외부 라우팅 엔진 호출은
  주입(adapter)으로 분리 → 단위 테스트 가능, 엔진 교체 가능.
- 1차는 정확성/명료성 우선(그리디+검증), 2차에서 RCSP로 최적성 강화.

---

## 7. 횡단 관심사

| 관심사 | 방침 |
|--------|------|
| 인증/인가 | Supabase Auth JWT, Postgres RLS로 행 수준 격리 |
| 실시간 | 충전소 가용성은 Supabase Realtime 채널로 구독 |
| 캐싱 | 경로 결과/충전소 목록은 BFF 단기 캐시 + 클라이언트 TanStack Query |
| 관측성 | Sentry(에러), 구조화 로그, 라우팅 서비스 지연 메트릭 |
| 비밀/설정 | 환경변수 + Supabase secrets, 클라이언트엔 anon key만 노출 |
| 데이터 동기화 | Cron Edge Function이 Open Charge Map 증분 동기화 |

---

## 8. 단계별 로드맵

- **Phase 1 (MVP)**: 인증, 차량 프로필, 충전소 검색/상세, 경로 계획(그리디), 즐겨찾기.
- **Phase 2**: RCSP 최적화, 에너지 모델 고도화(표고/기온), 리뷰, 충전 세션 기록.
- **Phase 3**: OCPI 연동(실시간 가용성·요금·원격 충전 개시), 결제.

---

## 9. 다음 설계 작업 (후속)

1. **데이터 모델/DB 스키마 확정** — PostGIS DDL + RLS 정책 + 마이그레이션.
2. **API 설계** — BFF 엔드포인트 및 요청/응답 스펙(특히 `/routes/plan`).
3. **경로 알고리즘 상세 설계** — 에너지 모델 공식, RCSP 라벨링 정의.
4. **UI/화면 플로우** — 지도·경로·정차 카드 와이어프레임.

> 이 문서는 후속 설계의 기준점이다. 결정이 바뀌면 ADR로 기록한다.
