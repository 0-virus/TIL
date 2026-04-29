# 2026-04-28

📚 오늘 배운 것: React Hooks 심화
--------------------------

### 1. useReducer

#### 정의

복잡한 상태 로직을 관리할 때 사용하는 hook. `useState`의 대안으로, 상태와 변경 로직을 한 곳에 모아 관리할 수 있다.

#### 기본 문법

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
```

|       항목       |                   설명                    |
|----------------|-----------------------------------------|
| `state`        | 현재 상태값                                  |
| `dispatch`     | 액션을 보내는 함수                              |
| `reducer`      | `(state, action) => newState` 형태의 순수 함수 |
| `initialState` | 초기 상태값                                  |

#### 사용 시점

상태가 객체 형태로 여러 값을 가질 때

다음 상태가 이전 상태에 의존할 때

상태 변경 로직이 복잡할 때

여러 상태가 서로 연관되어 함께 변경될 때
> 💡 단순히 useState가 많다고 useReducer를 쓰는 게 아니라, **상태들이 서로 얽혀있고 변경 로직이 복잡할 때** 쓴다.

*** ** * ** ***

### 2. action과 payload

#### action 객체 구조

```jsx
{
  type: '액션이름',     // 어떤 동작인지 (필수)
  payload: 데이터        // 동작에 필요한 데이터 (선택)
}
```

#### payload란?

action이 reducer에게 전달하는 **데이터**

필수가 아닌 **관례 (Flux Standard Action)**

문자열, 숫자, 객체, 배열 등 어떤 형태든 가능

```jsx
dispatch({ type: 'ADD', payload: '리액트 공부' });
dispatch({ type: 'DELETE', payload: 3 });
dispatch({ type: 'UPDATE', payload: { id: 1, text: '수정' } });
```

*** ** * ** ***

### 3. `...state` 스프레드의 의미

#### 의문점

`ADD_TODO` 케이스에서 모든 속성을 덮어쓰는데, `...state`가 의미 있는가?

```jsx
case "ADD_TODO":
  return {
    ...state,                              // ← 의미 있나?
    todoList: [state.todo, ...state.todoList],
    todo: "",
  };
```

#### 결론

**현재 코드에서는 결과적으로 동일** (의미 없음)

하지만 **써야 하는 이유**가 있음

**미래 확장성** : state에 속성이 추가되면 `...state` 없이는 누락됨

**표준 패턴** : reducer 작성 시 `...state`를 먼저 쓰는 게 관례

**일관성**: 다른 case들과 형태 통일

*** ** * ** ***

### 4. useReducer vs Redux

|    항목    | useReducer |     Redux     |
|----------|------------|---------------|
| 범위       | 컴포넌트 (지역)  | 앱 전체 (전역)     |
| 설치       | React 내장   | 별도 라이브러리      |
| 미들웨어     | ❌          | ✅ thunk, saga |
| DevTools | ❌          | ✅ 시간여행 디버깅    |
| 러닝커브     | 낮음         | 높음            |

**선택 가이드**

컴포넌트 트리 일부 → `useReducer`

앱 일부 영역 공유 → `useReducer + Context`

앱 전체 복잡한 상태 → `Redux Toolkit`

*** ** * ** ***

### 5. useRef

#### 정의

렌더링과 무관하게 값을 기억하거나, DOM 요소에 직접 접근하기 위한 hook.

```jsx
const ref = useRef(initialValue);
// ref.current로 접근
```

#### 핵심 특징

값이 바뀌어도 **리렌더링되지 않음**

리렌더링되어도 값이 **유지됨**

변경이 **즉시 반영됨**

#### 두 가지 주요 용도

**용도 1: DOM 요소 직접 접근**

```jsx
const inputRef = useRef(null);
useEffect(() => inputRef.current.focus(), []);
return <input ref={inputRef} />;
```

**용도 2: 리렌더 없이 값 유지** (타이머 ID, 이전 값 등)

```jsx
const timerId = useRef(null);
timerId.current = setInterval(...);
```

#### useState vs useRef

|   비교    | useState | useRef |
|---------|----------|--------|
| 리렌더링    | ✅        | ❌      |
| DOM 접근  | ❌        | ✅      |
| 화면 표시 값 | 적합       | 부적합    |

*** ** * ** ***

### 6. useContext

#### 정의

props 전달 없이 컴포넌트 트리 전체에서 데이터를 공유하기 위한 hook.

#### 해결하는 문제: Props Drilling

중간 컴포넌트들이 사용하지도 않는 props를 단순히 전달만 하기 위해 받는 문제.

#### 사용 3단계

**1단계: Context 생성**

```jsx
const ThemeContext = createContext('light');
```

**2단계: Provider로 감싸기**

```jsx
<ThemeContext.Provider value={theme}>
  <Page />
</ThemeContext.Provider>
```

**3단계: useContext로 사용**

```jsx
const theme = useContext(ThemeContext);
```

#### useReducer + useContext 조합

작은\~중간 프로젝트에서 Redux 대안으로 활용 가능. 전역 상태 + 복잡한 로직을 깔끔하게 관리.

```jsx
<TodoContext.Provider value={{ state, dispatch }}>
```

*** ** * ** ***

🐛 오늘 만난 에러
-----------

### "Objects are not valid as a React child"

#### 원인

state 구조를 문자열 → 객체로 변경했는데, 렌더링 부분에서 객체를 그대로 출력 시도.

```jsx
// 변경 전
todoList: ["할 일"]

// 변경 후
todoList: [{ todo: "할 일", completed: false }]

// 잘못된 렌더링
<li>{todo}</li>  // ← 객체 자체를 렌더링하려고 함 ❌
```

#### 해결

객체에서 필요한 속성을 꺼내서 렌더링.

```jsx
<li>{item.todo}</li>  // ✅
```

#### 추가로 발견한 문제들

`key` prop 누락 → `id={index}` ❌ → `key={index}` ✅

`e.target.id` 사용 시 button에 id가 없음 + 문자열/숫자 타입 불일치

map 콜백 파라미터명 `todo`가 객체 속성 `todo`와 충돌 → `item`으로 변경

*** ** * ** ***

💡 추가로 적용한 것: 취소선 스타일
---------------------

`completed` 상태에 따라 동적 스타일 적용.

```jsx
<li style={{
  textDecoration: item.completed ? "line-through" : "none"
}}>
```

토글 액션 추가

```jsx
case "TOGGLE_TODO":
  return {
    ...state,
    todoList: state.todoList.map((item, index) =>
      index === action.payload
        ? { ...item, completed: !item.completed }
        : item
    ),
  };
```

*** ** * ** ***

🎯 정리
-----

### Hook 3대장 비교

|     Hook     |      용도      | 리렌더링 |  범위   |
|--------------|--------------|------|-------|
| `useState`   | 단순 상태        | ✅    | 컴포넌트  |
| `useReducer` | 복잡한 상태       | ✅    | 컴포넌트  |
| `useRef`     | DOM 접근, 값 유지 | ❌    | 컴포넌트  |
| `useContext` | 전역 데이터 공유    | ✅    | 트리 전체 |

### 오늘의 깨달음

상태 관리 도구는 **상황에 맞게** 선택해야 한다

`...state`는 당장 의미 없어 보여도 **확장성**을 위한 안전장치

React의 렌더링 규칙: **객체는 직접 렌더링 불가**, 속성을 꺼내거나 배열로 변환해야 함

단순한 hook 사용법보다 **"왜 이걸 쓰는가"**를 이해하는 게 중요하다
