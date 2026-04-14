# TIL - 2026-04-12

## 1. Java 콘솔 게임 - 한국어 조사 처리 시스템 (은/는, 이/가, 을/를)

한국어 게임에서 NPC 이름 뒤에 붙는 조사가 받침 유무에 따라 달라지는 문제를 DB 기반으로 해결했다.

**방식**: NPC 테이블에 `pp` (postposition) 컬럼을 추가하여, 이름이 받침으로 끝나면 `0`, 모음으로 끝나면 `1`로 저장한다.

```java
// pp 값에 따라 조사 분기
String particle = (npc.getPp() == 0) ? "이" : "가";
System.out.println(npc.getName() + particle + " 나타났다!");
// "혜진이 나타났다!" vs "선혁가 → 선혁이 나타났다!"
```

한글 유니코드 연산으로 자동 판별하는 방법도 있지만, DB에 명시적으로 저장하는 방식이 예외 처리(영문 이름, 특수 케이스)에 더 유연하다. 다만 NPC 데이터를 추가할 때마다 pp 값을 직접 세팅해야 하는 번거로움이 있다.

---

## 2. Java 콘솔 게임 - JLine3 도입 시도와 롤백 (화살표 키 입력)

맵 탐색 시 WASD 대신 화살표 키를 쓰기 위해 JLine3 라이브러리를 도입했다가 롤백한 경험.

**문제점**:
- JLine의 Raw 모드가 `System.in` 전체를 가로채서, 기존 `Scanner.nextLine()` 기반 입력 로직이 전부 깨짐
- IntelliJ 터미널이 "dumb terminal"로 인식되어 JLine이 정상 동작하지 않음
- 게임 전체에서 Scanner와 JLine을 혼용하려면 입력 시스템을 처음부터 다시 설계해야 함

**결론**: 화살표 키를 포기하고 WASD + Enter 방식으로 유지. 라이브러리 도입 전에 기존 코드와의 호환성을 먼저 검증해야 한다는 교훈을 얻었다.

---

## 3. Docker - Volume Mount와 데이터 영속성

Docker Compose로 MySQL을 띄우면서 볼륨 마운트의 두 가지 유형을 이해했다.

### Bind Mount (파일/폴더 직접 연결)
```yaml
volumes:
  - ./init.sql:/docker-entrypoint-initdb.d/init.sql
```
호스트의 `init.sql`을 컨테이너 내부 경로에 직접 매핑. MySQL은 `/docker-entrypoint-initdb.d/` 안의 SQL 파일을 **최초 실행 시 자동 실행**한다. 이미 데이터가 있으면(named volume에 기존 DB가 남아있으면) init 스크립트를 건너뛴다.

### Named Volume (Docker 관리 영역)
```yaml
volumes:
  mysql_data:
volumes:
  - mysql_data:/var/lib/mysql
```
컨테이너를 삭제해도 `mysql_data` 볼륨이 남아있으면 DB 데이터가 유지된다. `docker volume rm`으로 볼륨을 명시적으로 삭제해야 완전히 초기화된다.

| 상황 | init.sql 실행 | 기존 데이터 |
|------|:---:|:---:|
| 최초 `docker compose up` | O | 없음 |
| 컨테이너 재시작 (볼륨 유지) | X | 유지 |
| 볼륨 삭제 후 `up` | O | 초기화 |

---

## 4. CMD vs IntelliJ 런타임 차이 (인코딩, 프로세스 실행)

같은 Java 코드가 IntelliJ와 CMD에서 다르게 동작하는 케이스를 여러 개 겪었다.

| 차이점 | IntelliJ | CMD |
|------|------|------|
| 인코딩 | 자동 UTF-8 플래그 | `chcp 65001` + `-Dfile.encoding=UTF-8` 필요 |
| 화면 지우기 | `ProcessBuilder("cmd", "/c", "cls")` + `inheritIO()` 작동 | `inheritIO()` 사용 시 stdin을 소비하여 후속 입력이 깨짐 |
| 배치 파일 줄바꿈 | LF/CRLF 모두 허용 | CRLF만 정상 동작 |

**해결**: `run.bat`에서 `chcp 65001`로 인코딩 설정, `javac`에 `-encoding UTF-8` 플래그 추가, 화면 지우기에서 `inheritIO()` 제거.

---

## 5. Java 콘솔 게임 - 이벤트 선행 조건(Prerequisite) 시스템

특정 이벤트가 다른 이벤트를 먼저 완료해야 발동되는 구조를 `GameManager`의 상태 플래그로 구현했다.

```java
// 배신 이벤트는 의심 이벤트 2개를 모두 본 뒤에만 발동
if (suspicionCount >= 2) {
    triggerBetrayalEvent();
}
```

DB에 선행 조건 테이블을 만드는 방법도 있지만, 이벤트 수가 적은 이 프로젝트에서는 코드 내 플래그로 충분했다. 이벤트가 많아지면 `event_prerequisite` 테이블을 도입하는 게 유지보수에 유리할 것이다.

---

## 6. Java 콘솔 게임 - 최종 보스 설계 (물리적으로 잡을 수 없는 적)

최종 보스 "딸깍"(AI 서비스 의인화)은 일반 공격으로 절대 죽지 않도록 설계했다.

- HP가 1 아래로 내려가지 않음 (히든 맵의 GC 적과 유사한 메커니즘)
- 유일한 처치 방법: 히든 맵에서 획득한 `}` (닫는 중괄호) 카드를 전투 중 사용
- 이 카드가 없으면 도망만 가능 → 히든 맵 탐색을 유도하는 게임 디자인

**구현 핵심**: `BattleManager`에서 특정 카드 사용 시 보스 HP를 0으로 강제 세팅하는 분기를 추가. 일반적인 데미지 계산 로직과 별도로 "특수 승리 조건"을 처리하는 패턴.

---

## 7. 발표 스크립트 작성 - 코드 기반 역추적으로 시행착오 정리

PPT 발표 스크립트를 작성할 때, 코드베이스를 직접 탐색하며 시행착오 사례를 역추적했다.

정리된 시행착오:
1. **화살표 키 입력 실패**: JLine Raw 모드가 System.in을 가로챔 → WASD로 롤백
2. **튜토리얼 저장 버그**: `createNewSave(2)` → `createNewSave(1)` 수정 (스테이지 2에서 시작되는 버그)
3. **init.sql의 `w`/`v` 값 반전**: 벽(wall)과 빈 공간(void)의 타일 마커가 반대로 들어가 있었음

발표는 "무엇을 만들었나"보다 "무엇이 안 됐고 어떻게 해결했나"가 더 설득력이 있다. 코드의 git history와 실제 구현을 비교하면 시행착오를 체계적으로 복원할 수 있다.

---

## 8. Docker Compose - MySQL 컨테이너 설정 패턴

팀 프로젝트 배포용 `docker-compose.yml` 설정 패턴을 정리했다.

```yaml
services:
  mysql:
    image: mysql:8.0
    ports:
      - "3307:3306"          # 호스트 3307 → 컨테이너 3306 (로컬 MySQL과 충돌 방지)
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: game_db
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql   # 자동 초기화
      - mysql_data:/var/lib/mysql                          # 데이터 영속화

volumes:
  mysql_data:
```

**포트를 3307로 매핑한 이유**: 팀원의 로컬 머신에 이미 MySQL이 3306에서 돌고 있을 수 있으므로 충돌을 피하기 위함. JDBC URL도 `localhost:3307`로 맞춰야 한다.

---

## 9. Java - Build Tool 없이 수동 컴파일/실행 (javac + batch)

Maven/Gradle 없이 `javac`로 직접 컴파일하는 배치 파일 구성.

```bat
@echo off
chcp 65001 > nul
javac -encoding UTF-8 -cp "lib/mysql-connector-j-9.2.0.jar" -d out src/*.java
java -Dfile.encoding=UTF-8 -cp "out;lib/mysql-connector-j-9.2.0.jar" Main
```

- `chcp 65001`: CMD 코드페이지를 UTF-8로 변경
- `-encoding UTF-8`: 소스 파일 인코딩 지정 (한글 리터럴 깨짐 방지)
- `-cp "out;lib/..."`: 컴파일된 클래스와 외부 jar를 클래스패스에 포함 (Windows는 `;` 구분자)

빌드 도구 없이도 동작하지만, 소스 파일이 30개가 넘어가면 `sources.txt`에 파일 목록을 관리하는 게 필요해진다. 프로젝트 규모가 커지면 빌드 도구 도입이 필수적이다.
