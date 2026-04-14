# TIL — 2026.04.14

오늘 부트캠프에서 **웹 기초 개념**과 **JavaScript 비동기 처리**를 배웠다.

---

## 1. 웹 기초

### 인터넷 vs 웹

- **인터넷**: 전 세계 컴퓨터들이 연결된 분산형 네트워크 인프라 자체
- **웹**: 인터넷 위에서 동작하는 문서 공유 서비스 중 하나
- 비유: 인터넷 = 도로망, 웹 = 그 도로 위를 달리는 택배 서비스

### 웹의 구성

- **웹 서버**: 데이터 저장, 클라이언트 요청 해석 → 응용 프로그램 실행 → 결과 응답 전송 (Nginx, Apache 등)
- **클라이언트**: 브라우저가 서버에 문서를 요청하고, 받은 HTML을 화면에 렌더링

### 정적 사이트 vs 동적 사이트

| 구분 | 설명 |
|------|------|
| 정적 | 저장된 파일을 그대로 전달. 누가 요청해도 동일한 내용 |
| 동적 | 요청 시점/사용자에 따라 서버가 내용을 생성해서 전달 |

### URL 구조

```
http://www.naver.com:80/경로명
 │         │          │    └── 리소스 경로
 │         │          └─────── TCP/IP 포트 (HTTP=80, HTTPS=443, 보통 생략)
 │         └────────────────── 서버 주소 (도메인)
 └──────────────────────────── 프로토콜
```

### PowerShell 실행 정책

```powershell
Get-ExecutionPolicy                                      # 현재 정책 확인
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser      # 개발 환경 추천 설정
```

| 정책 | 설명 |
|------|------|
| Restricted | 스크립트 실행 완전 차단 (기본값) |
| AllSigned | 서명된 스크립트만 실행 |
| **RemoteSigned** | 로컬은 자유, 외부 스크립트는 서명 필요 ← 개발 환경 추천 |
| Unrestricted | 모두 허용 (비권장) |

---

## 2. JavaScript 비동기 처리

### 동기 vs 비동기

- **동기**: 앞 작업이 끝나야 다음으로 넘어감. UI가 블로킹될 수 있음
- **비동기**: 작업을 시작해놓고, 결과를 기다리는 동안 다른 일을 처리 가능
- JS는 **싱글 스레드** 언어라 I/O 작업을 동기로 처리하면 UI가 멈춰버림 → 비동기가 필수

### 비동기 구현 방식의 발전

```
Callback (구식)  →  Promise (개선)  →  async/await (현재 표준)
```

세 방식은 서로 다른 게 아니라 계층적으로 발전한 것:
- `async/await`는 내부적으로 `Promise` 사용 (문법 설탕)
- `Promise`는 `Callback` 패턴을 체계화한 것

#### ① Callback

```javascript
setTimeout(() => console.log("1초 후"), 1000);
```

- 완료 후 할 일을 함수 **안에** 직접 중첩해서 써야 함
- 단계가 깊어질수록 코드가 오른쪽으로 밀리는 **콜백 지옥(Pyramid of Doom)** 발생

```javascript
// 콜백 지옥 예시
fetchUser(userId, (user) => {
    fetchPosts(user.id, (posts) => {
        fetchComments(posts[0].id, (comments) => {
            // 계속 중첩...
        });
    });
});
```

#### ② Promise

```javascript
new Promise(resolve => setTimeout(resolve, 1000))
  .then(() => console.log("1초 후"));
```

- 완료 신호를 **객체로 들고 다니면서** `.then()`으로 밖에서 자유롭게 연결 가능
- 상태: `pending` → `fulfilled` / `rejected`
- `resolve(값)` 호출 시: 상태 변경 + `.then()`에 값 전달
- `reject(에러)` 호출 시: 상태 변경 + `.catch()`에 에러 전달
- **한 번 상태가 바뀌면 다시는 안 바뀜** (불변성) → `resolve()`를 두 번 호출해도 첫 번째만 유효

#### ③ async/await

```javascript
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function main() {
  await sleep(1000);
  console.log("1초 후");
}
```

- 코드가 **동기처럼 읽힘**, 가독성 대폭 향상
- `await`는 **Promise를 반환하는 함수** 앞에 붙여야 의미 있음
- `async` 함수는 내부에서 뭘 `return`하든 항상 `Promise`로 감싸서 반환함
- 에러 처리를 `try/catch`로 깔끔하게 할 수 있음

**같은 동작, 다른 표현 비교:**

```javascript
// 콜백 방식
fetchUser(userId, (user) => {
    fetchPosts(user.id, (posts) => { ... });
});

// async/await 방식
const user     = await fetchUser(userId);
const posts    = await fetchPosts(user.id);
const comments = await fetchComments(posts[0].id);
```

### Promise 정적 메서드

여러 개의 Promise를 한꺼번에 다룰 때 사용.

| 메서드 | 동작 | 활용 |
|--------|------|------|
| `Promise.all` | 전부 성공 시 결과 배열 반환, **하나라도 실패 시 즉시 reject** | 여러 API를 동시에 호출하고 결과를 전부 써야 할 때 |
| `Promise.allSettled` | 성공/실패 상관없이 **전부 끝날 때까지 대기** | 일부 실패해도 나머지 결과를 써야 할 때 |
| `Promise.race` | **가장 먼저 완료된 것** (성공/실패 무관) 반환 | 타임아웃 구현 |
| `Promise.any` | **가장 먼저 성공한 것** 반환, 모두 실패 시 reject | 여러 서버 중 하나라도 응답하면 되는 상황 |

> `race` vs `any`: race는 실패도 결과로 인정, any는 성공만 결과로 인정

```javascript
// Promise.race로 타임아웃 구현
const result = await Promise.race([
  fetch('/api/data'),
  new Promise((_, reject) => setTimeout(() => reject('timeout'), 3000))
]);
```

---

## 오늘의 핵심 요약

- 인터넷 ≠ 웹. 웹은 인터넷 위에서 동작하는 서비스 중 하나.
- JS 비동기는 Callback → Promise → async/await 순으로 발전했고, 계층적으로 연결된 개념.
- `await`는 Promise를 반환하는 것 앞에 붙여야 의미 있다.
- Promise는 한 번 상태가 바뀌면 불변이다.
