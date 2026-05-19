# Servlet 한 요청이 처리되는 전체 흐름 — Controller → DAO → JSP

> 2026-05-19 필기 · `jwbook` 실습 디버깅 복습용
> 근거: [[mvc-pattern]] · [[jdbc]] · [[dao-pattern]] · [[jstl-el]] · [[jsp]] · [[servlet-jdbc-debugging]]

브라우저가 보낸 요청 하나가 화면으로 돌아오기까지, 코드의 어느 계층을 거치는지를
한 장으로 정리한다. Spring으로 넘어가기 전, **이 골격을 손으로 짚어두는 것**이 목적이다.

---

## 큰 그림 — 한 요청의 여정

```
[브라우저]
   │  GET /student?action=list
   ▼
[Servlet / Controller]   ← 요청 해석, 흐름 제어   (MVC의 C)
   │  studentDAO.findAll()
   ▼
[DAO]                    ← SQL · JDBC 전담
   │  Connection → PreparedStatement → ResultSet
   ▼
[Database]
   ▲
   │  List<Student>
[Controller]
   │  req.setAttribute("studentList", list)
   │  req.getRequestDispatcher(view).forward(req, resp)
   ▼
[JSP / View]             ← 데이터를 HTML로 렌더링  (MVC의 V)
   │  ${studentList}, <c:forEach>
   ▼
[브라우저]  ◀── 완성된 HTML
```

핵심은 **세 계층이 각자 한 가지 일만 한다**는 것이다. 이 분리가 [[mvc-pattern]]이고,
DAO를 끼운 것은 "DB 접근"이라는 관심사를 Controller에서 한 번 더 떼어낸 것이다.

---

## 1. Controller — 요청을 해석하고 흐름을 정한다

`action` 파라미터를 보고 무슨 일을 할지 분기한 뒤, 결과를 담아 JSP로 넘긴다.
**SQL을 직접 쓰지 않는다** — 그건 DAO의 몫이다.

```java
String action = req.getParameter("action");
if (action == null) action = "list";   // ★ null 기본값 처리
switch (action) {
    case "list": ...
    case "form": view += "student/create.jsp"; break;
}
List<Student> list = studentDAO.findAll();   // DAO에 위임
req.setAttribute("studentList", list);       // 데이터를 담고
req.getRequestDispatcher(view).forward(req, resp);  // ★ 넘긴다
```

**왜 `forward`가 따로 필요한가** — `setAttribute`는 데이터를 *담는* 것이고
`forward`는 화면으로 *넘기는* 것이다. 별개의 두 단계라서, `forward`를 빼먹으면
view 경로 문자열만 만들어지고 화면은 그대로 멈춘다.

---

## 2. DAO + JDBC — 데이터 접근을 전담한다

[[dao-pattern]]은 SQL과 [[jdbc]] 호출을 한 객체에 모은다. Controller는
`findAll()`의 *의도*만 알면 되고, "어떤 SQL로 어떻게"는 DAO 안에 숨는다.

### 반환 타입이 의미를 말한다

| 메서드 | 반환 | 이유 |
| --- | --- | --- |
| `findAll()` | `List<Student>` | 결과가 여러 건 |
| `findById(id)` | `Student` / `null` | 단건, 없으면 null |
| `create(s)` | `void` / `int` | INSERT, int면 영향 행 수 |

### JDBC 4객체와 두 가지 실행 메서드

```
Driver → Connection → PreparedStatement → ResultSet
```

- **`executeQuery()`** — SELECT용, `ResultSet` 반환
- **`executeUpdate()`** — INSERT/UPDATE/DELETE용, **영향받은 행 수(int)** 반환

### `ResultSet`은 커서다

`rs.next()`는 커서를 다음 행으로 옮기고 **행이 있으면 `true`**를 준다.
이 반환값이 곧 루프 종료 조건이다.

```java
while (rs.next()) {              // 여러 건
    list.add(new Student(...));
}
if (rs.next()) { ... }           // 단건 — 없을 때(null)를 처리
```

### 자원은 연 순서의 역순으로 닫는다

```java
if (pstmt != null) pstmt.close();   // 하위 자원 먼저
if (conn  != null) conn.close();    // 상위 자원 나중
```

하위 자원(pstmt)은 상위 자원(conn)에 의존하므로 conn을 먼저 닫으면 안 된다.
`null` 체크는 `open()` 실패 시의 NPE를 막는다. 안 닫으면 커넥션이 쌓여 고갈된다.

---

## 3. JSP + EL/JSTL — 데이터를 HTML로 그린다

[[jsp]]는 View다. Controller가 `setAttribute`로 담은 값을 [[jstl-el]]로 꺼낸다.

```jsp
<%@ page contentType="text/html;charset=UTF-8" isELIgnored="false" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<c:forEach var="s" items="${studentList}">
    <td>${s.name}</td>
</c:forEach>
```

`<% %>` 스크립틀릿 대신 `${...}`(EL)과 `<c:forEach>`(JSTL)를 쓰면 화면이
선언적으로 깔끔해진다 — JSP가 다시 Java로 지저분해지는 것을 막는다.

---

## 자주 한 실수 — 다시 안 내려는 체크리스트

| 증상 | 원인 | 막는 법 |
| --- | --- | --- |
| `switch`에서 NPE | `getParameter`가 `null` | 진입 전 기본값 (`action="list"`) |
| `${...}`가 빈 값 | attribute 이름 불일치 | `setAttribute` 이름 = `${이름}` |
| `<c:forEach>` 무동작 | `taglib` URI 누락 | taglib 지시자 선언 |
| `${...}`가 글자로 출력 | `isELIgnored="false"` 누락 | page 지시자를 JSP마다 확인 |
| `SQLException`으로 끝남 | `while(true)`로 순회 | `while(rs.next())` |
| 커넥션 누수 | `close()` 비었거나 conn만 닫음 | pstmt→conn 역순 + null 체크 |
| 화면이 안 바뀜 | `forward` 누락 | `getRequestDispatcher().forward()` |
| 날짜가 이상함 | `"yyyy-mm-dd"`의 `mm`=분 | 월은 대문자 `MM` |
| `ClassCastException` | `(java.sql.Date) birth` 강제 캐스팅 | `new java.sql.Date(birth.getTime())` |
| HTTP 404 | 톰캣 `contextPath` 설정 | 코드 다 봤으면 **실행 환경**을 의심 |

> 디버깅의 원리: 오류는 **층(layer)** 으로 의심한다. 코드(JDBC·빌드)를 다 뒤져도
> 안 나오면, 그 아래 실행 환경(톰캣 설정)을 본다 — 404가 그 사례였다.

---

## Spring으로 이어지는 지점

지금 손으로 짠 것들이 Spring에서 무엇으로 바뀌는지 — 이게 [[servlet-to-spring-mvc]]의 핵심.

| 지금 (Servlet+JDBC)            | Spring MVC                      |
| ---------------------------- | ------------------------------- |
| `doGet`/`doPost` 오버라이드       | `@GetMapping`/`@PostMapping`    |
| 손으로 짠 DAO 클래스                | `@Repository` / Spring Data JPA |
| `Connection` 열고 `close()` 직접 | `DataSource`·`JdbcTemplate`이 관리 |
| JSP + JSTL/EL                | Thymeleaf 템플릿 엔진                |

**왜 이 순서로 배우나** — Spring은 위 오른쪽을 다 감춰준다. 하지만 막히면 결국
왼쪽으로 내려가야 한다. DAO의 반복 코드(연결·순회·해제)를 직접 겪어봐야,
`@Repository` 한 줄이 무엇을 없애주는지 "이해하고" 쓰게 된다.

---

_출처: 위키 페이지 [[mvc-pattern]] · [[jdbc]] · [[dao-pattern]] · [[jstl-el]] ·
[[jsp]] · [[servlet-jdbc-debugging]] · [[servlet-to-spring-mvc]] 종합._
