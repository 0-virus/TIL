# 2026-05-14 작업 일지

## 완료된 작업

---

### 1. 식단 캘린더 레이아웃 재구성 + AI 생성 폼 통합
**브랜치:** `feature/meal-plan-layout-ai-form` → PR #87 → dev → main  
**커밋:** `005aca9`, `df0497d`

#### 변경 내용
- **MealPlanPage 레이아웃 개편**
  - 달력을 전체 너비(12열)로 확장
  - 달력 아래 행: 식단 정보 4열 + 대체식단 8열(조건부) 분리
  - `showAlternate` 조건: `needs-alt` 또는 `has-alt` 상태인 날짜에서만 대체식단 카드 표시

- **AI 생성 버튼 이동 (선택 모드 전용)**
  - 기존 DayDetailPanel 내부 → 선택 모드 헤더 버튼으로 이동
  - 1개 이상 날짜 선택 시에만 활성화
  - 비연속 날짜 선택 후 클릭 시 경고 메시지만 표시, 폼 미생성
  - 버튼 클릭 시 페이지 최하단에 AI 섹션 표시 + `scrollIntoView` 자동 스크롤

- **PanelAiSection 전체 폼으로 확장**
  - 기존: 선호 식재료 + 제외 식재료 2필드만 존재
  - 변경: 삭제된 AIMealPlanPage의 전체 폼 이식
    - 기간 (from/to, 선택 날짜 자동 채움)
    - 영양소 목표 (동적 추가/삭제/자동 계산)
    - NEIS 학교 검색
    - 단가 제약 토글
    - 선호/제외 식재료
  - 저장 후 `onSaved()` 콜백 호출로 페이지에 결과 반영

- **DayDetailPanel 경량화**
  - AI 섹션/버튼 제거, AlternatePlanCard 제거
  - 제거된 props: `month`, `onAiSaved`

- **MealItemRow: 알레르기 태그 위치 변경**
  - 인라인 행 내부 → 메뉴명 위 별도 `flex-wrap` 행으로 이동

- **AllergenBadge 색상 변경**
  - 일반 알레르기 번호 뱃지: `secondary(회색)` → `#cae9ff(연파랑)`

#### 버그 수정
- dev → main 머지 충돌 해결: DayDetailPanel의 `month` prop 및 `AlternatePlanCard` 잔존 코드 제거
- TypeScript 에러: `MealPlanPage`에서 DayDetailPanel에 존재하지 않는 `month={month}` 전달 제거 (`69df878`)
- CI lint 에러 2건 수정 (`f94f9ff`)

---

### 2. 달력 UX 개선 + 대시보드 버그 수정
**브랜치:** `feature/ux-calendar-drag-dashboard-fixes` → PR #89 → dev  
**커밋:** `db8ae4c`

#### 변경 내용
- **MonthlyMealCalendar 날짜 셀 hover 팝업 애니메이션**
  - `cubic-bezier(0.34, 1.56, 0.64, 1)` 스프링 커브로 셀이 위로 튀어오르는 효과
  - 월 외 날짜(disabled)는 애니메이션 미적용

- **selectMode 드래그 다중 선택**
  - `mousedown` → 드래그 시작, 시작 셀 선택 여부로 add/remove 모드 자동 결정
  - `mouseover` → 날짜 범위 실시간 프리뷰 (추가: 노란색, 제거: 빨간색 하이라이트)
  - `mouseup` → 범위 내 날짜 일괄 커밋
  - 드래그 중 hover 팝업 억제, `user-select: none` 적용

- **알레르기 유형별 분포 파이차트 라벨 겹침 수정**
  - 인라인 label 제거 → Legend formatter로 이름 + % 표시
  - 항목 수에 비례한 동적 높이: `max(280, 200 + ⌈n/2⌉ × 26)`

- **대시보드 학교 현황: 알레르기 보유 학생 수 표시**
  - `getSchoolStats`에 confirmed 알레르기 보유 학생 카운트 쿼리 추가
  - 총 학생 수 옆에 "N명 알레르기 보유" 표시

---

### 3. 서비스 랜딩 페이지 추가
**브랜치:** `feature/landing-page` → PR #90 → dev  
**커밋:** `c81a1ea`, `e06c63d`

#### 변경 내용
- **LandingPage.tsx 신규 생성**
  - 공개 라우트 `/` — 미인증 사용자 진입점, 로그인 시 `/dashboard`로 리다이렉트
  - 구성: Navbar(글래스모피즘) · Hero · Features(6개 카드) · How it works(3단계) · CTA · Footer
  - Hero: 좌측 텍스트 영역 + 우측 마스코트 이미지

- **LandingPage.css 신규 생성**
  - `@keyframes float3d`: `translateY + rotate + scale + drop-shadow`로 마스코트 3D 부유 효과
  - `@keyframes fadeInUp`: Hero 텍스트 순차 등장 애니메이션 (4단계 stagger)
  - 글래스모피즘 Navbar (`backdrop-filter: blur(12px)`)
  - Feature 카드 hover 효과

- **마스코트 이미지** (`frontend/public/mascot.png`)
  - `mix-blend-mode: multiply` 적용으로 배경 체커보드 패턴 제거

- **How it works 스텝 중앙 정렬**
  - 번호 원이 각 설명 텍스트의 가운데 위에 위치하도록 레이아웃 재구성
  - 연결선을 좌/우 절반으로 분할하여 원 중앙 배치

- **라우팅 변경** (`App.tsx`, `LoginPage.tsx`, `Sidebar.tsx`)
  - 기존 `/` → DashboardPage를 `/dashboard`로 이동
  - `/` → LandingPage (공개, 비인증)
  - 로그인 후 리다이렉트: `/` → `/dashboard`
  - Sidebar 모든 역할의 대시보드 링크: `/` → `/dashboard`

---

## PR 목록

| PR | 제목 | 머지 대상 |
|----|------|----------|
| #87 | feat(fe): 식단 캘린더 레이아웃 재구성 + AI 생성 폼 통합 | dev |
| #89 | feat(fe,be): 달력 UX 개선 + 대시보드 버그 수정 3종 | dev |
| #90 | feat(fe): 서비스 랜딩 페이지 추가 | dev |

dev → main 직접 머지 (충돌 해결 포함)

---

## 수정된 주요 파일

| 파일 | 변경 내용 |
|------|----------|
| `frontend/src/pages/MealPlanPage.tsx` | 레이아웃 재구성, AI 섹션 통합 |
| `frontend/src/components/meal/PanelAiSection.tsx` | 전체 AI 폼으로 확장 |
| `frontend/src/components/meal/DayDetailPanel.tsx` | 경량화 (AI/Alternate 제거) |
| `frontend/src/components/meal/MealItemRow.tsx` | 알레르기 태그 위치 변경 |
| `frontend/src/components/allergen/AllergenBadge.tsx` | 뱃지 색상 변경 |
| `frontend/src/components/MonthlyMealCalendar.tsx` | hover 애니메이션 + 드래그 선택 |
| `frontend/src/pages/AnalyticsDashboardPage.tsx` | 파이차트 라벨 + 알레르기 학생 수 |
| `backend/src/services/analytics/analytics.service.ts` | 알레르기 보유 학생 카운트 쿼리 |
| `frontend/src/pages/LandingPage.tsx` | 신규 생성 |
| `frontend/src/pages/LandingPage.css` | 신규 생성 |
| `frontend/public/mascot.png` | 신규 추가 |
| `frontend/src/App.tsx` | 라우팅 변경 (`/` → LandingPage) |
| `frontend/src/pages/LoginPage.tsx` | 로그인 후 리다이렉트 경로 수정 |
| `frontend/src/components/layout/Sidebar.tsx` | 대시보드 링크 경로 수정 |
