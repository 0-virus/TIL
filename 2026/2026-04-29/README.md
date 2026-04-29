# 2026-04-29

오늘은 **백엔드 선택과 배포 아키텍처** , **GitHub Pages 정적 배포** , **React Router / fetch 패턴**까지 풀스택 전환의 핵심 주제를 한 번에 훑은 날.

*** ** * ** ***

1. 환경 / 도구
----------

### `nodemon App.js`가 터미널에서 안 먹히는 이유

`nodemon`이 `devDependencies`에 있을 때는 `node_modules/.bin/`에만 존재하고 전역 PATH엔 없음.

해결 3가지:

```bash
npx nodemon App.js                    # 추가 설치 없이 즉시 실행
npm run dev                           # package.json scripts에 등록
npm install -g nodemon                # 전역 설치 (어디서나 가능)
```

Node 진입점 파일은 관례상 `app.js` / `index.js` (소문자). Linux 배포 환경은 대소문자를 구분하므로 `App.js`는 잠재적 함정.

*** ** * ** ***

2. 백엔드 선택: json-server vs Express vs NestJS
-------------------------------------------

|    항목     | json-server | Express.js |          NestJS           |
|-----------|-------------|------------|---------------------------|
| 셋업        | 거의 0줄       | 직접 구성      | 직접 구성                     |
| CRUD      | 자동 생성       | 직접 구현      | 직접 구현                     |
| 커스텀 로직    | 거의 불가       | 완전 자유      | 완전 자유                     |
| DB        | JSON 파일 고정  | 아무 DB나     | 아무 DB나                    |
| 인증 / RBAC | 기본 없음       | 직접 구현      | 데코레이터 + Guard             |
| 스케줄러      | ❌           | node-cron  | `@Cron()` 내장              |
| 구조 강제     | ---         | 자유         | Module/Service/Controller |
| 적합 상황     | 프로토타입 / 목업  | 실서비스 전반    | 도메인이 많고 구조 잡힌 팀 프로젝트      |

**핵심**: json-server는 "가짜 백엔드"고, Express는 "진짜 백엔드를 만드는 도구". NestJS는 그 위에 얹는 프레임워크 --- Spring Boot의 Node 사촌.

**알레르기 관리 시스템 같은 프로젝트**(RBAC, 알림 스케줄러, AI 연동, AES-256 암호화, 500명 동접)는 json-server로는 절대 불가. Express 또는 NestJS로 가야 함.

Java를 했다면 NestJS가 오히려 익숙할 수 있다 (`@Service` ↔ `@Injectable`, `@Scheduled` ↔ `@Cron`).

*** ** * ** ***

3. Express 배포 정석 프로세스
---------------------

    로컬 개발 → GitHub push → GitHub Actions (테스트/빌드/Docker push)
            → EC2 (Nginx ↔ Express + PM2 ↔ DB)

### 각 도구의 역할 분리

**Docker** = 앱을 박스에 포장 (Node.js 런타임 + 패키지 + 코드 통째로). "내 컴에서는 되는데" 문제 제거.

**EC2** = 그 박스를 올려놓을 컴퓨터 (AWS 클라우드 서버).

**Nginx** = 80/443 → 3000으로 리버스 프록시 + HTTPS(Let's Encrypt) 처리.

**PM2** = Express 앱이 크래시나면 자동 재시작.

**GitHub Actions** = push 트리거로 빌드 → Docker Hub push → EC2 SSH 배포까지 자동화.

### `package.json` vs Docker

`package.json`은 "패키지 목록"만 보장 → Node.js 자체 버전은 별개.

Docker는 OS + Node.js 버전 + 패키지 전부 박스에 넣어버려서 환경 자체를 고정함.

소규모 프로젝트는 Node 버전 차이로 깨지는 경우 많지 않아서 Docker 없이 EC2에 직접 npm install로 운영해도 됨.

### Vercel은 Express 백엔드에 부적합

Vercel은 서버리스(요청이 올 때만 함수 실행). 상시 실행 서버가 아니라서 `node-cron` 같은 스케줄러가 동작 안 함.

**현실적 조합**: 프론트는 Vercel, 백엔드는 Render (Render 무료 플랜은 15분 idle 시 sleep → 첫 요청 30초\~1분 지연되는 함정).

이 조합을 쓰면 Docker / EC2 / Nginx / PM2 전부 필요 없음. 학교 프로젝트는 Render+Vercel, 포트폴리오 정석은 EC2+Docker.

*** ** * ** ***

4. React fetch 패턴 --- 자주 터지는 함정 4가지
-----------------------------------

### ① `async function App()` 금지

```jsx
// ❌ Only Server Components can be async at the moment.
async function App() { ... }

// ✅ 컴포넌트는 동기, fetch는 useEffect 안에서
function App() {
  const [movies, setMovies] = useState([]);
  useEffect(() => {
    async function load() {
      const res = await fetch(...);
      setMovies(await res.json());
    }
    load();
  }, []);
}
```

### ② `SyntaxError: Unexpected token '<', "<!doctype "`

서버가 JSON 대신 HTML 페이지를 반환했는데 `res.json()`으로 파싱 → 첫 글자 `<`에서 폭발.

원인: 백엔드 미기동 / URL 오타 / 포트 번호 잘못 / API 경로 누락.

진단: `res.status`, `res.headers.get("content-type")`, `await res.text()` 찍어보면 즉시 보임.

### ③ json-server는 `?id=`로 필터 못 함

|        요청        |          결과          |
|------------------|----------------------|
| `/movies?rank=1` | ✅ 배열 반환              |
| `/movies?id=1`   | ❌ id는 특별 취급, 동작 안 함  |
| `/movies/1`      | ✅ 객체 반환 (path param) |

→ id로 단건 조회는 **반드시 path**, 응답 형태도 객체 vs 배열로 다름.

### ④ `useParams`는 항상 문자열

```jsx
const { id } = useParams();        // "1"
Number(id) === 1                   // ✅
id === 1                           // ❌ 항상 false
```

`useEffect` 의존성 배열에 `[id]` 꼭 넣어야 다른 영화로 이동 시 재fetch.

### App.jsx 구성 순서 (이 순서대로 쓰면 에러 거의 안 남)

import (라이브러리 → 컴포넌트 → 스타일)

함수 선언 (async 금지)

`useState`

`useEffect`

이벤트 핸들러

early return (loading/error)

JSX

`export default`

*** ** * ** ***

5. GitHub Pages 정적 배포에서 4가지가 동시에 터지는 이유
---------------------------------------

|               증상                |                                 원인                                 |                                 해결                                 |
|---------------------------------|--------------------------------------------------------------------|--------------------------------------------------------------------|
| 배포 빈 화면, `/assets/*.js` 404     | `vite.config.js`에 `base` 누락                                        | `base: "/저장소명/"` 추가                                                |
| `main` 브랜치 그대로 배포 → `dist/`가 없음 | 빌드 결과물 미배포                                                         | `gh-pages` 패키지 또는 GitHub Actions로 `dist/` 푸시                       |
| `/movies/1` 새로고침 시 404          | `BrowserRouter`는 SPA 라우트를 모르는 정적 호스팅에 부적합                          | `HashRouter`로 교체 (URL이 `/#/movies/1` 형태)                           |
| 데이터가 비어있음                       | `fetch("http://localhost:3001/...")` --- 사용자 PC의 3001을 호출 → 무조건 실패 | `db.json`을 `public/`으로 옮기고 `${import.meta.env.BASE_URL}db.json` 호출 |

### 자잘하지만 자주 만나는 것들

`package.json`**에** `"scripts"`**키 중복** → JSON은 뒤의 키만 유효 → `build` 스크립트 사라지는 현상.

**GitHub Pages Source 설정** → `npm run deploy`가 `gh-pages` 브랜치에 푸시했어도, Settings \> Pages가 `main`을 가리키고 있으면 소스 코드를 그대로 서빙함. 반드시 `gh-pages` 브랜치로 변경.

**이미지 경로** : `db.json`에 `/살목지.webp`로 두면 배포 시 `github.io/살목지.webp` 호출 → 404. leading `/` 제거하고 컴포넌트에서 `${import.meta.env.BASE_URL}` prefix로 붙여야 함.

**정적 db.json은** `/db.json/movies/1`**같은 경로 안 됨** --- 정적 파일이라 라우팅이 없음. 전체 받아서 `.find()`로 클라이언트 필터링.

*** ** * ** ***

6. 결론적으로 오늘 정리된 사고방식
--------------------

**개발 환경** 과 **배포 환경**은 다르게 생각해야 한다 (json-server는 로컬 전용, localhost fetch는 배포에서 무조건 실패).

도구를 고를 때는 "기능이 되냐"보다 "프로젝트 규모와 요구사항에 적정하냐"를 먼저 본다 (json-server vs Express, Vercel vs Render, BrowserRouter vs HashRouter).

인프라 도구들은 역할이 분리되어 있다 (Docker = 포장, EC2 = 컴퓨터, Nginx = 트래픽 입구, PM2 = 프로세스 감시).
