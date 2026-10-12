# KG_P1_03 Movie 02 Basic Search (섹션 4)

> 강의: 지식그래프(KG) 검색 - 기본 검색 (Cypher 쿼리 분석)

## 이 강의의 학습 목표

이 강의를 마치면 다음을 할 수 있어요.

1. `MATCH → WHERE → RETURN → ORDER BY → LIMIT` 흐름으로 영화 데이터를 조회하고, 결과를 pandas DataFrame으로 바꿀 수 있어요.
2. 관계 패턴과 `count()`로 "배우별 출연 수", "장르별 영화 수" 같은 집계를 할 수 있어요. 그리고 SQL과 달리 `GROUP BY`가 없는데도 어떻게 묶이는지 설명할 수 있어요.
3. 영화를 가운데 두고 양쪽에서 들어오는 V자 패턴으로 "함께 출연한 배우", "같이 일한 감독"을 찾을 수 있어요.
4. `OPTIONAL MATCH`, `WITH`, `collect()`, `size()`를 엮어서 장르와 배우에 가중치를 준 추천 쿼리를 읽고 고칠 수 있어요.

읽기 전에 이 질문 하나를 머리에 두고 보면 좋아요. **"Apollo 13을 본 사람에게 '평점 높은 드라마'를 추천하면, 그게 Apollo 13과 관련 있는 추천일까?"**

## 한눈에 보기

이번 강의는 앞에서 CSV로 만든 영화 그래프(영화 4,803편, 노드 약 1.6만 개, 관계 약 4만 개)를 Cypher로 꺼내 보는 시간이에요. 쿼리는 갈수록 길어지지만 결국 두 동사만 알면 읽을 수 있어요.

- **`MATCH`**: 그래프에서 이 **모양(패턴)** 을 찾아라.
- **`WITH`**: 여기까지 **정리해서** 다음 단계로 넘겨라.

강의 흐름은 단순한 조회에서 시작해서 추천까지 점점 쌓아 올리는 구조예요.

| 단계 | 예시 | 새로 배우는 것 |
|---|---|---|
| 1. 기본 조회 | 평점 상위 10편 | `WHERE`, `ORDER BY`, `LIMIT`, 별칭(`AS`) |
| 2. 집계 | 배우별 출연 수, 장르별 영화 수 | 관계 패턴 + `count()` |
| 3. 관계 분석 | Tom Hanks와 함께 출연한 배우, DiCaprio와 일한 감독 | V자 패턴, 파라미터 `$` |
| 4. 추천 | Apollo 13과 비슷한 영화 | `collect()`, `OPTIONAL MATCH`, `WITH`, 가중치 점수 |

## 1. 기본 조회와 DataFrame

```cypher
MATCH (m:Movie)
WHERE m.rating IS NOT NULL
RETURN m.title AS Movie, m.released AS Released, m.rating AS Rating
ORDER BY m.rating DESC
LIMIT 10
```

읽는 순서대로 풀면 "영화를 찾아서(m), 평점이 있는 것만 남기고, 제목, 개봉일, 평점을 돌려주는데, 평점 높은 순으로 10개만"이에요.

여기서 `AS`로 붙인 별칭이 그대로 결과의 키가 돼요. 그래서 `graph.query()`의 결과는 `[{'Movie': ..., 'Released': ..., 'Rating': ...}, ...]` 같은 딕셔너리 리스트로 나오고, `pd.DataFrame(result)` 한 줄이면 바로 표가 돼요. 이걸 CSV나 엑셀로 저장하거나 RDB에 넣는 것도 쉬워요.

같은 쿼리를 Neo4j 브라우저 입력창에 붙여 넣어도 바로 실행돼요. 다만 브라우저는 그래프를 기본 25개, 최대 300개까지만 그려 주니까, 데이터를 전부 다루거나 코드로 후처리할 때는 Python SDK가 맞아요. 둘 다 쓸 줄 알아 두면 좋아요.

## 2. 관계 패턴으로 집계하기

```cypher
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
RETURN p.name AS Actor, count(m) AS MovieCount
ORDER BY MovieCount DESC
LIMIT 10
```

이번에는 `MATCH`에 관계까지 넣었어요. "사람이 영화에 출연했다"는 모양을 전부 찾고, 그걸 배우별로 세는 거예요. 결과는 Robert De Niro 54편, Samuel L. Jackson 44편, Bruce Willis 39편 순이에요.

그런데 SQL이라면 `GROUP BY p.name`이 있어야 할 텐데 여기에는 없죠. Cypher는 **`RETURN`에서 집계 함수가 아닌 컬럼(`p.name`)이 자동으로 묶는 기준**이 돼요. 그래서 `p.name`별로 묶이고, 묶음마다 `count(m)`을 세는 거예요.

장르별 영화 수도 같은 모양이에요. 다만 화살표 방향에 주의해야 해요. 관계는 `(m:Movie)-[:IN_GENRE]->(g:Genre)`처럼 영화에서 장르로 향하고, 방향을 반대로 쓰면 다른 관계로 봐서 아무것도 안 나와요. 결과는 Drama 2,297편이 가장 많고, TV Movie가 8편으로 가장 적어요.

## 3. V자 패턴: 영화를 가운데 두고 찾기

"Tom Hanks와 함께 출연한 배우"를 찾으려면, 같은 영화에 양쪽에서 `ACTED_IN`으로 들어오는 모양을 그리면 돼요.

```
(Tom Hanks) ─ACTED_IN→ (영화) ←ACTED_IN─ (co_actor)
   고정                   가운데             찾을 대상
```

```cypher
MATCH (actor:Person {name: $actor_name})-[:ACTED_IN]->(m:Movie)<-[:ACTED_IN]-(co_actor:Person)
WHERE actor <> co_actor
RETURN co_actor.name AS CoActor, count(m) AS MoviesTogether
ORDER BY MoviesTogether DESC
LIMIT 5
```

- **`$actor_name`**: 파라미터예요. 쿼리는 그대로 두고 `params={"actor_name": "Tom Hanks"}`로 값만 바꿔 넣어요. 결과는 Tim Allen이 3편으로 1위예요(Toy Story 시리즈).
- **`WHERE actor <> co_actor`**: 강의에서는 자기 자신을 빼려고 넣었어요. 그런데 실제로 이 조건을 빼고 돌려 봐도 자기 자신이 나오는 행은 0개예요. Cypher는 한 패턴 안에서 **같은 관계를 두 번 지나가지 않기** 때문에, 이미 쓴 `ACTED_IN` 관계로 되돌아와 자기 자신에 닿을 수 없거든요. 그래도 안전장치로 두는 건 괜찮아요.

"배우와 같이 일한 감독"도 같은 V자예요. 한쪽 화살표만 `DIRECTED`로 바꾸면 돼요.

```cypher
MATCH (a:Person)-[:ACTED_IN]->(m:Movie)<-[:DIRECTED]-(d:Person)
WHERE a.name = $actor_name
RETURN d.name AS Director, count(m) AS CollaborationCount
```

Leonardo DiCaprio로 돌리면 Martin Scorsese가 5편으로 1위, Baz Luhrmann이 2편으로 그다음이에요. 같은 Person 레이블이라도 어떤 관계로 영화에 연결됐는지에 따라 배우도 되고 감독도 되는 거예요.

## 4. 그래프 기반 추천

### 4-1. 장르만 보는 추천

```cypher
MATCH (m:Movie {title: $movie_title})-[:IN_GENRE]->(g:Genre)<-[:IN_GENRE]-(rec:Movie)
WHERE m <> rec AND rec.rating > 7.0
RETURN rec.title AS RecommendedMovie, rec.rating AS Rating,
       collect(g.name) AS SharedGenres
ORDER BY Rating DESC, size(SharedGenres) DESC
LIMIT 5
```

이것도 V자 패턴이에요. 가운데에 영화 대신 장르가 있을 뿐이죠. 영화 하나가 장르를 여러 개 공유할 수 있으니까 `collect(g.name)`으로 공유 장르를 리스트로 모아요. `collect`는 여러 행의 값을 리스트 하나로 모으는 집계 함수예요.

그런데 처음 질문으로 돌아가 보면, 이 추천에는 문제가 있어요. Apollo 13의 장르는 **Drama 하나뿐**이라서, 후보 526편이 전부 "공유 장르 1개"로 똑같아요. 그래서 결국 평점순으로만 정렬되고, 결과는 Me You and Five Bucks, Dancer, Texas Pop. 81처럼 평점 10점인 드라마들이 나와요. 즉 Apollo 13과 별 상관없는 "평점 높은 드라마 목록"이 되는 거죠. 그래서 다음 복합 추천에서는 배우까지 같이 봐요.

### 4-2. 장르와 배우에 가중치를 준 복합 추천

이번 강의에서 가장 긴 쿼리예요. 단계별로 번호를 붙여 읽으면 쉬워요.

```cypher
// ① 기준 영화
MATCH (m:Movie {title: $movie_title})

// ② 후보: 평점 7 넘는 다른 영화 전부
MATCH (rec:Movie) WHERE m <> rec AND rec.rating > 7.0

// ③-1 공통 장르 (없어도 후보는 남김)
OPTIONAL MATCH (m)-[:IN_GENRE]->(g:Genre)<-[:IN_GENRE]-(rec)
WITH m, rec, COLLECT(g.name) AS genres

// ③-2 공통 배우 (없어도 후보는 남김)
OPTIONAL MATCH (m)<-[:ACTED_IN]-(a:Person)-[:ACTED_IN]->(rec)
WITH m, rec, genres, COLLECT(a.name) AS actors

// ④ 점수 매기고 0점 빼기
WITH rec, size(genres) * 2 + size(actors) * 3 AS score, genres, actors
WHERE score > 0

RETURN rec.title, rec.rating, score, genres, actors
ORDER BY score DESC, rec.rating DESC
LIMIT 5
```

```mermaid
flowchart LR
    A["① Apollo 13"] --> B["② 후보<br/>평점 7 초과 808편"]
    B --> C["③-1 공통 장르<br/>collect → genres"]
    C --> D["③-2 공통 배우<br/>collect → actors"]
    D --> E["④ 장르×2 + 배우×3<br/>0점 제외 534편"]
    E --> F["점수순 5편"]
```

③-1을 말로 풀면 이래요. "Apollo 13이 속한 장르 g가 있는데, 후보 rec도 그 g에 속하니? 속하면 g를 가져오고, 아니면 g는 비워 둬. 그래도 rec은 버리지 마." 이 마지막 부분이 `OPTIONAL MATCH`의 역할이에요. 그냥 `MATCH`는 패턴이 없으면 그 행을 아예 지워 버리지만, `OPTIONAL MATCH`는 null로 채워서 다음 단계로 넘겨 줘요.

그래서 이 차이가 결과를 크게 바꿔요. 두 곳을 모두 그냥 `MATCH`로 쓰면 **장르도 같고 배우도 겹치는** 영화만 남아서 후보가 23편으로 줄어요. 반면 `OPTIONAL MATCH`로 쓰면 장르만 같거나 배우만 겹치는 영화도 살아남아서 534편이 점수 경쟁을 해요.

그리고 중간중간 들어간 `WITH`가 "여기까지 정리해서 넘겨"예요. `WITH m, rec, COLLECT(g.name) AS genres`에서 `COLLECT`는 집계 함수라서, 2절에서 본 것처럼 `m, rec`이 묶는 기준이 돼요. 즉 후보 영화 한 편당 한 행으로 정리되고, 공유 장르는 리스트로 모여요. 이렇게 한 번 정리해 두지 않으면 장르 행과 배우 행이 곱해져서 개수가 부풀려져요. 덧붙이면 `WITH`에 적지 않은 변수는 다음 단계로 넘어가지 않아요. 그래서 `g`, `a`는 여기서 사라지고 `genres`, `actors`만 남아요.

④에서는 `size()`로 리스트 길이를 세서 점수를 매겨요. 배우 겹침에 3점, 장르 겹침에 2점을 줬는데, 이 가중치는 정답이 있는 게 아니라 직접 바꿔 보면서 정하는 값이에요. 마지막으로 아무것도 안 겹치는 0점짜리는 뺐어요.

결과는 이렇게 나와요.

| 영화 | 평점 | 점수 | 공통 장르 | 공통 배우 |
|---|---|---|---|---|
| Forrest Gump | 8.2 | 8 | Drama | Tom Hanks, Gary Sinise |
| The Green Mile | 8.2 | 5 | Drama | Tom Hanks |
| Saving Private Ryan | 7.9 | 5 | Drama | Tom Hanks |
| The Truman Show | 7.8 | 5 | Drama | Ed Harris |
| A Beautiful Mind | 7.7 | 5 | Drama | Ed Harris |

4-1과 비교하면 확실히 "Apollo 13을 본 사람"을 위한 추천처럼 보여요. 같은 배우가 나온다는 **관계**를 점수에 넣었기 때문이에요.

참고로 집계 함수와 키워드는 대소문자를 구분하지 않아요. 그래서 `COLLECT`와 `collect`는 같아요. 노트북에 둘 다 섞여 있는데, 실제로 쓸 때는 한 가지로 통일하는 게 좋아요.

## 5. 정리하며 기억할 것

- `MATCH`는 모양을 찾고, `WITH`는 정리해서 넘겨요. 긴 쿼리도 이 두 동사 단위로 끊어 읽으면 돼요.
- 집계 함수를 쓰면 `RETURN`이나 `WITH`에 함께 적은 나머지 컬럼이 자동으로 묶는 기준이 돼요. 그래서 `GROUP BY`가 없어요.
- 영화나 장르를 가운데 두는 V자 패턴 하나로 공동 출연, 협업, 같은 장르 찾기를 다 할 수 있어요.
- 없을 수도 있는 관계를 찾을 때는 `OPTIONAL MATCH`를 써야 후보가 사라지지 않아요.
- 추천 점수는 관계를 몇 개 공유하는지 `size()`로 세고 가중치를 곱해서 만들어요. 그리고 공유하는 게 하나뿐이면 점수가 다 같아져서 추천이 무의미해져요.

## 다음이랑 어떻게 이어지나

여기까지는 `{title: 'Apollo 13'}`처럼 속성 값이 **정확히 일치**해야 찾을 수 있었어요. 그래서 대소문자나 철자가 하나만 틀려도 못 찾죠. 다음 [전문 검색](../KG_P1_03_neo4j_movie_03_full-text_search)에서는 텍스트를 단어로 쪼개서 대소문자 없이, 오타가 있어도 찾는 방법을 배워요. 그리고 그렇게 찾은 영화에서 이번에 배운 관계 패턴으로 이어 가는 것까지 해요.
