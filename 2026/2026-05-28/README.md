# 2026-05-28 · 오늘 한 일

> 어제 JPA에서 "작업대(영속성 컨텍스트)의 수명 = 트랜잭션 경계"라는 걸 잡았다.
> 오늘은 그 **트랜잭션 경계를 누가, 어떻게 긋는가**로 들어갔다 — `@Transactional`과 **프록시**.
> Daker-server 프로젝트를 강의 복습하며 재점검하다 나온 대화를 위키에 인제스트.
> 결과: 위키 개념 **2건 신설**, 소스 1건 신설, 기존 페이지 **3곳 보강**.
> 한 줄 요약 — **"`@Transactional`이 대신 짜주는 try-catch는 누가, 언제 실행하나"를 푼 날.**

---

## 무엇을 인제스트했나

`raw/dialogues/2026-05-28 Spring 도메인 구조와 @Transactional — 프록시 vs 컨테이너.md` —
Daker-server 프로젝트를 보며 나눈 8턴 대화. 인제스트하면서 두 가지를 지시했다:
① 자주 쓰는 **Spring 어노테이션을 따로 모아 인덱스**로 만들 것,
② **JPA 개념 페이지를 대화에서 나온 도식으로 더 직관적으로 보강**할 것.

---

## 1. `@Transactional` = 바닐라 try-catch의 "선언적 버전"

### 핵심 깨달음

- 어제 [[entity-manager]]로 직접 짠 트랜잭션 코드(`tx.begin()` / `commit()` / `rollback()` / `em.close()`)가
  그대로 `@Transactional` 한 줄로 압축된다. 직접 짜면 **프로그래밍 방식(programmatic)**,
  어노테이션에 위임하면 **선언적(declarative)** 방식.
- ⚠️ **함정 — 기본 롤백은 unchecked 예외만.** 바닐라 `catch(Exception)`은 다 잡지만,
  `@Transactional`은 기본적으로 `RuntimeException`·`Error`에서만 롤백한다. **checked 예외
  (`IOException` 등)는 롤백 없이 그대로 commit**된다. 막으려면 `rollbackFor = Exception.class`.
- **`readOnly = true`는 단순 힌트가 아니다.** JPA가 "안 바뀜"을 알면 [[persistence-context|변경 감지]]
  스냅샷을 **아예 안 만든다** → 조회가 많을수록 메모리·속도 이득. 조회 메서드엔 `readOnly`, 변경
  메서드엔 기본 `@Transactional`이 정석.

---

## 2. 프록시 ≠ 컨테이너 — 내가 헷갈렸던 지점

### 핵심 깨달음

- `@Transactional`이 붙은 빈을 호출하면, 내가 받은 객체는 진짜 객체가 아니라 **프록시**다.
  프록시가 `begin/commit/rollback`을 감싸고 진짜 메서드에 위임한다.
- **컨테이너와 프록시는 다른 것이다.** 컨테이너(앱당 1개)가 **프록시(빈마다 1개)를 만들어 주입**한다.
  주입은 **앱 시작 시점**, 트랜잭션 처리는 **호출 시점** — 호출할 땐 이미 컨테이너는 빠져 있고 프록시만 일한다.
- ⚠️ **자기호출(self-invocation) 함정.** 프록시는 *바깥에서 들어오는* 호출만 가로챈다.
  같은 클래스 안에서 `this.다른메서드()`로 부르면 프록시를 안 거쳐 **`@Transactional`이 안 먹는다.**
  (네트워크 [[proxy]]와 같은 "중간에 낀 대리자" 아이디어가 JVM 객체 수준에서 반복된 것.)

---

## 3. 트랜잭션 전파(propagation) — 합칠까, 새로 열까

### 핵심 깨달음

- **`REQUIRED`(기본값)는 새 트랜잭션을 안 만든다 — 기존 것에 합친다.** 서비스가 다른 서비스를
  불러도 트랜잭션은 1개. 그래서 **안쪽 예외가 바깥까지 통째로 롤백**시킨다.
- ⚠️ **`UnexpectedRollbackException` 함정.** 안쪽에서 난 예외를 바깥에서 try-catch로 잡아 "살린"
  것 같아도, 트랜잭션이 이미 **rollback-only**로 표시돼 커밋 시점에 예외가 터진다. 한 트랜잭션을
  공유하니 부분 커밋이 안 되는 것.
- 진짜로 분리하려면 **`REQUIRES_NEW`** — 안쪽이 독립된 트랜잭션을 새로 연다(로그 적재 등에 사용).

---

## 위키 변경

- **신설(개념 2)**:
  - [[spring-annotations]] — 지시 ①. 자주 쓰는 어노테이션을 7개 그룹(빈 등록·DI·웹 매핑·응답/예외·
    트랜잭션/AOP·JPA 매핑·설정) 표로 정리, 각 행이 "왜" 페이지로 링크. **"어노테이션은 표시일 뿐,
    읽는 쪽(컨테이너/Hibernate)이 일한다"**는 큰 그림 도식 포함.
  - [[transaction-propagation]] — `REQUIRED` vs `REQUIRES_NEW`, rollback-only + `UnexpectedRollbackException` 도식, 전파 옵션 6종.
- **신설(소스 1)**: [[spring-transaction-proxy-dialogue]] — 8턴 대화 요약.
- **보강 3곳**(지시 ② — 대화 도식 이식):
  - [[transaction]] — "바닐라 JPA와의 대응" 섹션, programmatic vs declarative, readOnly의 "왜", 기본 롤백 규칙, 자기호출 경고.
  - [[entity-manager]] — "컨테이너가 EntityManager를 관리한다" 2단계 도식, 공유 EM 프록시(ThreadLocal).
  - [[aop]] — "컨테이너 vs 프록시" 구분 절, 자기호출 함정.
- 백링크: [[spring-framework]] 허브에 두 신설 페이지 추가.

### 오늘의 위키 변동 요약

| 항목 | 변화 |
| --- | --- |
| 개념 페이지 | 92 → 94 (**+2**) |
| 소스 페이지 | 22 → 23 (**+1**) |
| 통합 원본 | 22 → 23 (**+1**) |
| 기존 페이지 보강 | 3곳 |

---

## 어제와 오늘을 관통하는 메타 인사이트

이틀이 **하나의 사슬**로 이어진다 — 둘 다 "**사라진 코드가 어디로 갔나**"를 추적했다.

| | 사라진 것 | 어디로 갔나 |
| --- | --- | --- |
| 5/27 JPA | `UPDATE` SQL | 작업대([[persistence-context]])의 변경 감지 |
| 5/28 트랜잭션 | `begin/commit/rollback` try-catch | 프록시(AOP)의 가로채기 |

그리고 두 추상화가 **같은 경계에서 만난다**: [[transaction|트랜잭션]] 경계 = 작업대의 수명 =
프록시가 `begin`/`commit`을 긋는 구간. 어제 "작업대 수명 = 트랜잭션"이라 했는데, 오늘 그 트랜잭션을
**프록시가 연다**는 걸 알게 됐다. 퍼즐의 양쪽이 맞물린 셈.

→ 추상화는 편하지만 **가린 것을 모르면 함정에 빠진다.** 오늘 만난 함정 셋(checked 예외 미롤백,
자기호출, `UnexpectedRollbackException`)이 전부 "프록시·트랜잭션이 실제로 어떻게 도는지"를
몰라서 생기는 것이었다. 원리를 보면 함정이 보인다.

---

## 다음 단계 후보

- **fetch join / `@EntityGraph`** (어제부터 밀린 후보) — LazyInitializationException + N+1 동시 해결.
- `@Transactional`의 **AOP 프록시 종류**(JDK 동적 프록시 vs CGLIB) — 자기호출 함정의 근본 이유와 연결.
- Daker-server 프로젝트에 위 원리들을 실제로 적용하며 함정 셋을 코드로 재현·확인.
