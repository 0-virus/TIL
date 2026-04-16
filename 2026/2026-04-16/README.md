# 2026-04-16

오늘의 TIL (2026-04-16)
====================

1. Next.js App Router 라우트 핸들러 시그니처
----------------------------------

Next.js App Router는 핸들러를 `handler(request, context)` 형태로 **positional**하게 호출한다.

따라서 `request`를 쓰지 않더라도 **첫 번째 인자 자리는 반드시 비워둬야** 한다.

`export async function GET({ params }: ...)`처럼 첫 인자를 생략하면, 런타임에서는 그 자리에 Request 객체가 들어가 `{ params }` 구조분해가 Request 객체에 적용되어 `params === undefined`가 되고 `await params`에서 500이 난다.

관례: `_request: NextRequest` 또는 `_: NextRequest`로 밑줄 접두사를 붙여 unused-vars 린트를 회피한다.

2. Prisma BigInt + NextResponse.json 직렬화 문제
-------------------------------------------

Prisma 모델 PK(`id`)가 `BigInt`인 상태에서 `NextResponse.json({ post })`처럼 Prisma 객체를 그대로 반환하면 `TypeError: Do not know how to serialize a BigInt`가 발생한다.

원인: `NextResponse.json()`은 표준 `JSON.stringify`를 사용하는데, `JSON.stringify`는 BigInt를 직렬화하지 못함.

**단일 패치로 해결** : `lib/prisma.ts` 상단에 아래 3줄을 넣으면 전 라우트에서 자동으로 BigInt가 문자열로 직렬화된다.

```ts
(BigInt.prototype as any).toJSON = function () {
  return this.toString();
};
```

3. Turbopack 컴파일 에러는 전역적으로 전파된다
-------------------------------

한 파일의 컴파일 에러(예: `totalCount` 중복 선언)만으로도 dev 서버 전체가 동작하지 못해 **모든 라우트가 500 HTML**을 반환할 수 있다.

API 테스트가 전부 500이면 먼저 컴파일 에러부터 의심.

4. tsconfig: `baseUrl` deprecated
---------------------------------

TS 5.5+에서 `baseUrl`이 deprecated, TS 7.0에서 제거 예정.

`paths`는 TS 4.1+부터 `baseUrl`**없이도** `tsconfig.json`**위치 기준** 으로 상대 경로 해석이 되기 때문에, `baseUrl` 줄을 그냥 삭제하면 된다.

`"ignoreDeprecations": "6.0"`으로 경고만 끌 수도 있지만, TS 7.0에서 어차피 막히므로 지금 지우는 쪽이 깔끔.

경고 메시지 원문: `Option 'baseUrl' is deprecated and will stop functioning in TypeScript 7.0.`

5. Figma MCP 사용 가능
------------------

현재 환경에 Figma MCP가 설치되어 있어 아래 URL 패턴을 처리할 수 있다.

`/design/...` → `get_design_context` (레이아웃/컴포넌트/색상/타이포, 스크린샷, React+Tailwind 참고 코드)

`/board/...` → `get_figjam` (FigJam 와이어프레임 파싱)

`/make/...` → 지원됨

URL의 `?node-id=...`를 포함하면 특정 프레임만 집어서 볼 수 있음.

6. 와이어프레임 분석 --- Zero:om 블로그
----------------------------

(추론 아님, Figma에서 직접 확인한 내용) 보라색 계열 "우주/유니버스" 컨셉, 총 **11개 화면** 구성.

주요 화면: 메인 페이지, 내 정보 모달, 블로그 상세, Sidebar, 글쓰기, 로그인, 회원가입, 설정(기본 정보/유니버스 관리/글 관리).

현재 API로 이미 커버되는 부분과 매핑표를 도출함.

7. 와이어프레임 기준으로 현재 API가 커버 못 하는 기능
---------------------------------

`PUT /api/blogs/profile`**누락** --- 현재는 GET만 있음. 설정-기본정보 화면의 "변경하기" 버튼들이 동작 불가.

**댓글 목록 독립 조회 API 없음** --- `GET /api/posts/[postId]`에서 `include: { comments: true }`로 모든 댓글을 한 번에 반환 중. 페이지네이션 + 작성자 정보 포함한 별도 엔드포인트 필요.

**이웃 DELETE 및 신청 메시지 필드 부재** --- `Neighbors` 모델에 `message` 필드 없음, DELETE 엔드포인트 없음.

**친구 신청 승인 플로우 부재** --- 와이어프레임은 친구 목록 / 발견 목록 / 친구 신청 3탭 구조로 pending/accepted 구분을 전제. 현재 스키마는 status 필드 없이 일방적 Neighbors row만 존재. *(추론: 두 가지 선택지를 제시 --- A. 스키마에* `status`*enum 추가, B. 현재 모델 유지하고 양방향 row 존재로 "친구" 판정.)*

**임시저장 버튼 연동** --- 스키마는 `PostStatus.draft` 이미 지원. 프론트에서 `status: "draft"` 전달만 하면 됨. *(추론: 별도로 "내 임시저장 목록" 조회 엔드포인트는 필요할 수 있다.)*

8. 그 외 오늘 작업/수정 이력
------------------

`seed-dummy.ts`와 `test-apis.sh` 작성 --- 3 users / 3 blogs / 4 categories / 5 posts / 3 tags / 3 comments / 3 likes / 2 neighbors / 2 notifications 시드 + 65 케이스 테스트.

최종 **65/65 통과** 달성.

적용된 수정: `BigInt.prototype.toJSON` 패치, `app/api/blogs/[blogId]/categories/route.ts`의 POST 시그니처에 `_request: NextRequest` 추가.

NextAuth 콜백 오타 수정: `token.neckname → token.nickname`, `token.projile_img → token.profile_img`.

`frontend` 브랜치에 커밋 `c95d285`로 반영 (seed/test 스크립트와 `.claude/`는 커밋에서 제외).

작업 도중 실수로 `api` 브랜치에서 테스트를 진행한 후 `frontend` 브랜치로 옮겨 **처음부터 다시 수행**한 에피소드도 있었음.
