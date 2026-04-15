# TIL - 2026-04-10

## 1. Java 콘솔 게임 - 스토리 리소스 관리와 STORY_GUIDE 동기화

텍스트 파일 기반 스토리 시스템을 운영할 때 **설계 문서(가이드)와 실제 파일 사이의 불일치**가 쉽게 벌어진다.

오늘 발견된 불일치 유형:

| 유형 | 예시 |
|------|------|
| 파일명 불일치 | 가이드: `hidden.txt` → 실제: `heap.txt` |
| 태그 오타 | `[hyejin_disappear]` → `[heyjin_disappear]` |
| 경로 오류 | `resources/story/` → `resource/story/` |
| 없는 파일 참조 | `battle.txt`, `ending.txt` — 실제로는 존재하지 않음 |
| 태그 누락 | 가이드에는 있지만 실제 파일에는 없는 태그 (`[trust_sunhyuk]` 등) |

**교훈**: 가이드를 실제 파일에 맞게 수정하는 게 반대보다 훨씬 안전하다. 가이드가 이상적인 설계를 담고 있더라도 이미 구현된 파일이 진실의 출처(source of truth)가 된다.

---

## 2. Java 콘솔 게임 - StoryManager / GameManager 리팩토링

스토리 시스템 코드(`StoryManager`, `GameManager`)가 실제 리소스 파일과 맞지 않아 한꺼번에 수정했다.

주요 수정 내용:

- `StoryManager.loadAll()` — `hidden.txt`/`"hidden"` prefix를 `heap.txt`/`"heap"`으로 변경, 없는 파일(`ending.txt`, `battle.txt`) 로드 코드 제거
- `GameManager` 경로 — `"resources/story"` → `"resource/story"`
- `GameManager` 태그 호출 정정:
  - `[avalanche]` 호출 제거 (태그 없음)
  - `[collapse]` 호출 전부 제거 (보스 전투 승리 후 층 전환으로 대체)
  - `[trust_sunhyuk]` 분기 제거
- 혜진 루트 분기 — `hyejinRoute` 플래그로 `_with_hyejin` / `_without_hyejin` 태그 선택
- 엔딩 — `ending.txt` 대사 호출 제거, 별도 ending 테이블 조회로 TODO 처리

---

## 3. DB 없이 동작하는 하드코딩 테스트 전략

DB 데이터 생성이 지연될 때 게임 플로우 전체를 검증하기 위해 **코드에서 직접 객체를 생성하는 하드코딩 데이터**로 전체 플레이를 가능하게 만들었다.

```java
// 예: playTutorial() 에서 DB 대신 직접 생성
Hero hero = new Hero("H001", 100, 5, 10, 0, "C001");
Card scannerCard = Card.createAttackCard("C001", 1, "Scanner", "SCAN_INPUT", 10, ...);
NPC hyejin = new NPC("N001", "혜진", "...", 50, false, 8, 12, 1, "C002");
BattleResult result = battleManager.startBattle(hero, hyejin, "혜진", playerCards, playerItems, enemyCard);
```

이 방식의 장점:
- DB 연동 없이도 프롤로그 → 튜토리얼 → 맵 탐색 → 층 이동 → 보스 전투 전체 플로우를 즉시 테스트 가능
- `StageManager`의 맵 이동 로직(`testGameLoop()`)을 `GameView`와 연결해서 게임 루프 검증 가능
- 나중에 DB가 완성되면 하드코딩 부분만 교체하면 됨

**교훈**: 외부 의존성(DB, API)이 없어도 플로우 테스트를 먼저 진행할 수 있도록 하드코딩 더미 데이터 전략을 초기부터 준비해두는 게 개발 속도에 도움이 된다.

---

## 4. Claude Code - 세션 파일 파싱으로 TIL 자동 생성

`~/.claude/projects/` 아래 JSONL 세션 파일을 파싱하여 특정 날짜의 작업 내용을 역추적하고 TIL을 작성할 수 있다.

```python
import json

with open("session.jsonl", "r", encoding="utf-8") as f:
    for line in f:
        obj = json.loads(line.strip())
        role = obj.get("type", "")
        if role in ("user", "assistant"):
            content = obj.get("message", {}).get("content", "")
            if isinstance(content, list):
                for c in content:
                    if c.get("type") == "text":
                        print(f"[{role}]", c["text"])
```

- JSONL 파일의 인코딩은 UTF-8이지만 Windows 터미널에서 출력 시 깨질 수 있으므로, 파일로 먼저 쓰고 읽는 편이 안전하다.
- 수정 날짜 기준으로 파일을 찾으려면 `find -newermt "2026-04-10" ! -newermt "2026-04-11"` 활용.
