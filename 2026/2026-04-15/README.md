# 2026-04-15

TIL --- HTML \& CSS 기초
======================

*** ** * ** ***

`<head>` vs `<body>`
--------------------

`<head>` : 브라우저를 위한 메타 정보 (title, meta, link, script 등). 화면에 렌더링되지 않음.

`<body>` : 실제로 화면에 렌더링되는 모든 콘텐츠.

`<link rel="stylesheet">`는 반드시 `<head>`에 두어야 함. `<body>` 중간에 두면 CSS 없이 날것의 HTML이 먼저 보였다가 CSS가 적용되면서 화면이 번쩍이는 현상(FOUC)이 생길 수 있음.

### 로드 순서

HTML 파일 수신 시작

브라우저가 위→아래로 파싱

`<head>` 안의 내용 처리 (CSS 다운로드, JS 다운로드 등)

`<body>` 파싱 시작 → DOM 구성

defer 스크립트 실행

페이지 완전히 로드됨
> `<head>`가 먼저 처리되긴 하지만, `<body>`가 다 파싱되기 전까지 화면에 아무것도 보이지 않음.

*** ** * ** ***

`defer` 속성
----------

`<script>` 태그에 붙이는 속성.
> "JS 파일 다운로드는 지금 시작하되, 실행은 HTML 파싱이 끝난 다음에 해줘"

```html
<head>
  <script src="main.js" defer></script>
</head>
```

`defer` 없음 : HTML 파싱 중 `<script>` 만나면 파싱 멈추고 JS 다운로드 + 실행 후 재개

`defer` 있음 : HTML 파싱하면서 JS 다운로드 병행 → 파싱 완료 후 JS 실행

*** ** * ** ***

DOM (Document Object Model)
---------------------------

브라우저가 HTML 파일을 읽으면 텍스트로 저장하는 게 아니라 **트리 구조의 객체**로 변환해서 메모리에 올려놓은 것.

    Document
    └── html
        └── body
            └── div#app
                └── p
                    └── "안녕"

DOM이 메모리에 올라와 있기 때문에 JavaScript로 페이지를 동적으로 변경할 수 있음.

HTML 파일 자체는 안 바뀌지만, 메모리의 DOM이 바뀌면 화면이 즉시 업데이트됨.

*** ** * ** ***

checkbox의 `value` 속성
--------------------

> 체크박스가 체크된 상태로 폼이 제출될 때, 서버로 전송되는 값

체크 O → 서버에 `name=value` 전송

체크 X → 전송 자체가 안 됨

`value`를 안 쓰면 기본값이 `"on"` → 어떤 체크박스가 선택됐는지 구분 불가

```javascript
checkbox.checked  // 체크 여부 (true / false)
checkbox.value    // value 속성값 (체크 여부 관계없이 항상 읽힘)
```

체크 여부는 반드시 `.checked`로 확인해야 함.

*** ** * ** ***

쿼리 스트링에 같은 이름 여러 개
------------------

HTTP 스펙 자체가 동일 키 중복을 허용함.

    fruit=apple&fruit=grape&fruit=banana

서버에서는 배열/리스트로 받음.

```java
// Spring
@RequestParam List<String> fruit  // → ["apple", "grape"]
```

```javascript
// JavaScript
params.get("fruit");      // "apple" (첫 번째만)
params.getAll("fruit");   // ["apple", "grape"] (전부)
```

> 같은 키가 여러 개일 땐 반드시 `getAll()`을 써야 함.

*** ** * ** ***

CSS 정렬
------

### `align-items`의 전제 조건

`align-items`는 반드시 **Flexbox 또는 Grid 컨테이너에서만** 작동함.

```css
/* 아무 효과 없음 */
div { align-items: center; }

/* 작동함 */
div { display: flex; align-items: center; }
```

### Flexbox의 두 축

**주축 (main axis)** : `flex-direction`이 결정 (기본값: 가로)

**교차축 (cross axis)** : 주축의 수직 방향

|        속성         |     담당 축      |
|-------------------|---------------|
| `justify-content` | 주축            |
| `align-items`     | 교차축           |
| `align-content`   | 교차축 (여러 줄 전체) |
| `justify-items`   | 주축 (Grid 전용)  |

### `align-items` vs `align-content`

`align-items` : **줄마다 따로따로** 교차축 정렬

`align-content` : **전체 묶음 한 번에** 교차축 정렬 (`flex-wrap: wrap`일 때만 의미 있음)

### `align-*`은 세로 정렬이 아님

`flex-direction`에 따라 달라짐.

| `flex-direction` | 교차축 (`align-*`이 담당) |
|------------------|---------------------|
| `row` (기본값)      | 세로 ↕                |
| `column`         | 가로 ↔                |

> `align-*`은 "세로 정렬"이 아니라 정확히는 **"주축의 반대 방향 정렬"**

### `justify-items`

Flexbox에서는 사실상 사용하지 않음. **Grid 전용** 속성.

*** ** * ** ***

block vs inline 기본 흐름
---------------------

`display` 미지정 시 Flexbox와 관계없는 상태. 주축/교차축 개념 자체가 없음.

|   종류   |      기본 동작      |             예시              |
|--------|-----------------|-----------------------------|
| block  | 세로로 쌓임 (한 줄 차지) | `div`, `p`, `form`, `h1`    |
| inline | 가로로 나열          | `span`, `a`, `input`, `img` |

> 세로로 쌓이는 건 "세로가 주축"이 아니라, block 요소의 기본 성질임.

### block에서 쓸 수 있는 정렬

`align-*` 속성은 block에서 아무 효과 없음.

|         상황          |        속성        |
|---------------------|------------------|
| 내부 inline 콘텐츠 가로 정렬 | `text-align`     |
| block 요소 자체를 가로 가운데 | `margin: 0 auto` |
| 세로 정렬               | flex 써야 함        |

### `text-align` vs `align-items`

`text-align` : **요소 내부**의 inline 콘텐츠 정렬

`align-items` / `justify-content` : **자식 요소 자체**를 정렬 (공간이 있어야 함)

`form`은 block 요소라 가로를 꽉 채우므로, 부모 div에 `align-items`를 써도 정렬할 여백이 없음. 이 경우 `text-align: center`를 쓰거나, `form`을 `inline-block`으로 바꾸고 부모를 flex로 만들어야 함.

*** ** * ** ***

Flexbox vs Grid
---------------

| <br /> |    Flexbox     |        Grid        |
|--------|----------------|--------------------|
| 차원     | 1차원            | 2차원                |
| 기준     | 콘텐츠 중심         | 레이아웃 중심            |
| 적합한 상황 | 네비게이션 바, 버튼 묶음 | 전체 페이지 레이아웃, 카드 격자 |

**Flexbox** : 한 번에 한 방향(행 또는 열)만 다룸. 콘텐츠 양에 따라 유동적으로 늘어나거나 줄어드는 구조에 적합.

**Grid** : 행과 열을 동시에 다룸. 틀을 먼저 잡고 거기에 아이템을 배치하는 구조.

### 같이 쓰는 패턴

Grid로 전체 페이지 뼈대를 잡고, 각 영역 안에서 Flexbox로 세부 정렬하는 패턴이 일반적.

```css
/* 큰 틀은 Grid */
.page {
    display: grid;
    grid-template-areas:
        "header"
        "content"
        "footer";
}

/* 세부 정렬은 Flex */
.header {
    display: flex;
    justify-content: space-between;
    align-items: center;
}
```

<br />

