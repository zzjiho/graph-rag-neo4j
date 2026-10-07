# KG_P1_01 Neo4j Introduction (섹션 1, 2)

## 키워드

- **AuraDB** : Neo4j 클라우드 버전 (완전 관리형). 무료는 노드 5만, 관계 17.5만까지
- **uv** : 파이썬 패키지, 가상환경 관리 도구 (pip보다 빠름). `uv.lock`이 깔린 버전을 고정
- **Cypher** : Neo4j한테 일 시키는 언어 (그래프 DB의 SQL)
- **노드** `( )` : 데이터 하나 (점)
- **레이블** `:Person` : 노드의 종류
- **속성** `{name: "홍길동"}` : 노드나 관계에 붙은 값
- **관계** `-[:KNOWS]->` : 노드끼리 잇는 화살표
- **변수** `p`, `r` : 쿼리 안에서 부르는 이름
- **MATCH / WHERE / RETURN** : 찾고, 거르고, 돌려주기
- **CREATE / MERGE** : 무조건 만들기 / 있으면 찾고 없으면 만들기
- **SET / REMOVE** : 속성 바꾸기 / 속성이나 레이블 떼기
- **DELETE / DETACH DELETE** : 지우기 / 관계까지 같이 지우기
- **ORDER BY / LIMIT / SKIP** : 정렬, 개수 제한, 건너뛰기

## 한 줄로 남는 것

그래프 DB는 데이터를 점(노드)이랑 화살표(관계)로 저장하고, Cypher는 그 점과 화살표를 그림 그리듯 글자로 적어서 찾고 고치는 언어다.

## 괄호 하나가 노드 하나

```
(p:Person {name: "홍길동", age: 30})
```

| 부분 | 이름 | 뜻 |
|---|---|---|
| `p` | 변수 | 이 쿼리 안에서만 쓰는 이름표 |
| `:Person` | 레이블 | 노드의 종류 (사람) |
| `{name: "홍길동", age: 30}` | 속성 | 그 노드가 가진 값 |

연락처로 치면 연락처 카드 한 장이 노드, "가족"이나 "회사" 같은 그룹이 레이블, 카드에 적힌 이름이랑 나이가 속성이다. 노드가 속성을 **가지고** 있는 거지, 노드가 속성인 건 아니다. "홍길동"은 노드에 적힌 이름 값이고, 노드는 그 사람 전체다.

RDB(MySQL, PostgreSQL)랑 다른 점 두 개.

- 노드 하나에 레이블을 여러 개 붙일 수 있다 (`:Person:Employee`). RDB는 한 행이 테이블 하나에만 속해서, 직원이기도 하면 `employee` 테이블에 행을 하나 더 만들고 외래 키(FK)로 이어야 한다.
- 스키마가 미리 정해져 있지 않아서, 같은 레이블이어도 노드마다 속성이 달라도 된다. RDB는 테이블 스키마로 컬럼을 미리 정해 두고, 값이 없으면 NULL로 둔다.

```cypher
(:Person {name: "홍길동", age: 30, email: "hong@example.com"})
(:Person {name: "김철수", age: 28})   // email이 없으면 그냥 안 적는다
```

Neo4j Browser 왼쪽 "Nodes (4)"에 뜨는 `City`, `Person`은 노드가 아니라 **레이블 목록**이다. 4는 노드 총 개수. `Person`을 누르면 `MATCH (n:Person) RETURN n LIMIT 25`가 돌아서 실제 노드들이 동그라미로 그려진다.

## 관계는 방향이 있다

```
(홍길동) ──KNOWS──▶ (김철수)
 노드      관계       노드
```

- **만들 때는 방향 필수.** CREATE에서 방향 없는 관계는 안 된다.
- **찾을 때는 방향 무시 가능.** `-[:KNOWS]-`처럼 화살표 없이 쓰면 양쪽 다 찾는다.
- 관계 타입은 대문자로 쓰는 게 관례 (`KNOWS`, `LIVES_IN`).
- `-[r:FRIENDS|KNOWS]->`는 둘 중 아무거나 (OR).

## 주요 구문 = SQL의 CRUD

| 구문 | 하는 일 | SQL로 치면 |
|---|---|---|
| `MATCH` + `WHERE` + `RETURN` | 패턴으로 찾아서 거르고 돌려주기 | `SELECT ... WHERE` |
| `CREATE` | 무조건 새로 만들기 | `INSERT` |
| `MERGE` | 있으면 찾고 없으면 만들기 | upsert |
| `SET` | 속성 바꾸기, 추가 | `UPDATE` |
| `DELETE` / `DETACH DELETE` | 지우기 | `DELETE` |
| `REMOVE` | 속성이나 레이블 떼기 | `UPDATE ... SET col = NULL` |

```cypher
MATCH (p:Person)-[:KNOWS]->(f:Person)
WHERE p.name = "홍길동" AND f.age >= 30
RETURN f.name
```

## CREATE는 중복을 만든다

CREATE는 같은 홍길동이 이미 있어도 또 만든다. 노트북 주석에도 "동일한 노드가 이미 존재해도 중복 생성되므로 주의"라고 나와 있다.

그래서 실무에서는 MERGE를 훨씬 많이 쓴다. `ON CREATE SET`을 붙이면 새로 만들어질 때만 속성을 넣는다.

```cypher
MERGE (p:Person {name: "홍길동"})
ON CREATE SET p.age = 30
```

## DELETE 함정

관계가 붙어 있는 노드를 DELETE로 지우면 에러가 난다. 관계부터 지우거나 DETACH DELETE로 한 번에 지운다.

```cypher
MATCH (p:Person {name: "홍길동"})
DETACH DELETE p
```

## 다음이랑 어떻게 이어지나

여기서는 관계를 1홉만 건너는 패턴만 다뤘다. 다음(KG_P1_02)은 여러 홉을 건너가는 가변 길이 경로(`*1..3`), 최단 경로, 집계, WITH다. "친구의 친구"처럼 몇 단계 떨어진 관계를 찾는 게 그래프 DB를 쓰는 진짜 이유라, 여기서 배운 MATCH 패턴이 거기서 길어진다.
