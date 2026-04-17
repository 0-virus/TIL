# 2026-04-17

1. Next.js + Tailwind CSS v4 프론트엔드 구현
-------------------------------------

### 프로젝트 구조

Next.js 16 App Router의 Route Group을 활용한 레이아웃 분리: `(auth)`, `(main)`

`(auth)` --- 네비게이션 없는 중앙 정렬 레이아웃 (로그인/회원가입)

`(main)` --- Topbar + Footer 포함 레이아웃 (메인 피드, 블로그, 설정 등)

### SessionProvider 설정

NextAuth의 `useSession`을 클라이언트 컴포넌트에서 쓰려면 `SessionProvider`로 감싸야 함

Server Component인 `app/layout.tsx`에서 직접 쓸 수 없으므로 `"use client"` 래퍼 컴포넌트(`AuthProvider.tsx`)를 만들어서 루트 레이아웃에 적용

### CSS 모듈 타입 에러

`import "./globals.css"` 시 TypeScript가 `Cannot find module or type declarations for side-effect import` 에러 발생

원인: TypeScript가 `.css` 파일을 모듈로 인식하지 못함

해결: 프로젝트 루트에 `global.d.ts` 생성 후 `declare module "*.css";` 선언

*** ** * ** ***

2. Tailwind CSS v4 커스텀 컬러 시스템
-----------------------------

### `@theme inline` 블록

Tailwind v4에서는 `@theme inline` 안에 `--color-*` CSS 변수를 선언하면 자동으로 유틸리티 클래스가 생성됨

예: `--color-primary: #9b59b6` → `bg-primary`, `text-primary`, `ring-primary` 등 사용 가능

```css
:root {
  --primary: #9b59b6;
  --primary-light: #d4a5e5;
}

@theme inline {
  --color-primary: var(--primary);
  --color-primary-light: var(--primary-light);
}
```

### ring 유틸리티

`ring-primary`는 ring의 **색상만** 지정하는 클래스

ring이 실제로 보이려면 **두께 클래스** (`ring-1`, `ring-2` 등)가 반드시 함께 있어야 함

올바른 사용: `ring-1 ring-gray-300`

### 색상 이름 주의

Tailwind의 회색 계열은 `grey`가 아니라 `gray` (미국식 철자)

`ring-grey-50`은 인식되지 않아 기본 검은색이 됨

*** ** * ** ***

3. CSS Gradient 종류
------------------

### 기본 타입

|        종류         |      설명       |
|-------------------|---------------|
| `linear-gradient` | 직선 방향         |
| `radial-gradient` | 중심에서 바깥으로     |
| `conic-gradient`  | 중심에서 시계 방향 회전 |

각각 `repeating-` 접두사를 붙여 반복 버전 사용 가능

### radial-gradient 모양 옵션

|                  옵션                  |           설명            |
|--------------------------------------|-------------------------|
| `circle`                             | 정원형                     |
| `ellipse`                            | 타원형 --- 화면 비율에 맞게 자연스러움 |
| `closest-side` / `farthest-side`     | 가장 가까운/먼 변까지            |
| `closest-corner` / `farthest-corner` | 가장 가까운/먼 모서리까지          |

### 위치 지정

```css
/* 특정 좌표를 중심으로 원형 그라데이션 */
radial-gradient(circle at 50% 10%, #d4a5e5, white 60%)

/* 크기 직접 지정 */
radial-gradient(120% 80% at 30% 0%, #d4a5e5, white)
```

### 여러 개 겹치기

```css
background:
  radial-gradient(ellipse at 30% 0%, #d4a5e5, transparent 50%),
  radial-gradient(ellipse at 80% 100%, aqua, transparent 40%),
  white;
```

### Tailwind에서의 그라데이션

`linear-gradient`는 Tailwind 유틸리티 지원: `bg-gradient-to-b from-primary-light to-background`

위치 지정도 가능: `from-0%`, `to-40%`, `via-30%` 등

`radial-gradient`와 `conic-gradient`는 Tailwind 기본 유틸리티 미지원 → `style` 속성으로 직접 CSS 작성 필요

*** ** * ** ***

4. API 구현 --- 이웃 목록 조회 \& 임시저장 목록 조회
------------------------------------

### GET /api/neighbors

로그인한 사용자가 발견한 이웃 블로그 목록 조회

상호 발견 여부(`isMutual`)를 별도 쿼리로 계산: 상대가 나를 발견한 Neighbors 레코드가 있는지 `Set`으로 비교

`filter` 쿼리 파라미터로 `mutual` / `one-way` 필터링 지원

### GET /api/posts/drafts

`PostStatus.draft` 상태인 게시글만 조회

`updated_at` 기준 최신순 정렬

본문은 200자까지 미리보기(`contentPreview`)로 반환

*** ** * ** ***

5. 기타
-----

### Windows 터미널 cURL 한글 인코딩 문제

Windows cmd/bash의 기본 인코딩(CP949)과 UTF-8 충돌로 cURL 테스트 시 한글이 깨져서 DB에 저장될 수 있음

API 코드 자체의 문제는 아님 --- 프론트엔드 `fetch`는 항상 UTF-8이라 정상 동작

테스트 환경 한정 문제

### 원격 DB 응답 속도

로컬 MySQL 대비 원격 PostgreSQL(Supabase 등)은 쿼리당 네트워크 왕복 시간이 추가됨

Next.js dev 모드에서는 API route 첫 호출 시 컴파일 오버헤드 + Prisma 커넥션 풀 초기화 비용도 추가

프로덕션에서는 서버와 DB가 같은 리전에 있으면 개선됨
