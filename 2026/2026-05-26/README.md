# 2026-05-26 · 오늘 한 일

> 오늘은 **세 갈래**의 작업을 했다.
> ① 위키의 빈자리 메우기 (서블릿 스펙) ② 실습 정리 인제스트 (MyBatis) ③ 메타 학습 (RAG vs LLM Wiki)
> 결과: 위키 페이지 **8건 신설**, 백링크·교차연결 보강 **11곳**, 새 카테고리 1개 신설.

---

## 1. 서블릿 스펙이라는 게 무슨 뜻이야? — Query

### 왜 이걸 채웠나

위키 곳곳([[filter]], [[interceptor]], [[filter-vs-interceptor]])에 "서블릿 스펙"이라는 표현이 **정의 없이** 등장하고 있었다. lint 기준으로 보면 "언급은 되지만 독립 페이지가 없는 중요 개념"에 해당 — 빈 자리를 메워야 할 때다.

### 핵심 깨달음

- **스펙 = "인터페이스 + 동작 규약"만 정의.** Tomcat·Jetty·Undertow는 같은 스펙의 다른 구현체. 콘센트 규격서와 콘센트 제품의 관계.
- **[[dispatcher-servlet]] 자체도 `HttpServlet`을 상속한 한 개의 서블릿이다.** Spring MVC 전체가 결국 서블릿 스펙 위에 올린 한 겹의 추상화임이 이 지점에서 드러난다.
- [[filter]]가 Spring 없이 Tomcat만으로도 동작하는 반면 [[interceptor]]는 [[dispatcher-servlet]] 안에서만 사는 것 — 이게 "스펙 기술 vs 스프링 기술"의 위치 차이로 자연스럽게 설명된다.
- `javax → jakarta` 패키지 이동(Spring Boot 3.x / Tomcat 10+ 분기)이 실무에서 자주 부딪히는 함정.

### 위키 변경

- **신설**: [[servlet-spec]] — Jakarta EE 표준 명세서. 주요 인터페이스 표, "스펙 기술 vs 스프링 기술" 대조표, 패키지 이동 함정.
- **백링크 보강**: [[filter]] 3곳, [[interceptor]] 1곳, [[filter-vs-interceptor]] 3곳에 박혀 있던 "서블릿 스펙" 문자열을 모두 [[servlet-spec]]으로 연결.

---

## 2. MyBatis 실습 정리 — Ingest

### 무엇을 인제스트했나

`raw/notes/study-notes.md` — 부트캠프 MyBatis 단원 실습(`usertest` 프로젝트) 중 Claude Code와의 대화에서 정리한 **8가지 막힘 지점**.

정착 형태는 **"개념 + 디버깅 일지 둘 다"**로 결정. 한쪽만으론 부족했다 — 개념만 있으면 실전 함정이 빠지고, 일지만 있으면 재사용이 어렵다.

### 핵심 깨달음

- **MyBatis ≠ ORM.** SQL Mapper다. SQL 통제권은 개발자에게 남는다. JPA와 우열이 아니라 **SQL 통제권 선택의 문제**. (한국 SI가 MyBatis를 선호하는 이유가 여기 있다.)
- JDBC의 반복 코드(연결·해제·ResultSet)는 사라졌지만 **새로운 함정**(XML namespace, `@PathVariable` 매칭 시점, `RuntimeException`은 자동 500)이 그 자리를 채운다. **추상화는 항상 트레이드오프** — 사라진 자리에 다른 게 들어온다.
- **응답의 의미는 헤더(HTTP status code), 데이터는 본문(DTO/Wrapper).** 두 축은 분업이지 대체가 아니다. → API Response Wrapper를 도입했다고 모두 200으로 통일하면 함정에 빠진다.
- Filter에서 던진 예외는 `@ControllerAdvice`에 잡히지 않는다 — Filter가 DispatcherServlet **앞**에 있기 때문. ([[filter-vs-interceptor]]의 위치 차이가 예외 처리에서 그대로 재현)

### 위키 변경

- **신설(개념 3)**:
  - [[mybatis]] — SQL Mapper 메타. ORM이 아닌 이유, namespace↔인터페이스 매핑, `#{}` vs `${}` 보안, `@Mapper` 방식.
  - [[api-response-wrapper]] — `{success, message, data}` 공통 응답 래퍼. DTO 위 메타 한 겹. **헤더 vs 본문 분업** 강조.
  - [[http-status-codes]] — "클라이언트가 다르게 반응해야 하면 구분"이 판단 기준. 던지는 3가지 방법(`ResponseStatusException`, `@ResponseStatus`, `@ControllerAdvice`).
- **신설(소스 1)**: [[mybatis-practice-debugging]] — 8건 디버깅 일지. [[servlet-jdbc-debugging]]과 같은 형식.
- **백링크 보강 6건**: [[orm]], [[jdbc]], [[dao-pattern]], [[three-tier-architecture]], [[restful-api]], [[dto-vs-entity]] 모두에 새 페이지와의 교차 연결 추가.
- **의도된 미생성**: `pathvariable-vs-requestparam`은 보류 — 소스 페이지 체크리스트 #8에 패턴만 남김. (관성으로 페이지를 다 만들지 않는 것도 중요한 판단)

---

## 3. RAG의 작동 원리와 활용 — Query (메타 학습)

### 왜 이걸 다뤘나

"RAG가 뭔지 대충은 알겠는데 정확한 작동 원리와 실제 활용을 알아 둬야겠다." → 사실 **이 저장소(NoteWiki) 자체가 RAG의 대안 패턴 위에 있다.** 그래서 RAG를 정확히 이해하는 건 곧 NoteWiki 운영 철학의 배경을 이해하는 일이다.

### 핵심 깨달음

- **RAG의 본질** = 답변 직전에 외부 문서를 검색해 컨텍스트에 주입하는 것. 3단계: Indexing(chunk→임베딩→벡터 DB) → Retrieval(질문 임베딩→유사도 top-k) → Generation(LLM이 컨텍스트 위에서 답변).
- **임베딩의 마법** = 분포 가설. 비슷한 맥락에서 등장하는 텍스트는 비슷한 벡터로 표현된다. 그래서 **키워드 일치가 아닌 의미 유사도**로 검색이 가능해진다.
- **함정 하나**: 인덱싱과 쿼리에 **같은 임베딩 모델**을 써야 한다. 모델이 다르면 같은 좌표계가 아니다 → 모델 버전 갈아끼우려면 전체 재인덱싱.
- **벡터 DB가 따로 있는 이유**: 일반 RDB는 정확 매칭에 최적화되어 있다. 벡터 검색은 ANN(근사 최근접) — HNSW·IVF 같은 특수 인덱스가 필요.
- **NoteWiki와의 관계**: Karpathy의 [[llm-wiki-pattern]]은 RAG의 **대안**으로 제시된 것. chunk 검색 대신 AI가 미리 컴파일한 위키를 유지한다. 우열이 아니라 트레이드오프:
  - 자료가 방대·자주 갱신 → RAG
  - 자료가 정제 가능·연결성·누적성이 가치 → Wiki
- 영균의 비전("AI 색인 도구")에서 가치는 **chunk가 아니라 연결**. 그래서 Wiki 패턴이 옳은 선택. 자료가 커지면 둘을 합칠 수 있다 — 위키 + raw/에 대한 RAG 보완.

### 위키 변경

- **신설(개념 4)**: [[rag]], [[embedding]], [[vector-database]], [[llm-wiki-pattern]]
- **신설(흐름 1)**: [[rag-vs-llm-wiki]] — 두 패턴의 트레이드오프, 선택 기준, 하이브리드 가능성.
- `index.md`에 **새 카테고리 "AI / LLM / 지식 검색"** 신설 — 위키의 지식 영역이 한 축 더 늘었다.

---

## 오늘의 위키 변동 요약

| 항목 | 변화 |
| --- | --- |
| 개념 페이지 | 71 → 79 (**+8**) |
| 흐름 페이지 | 9 → 10 (**+1**) |
| 소스 페이지 | 19 → 20 (**+1**) |
| 통합 원본 | 19 → 20 (**+1**) |
| 새 카테고리 | "AI / LLM / 지식 검색" |
| 백링크 보강 | 11곳 |

---

## 오늘을 관통하는 메타 인사이트

세 작업이 따로 떨어진 듯하지만 **하나의 흐름**으로 연결된다:

1. **서블릿 스펙** — Spring MVC가 결국 서블릿 스펙 위에 올린 한 겹의 추상화임을 드러냈다. ([[dispatcher-servlet]]도 결국 `HttpServlet`이다)
2. **MyBatis 함정** — JDBC의 반복 코드를 없앤 자리에 새로운 함정이 들어왔다. (추상화는 트레이드오프)
3. **RAG vs Wiki** — 매번 검색이 만든 단편성의 자리에, 미리 컴파일된 연결을 두는 패턴이 등장했다. (이 역시 트레이드오프)

→ **공통 패턴**: 추상화·자동화는 무언가를 가려준 만큼 **다른 자리에 새로운 책임을 만든다**. 원리를 파악한다는 건 "사라진 게 어디로 갔나"를 추적하는 일이다.

이건 NoteWiki의 핵심 원칙과도 정렬된다 — "**무엇**보다 **왜 이렇게 동작하는가**." 오늘 한 세 가지 모두 그 질문에 답하는 작업이었다.

---

## 다음 단계 후보

- 자료가 충분히 커지면 **하이브리드 패턴 시도** — 위키 + raw/에 대한 임베딩 검색 보완. (지금은 자료가 작아 Wiki만으로 충분, 하지만 1년 후엔?)
- [[mybatis-practice-debugging]]의 체크리스트 #8 (`pathvariable-vs-requestparam`)이 실무에서 다시 부딪히면 그때 개념 페이지로 승격.
- [[servlet-spec]] 신설로 [[filter]]·[[interceptor]] 본문의 "서블릿 스펙" 언급이 모두 깔끔히 링크된 상태 → 다음 lint 때 재점검할 패턴이 줄었다.
