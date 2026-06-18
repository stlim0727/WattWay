# ADR-0001: 모노레포 (Turborepo + pnpm)

- 상태: Accepted
- 날짜: 2026-06-18

## 맥락

웹(Next.js)과 모바일(Expo)이 동일한 도메인 로직(잔량 모델, 충전 정차 최적화)과
타입(DB 스키마, API 계약)을 공유해야 한다. 별도 저장소로 나누면 타입 동기화 비용과
중복 구현 위험이 크다.

## 결정

단일 모노레포에 `apps/*`(web, mobile, routing-service)와 `packages/*`(core, types,
api-client, ui-tokens, config)를 둔다. Turborepo로 빌드/캐시를 관리하고 pnpm
workspaces로 의존성을 연결한다.

## 결과

- 장점: 타입/도메인 로직 단일 출처, 원자적 변경, 빌드 캐시.
- 단점: 초기 설정 복잡도, CI에서 영향 범위 빌드 필요(Turbo로 완화).
