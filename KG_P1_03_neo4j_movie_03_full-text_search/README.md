# KG_P1_03 Movie 03 Full-text Search (섹션 4)

> 강의: 지식그래프(KG) 검색 - 전문 검색 (Full-text Search)

## 이 강의의 학습 목표

이 강의를 마치면 다음을 할 수 있어요.

1. `WHERE title = ...` 같은 기본 검색과 전문 검색이 텍스트를 어떻게 다르게 보는지 설명할 수 있어요.
2. 풀텍스트 인덱스를 만들고, `db.index.fulltext.queryNodes`로 검색해서 관련성 점수와 함께 결과를 받을 수 있어요.
3. 퍼지, 와일드카드, 구문, 논리 연산자, 가중치 검색을 검색어 문법으로 골라 쓸 수 있어요.
4. 전문 검색으로 찾은 노드에서 그래프 탐색으로 이어 가며 관련 정보를 넓힐 수 있어요.

읽기 전에 이 질문 하나를 머리에 두고 보면 좋아요. **"제목에 love가 들어간 영화를 찾으라고 하면, 몇 편이라고 답해야 할까?"**

## 한눈에 보기

지금까지의 Cypher 검색은 속성 값을 **문자열 그대로** 비교했어요. 그래서 대소문자 하나, 철자 하나만 달라도 못 찾았죠. 전문 검색은 텍스트를 먼저 **토큰(단어)** 으로 쪼개서 인덱스에 넣어 두고, 검색어도 똑같이 쪼개서 단어끼리 맞춰 봐요. 강사는 이걸 키워드 검색, 또는 BM25 검색이라고 불렀어요.

```
"Love & Basketball"   →  쪼개고 소문자로  →  [love] [basketball]
검색어 "LOVE"          →  쪼개고 소문자로  →  [love]               → 일치!
```

처음 질문으로 돌아가 볼게요. 실제 데이터에서 세 가지 방법으로 세어 보면 답이 모두 달라요.

| 방법 | 결과 | 왜 |
|---|---|---|
| `WHERE m.title CONTAINS 'love'` | 5편 | 대소문자를 구분해서, 소문자 `love`가 들어간 제목만 찾아요 |
| `WHERE toLower(m.title) CONTAINS 'love'` | 71편 | 글자 조각으로 찾아서 `Glove`, `Lovely`, `Beloved`까지 걸려요 |
| 전문 검색 `love` | 56편 | **단어** `love`가 있는 제목만 찾아요 |

즉 전문 검색은 "대소문자는 무시하되, 단어 단위로" 찾아 줘요. 사람이 "love가 들어간 영화"라고 할 때 기대하는 게 보통 이쪽이죠. 게다가 결과마다 **관련성 점수(score)** 가 붙어서, 더 관련 있는 순서로 정렬하거나 일정 점수 이상만 쓰는 식으로 활용할 수 있어요.

## 1. 풀텍스트 인덱스 만들기

전문 검색은 풀텍스트 인덱스가 있어야만 쓸 수 있어요. 앞에서 만든 제약 조건이나 일반 인덱스와는 별개로, 이름을 붙여서 따로 만들어요.

```cypher
CREATE FULLTEXT INDEX movie_title_fulltext IF NOT EXISTS
FOR (m:Movie) ON EACH [m.title]
```

- **`movie_title_fulltext`**: 인덱스 이름이에요. 검색할 때 이 이름으로 인덱스를 지정해요.
- **`ON EACH [m.title]`**: 리스트라서 속성을 여러 개 넣을 수 있어요. 강의에서는 이 모양으로 인덱스를 세 개 만들었어요.

| 인덱스 | 대상 |
|---|---|
| `movie_title_fulltext` | 영화 제목 |
| `movie_title_tagline_fulltext` | 영화 제목 + 태그라인 (`ON EACH [m.title, m.tagline]`) |
| `person_name_fulltext` | 사람 이름 (배우, 감독). "Leonardo DiCaprio"를 정확히 치기 어려우니까요 |

만든 뒤에는 `SHOW FULLTEXT INDEXES`로 확인해요. 결과의 `fulltext.analyzer`가 `standard-no-stop-words`로 나오는데, 이게 위에서 본 "쪼개고 소문자로 바꾸는" 규칙이에요. 대소문자를 구분하지 않는 이유가 바로 이거예요.

## 2. 기본 검색: queryNodes

검색은 Neo4j에 내장된 프로시저를 `CALL`로 불러서 해요.

```cypher
CALL db.index.fulltext.queryNodes("movie_title_fulltext", $search_term)
YIELD node, score
RETURN node.title AS MovieTitle, score AS SearchRelevance
ORDER BY SearchRelevance DESC
LIMIT 5
```

- **`CALL ... YIELD node, score`**: 인덱스 이름과 검색어를 넘기면, 찾은 노드(`node`)와 관련성 점수(`score`)를 돌려줘요. 그다음부터는 평소처럼 `RETURN`, `ORDER BY`를 쓰면 돼요.
- 검색어는 `$search_term` 파라미터로 넘겨요.

강의 예시를 실제로 돌려 보면 이렇게 나와요.

- **`love`** (제목): Love Stinks, Love & Basketball, Love Happens 등이 나오고, 상위 5편의 점수가 모두 2.282로 같아요. 다섯 편 모두 love가 들어 있고 제목 길이(두 단어)도 같기 때문이에요. BM25는 단어가 드물수록, 그리고 제목이 짧을수록 점수를 높게 줘요.
- **`dream`** (제목 + 태그라인): Opal Dream처럼 제목에 있는 영화뿐 아니라, The Walk(태그라인 "Dream High.")처럼 **태그라인에만** dream이 있는 영화도 찾아요. 제목과 태그라인 둘 다에 있으면(Goal!: The Dream Begins) 점수가 가장 높아요.
- **`tom`** (사람 이름): Tom Noonan, Tom Hulce, Tom Bosley처럼 이름에 Tom이 들어간 사람을 배우, 감독 구분 없이 찾아요.

## 3. 검색어 문법

여기서부터가 이번 강의의 핵심이에요. 퍼지, 와일드카드 같은 기능은 **Cypher를 바꾸는 게 아니라 검색어 문자열만 바꿔서** 써요. 쿼리는 2절 그대로 두고 `$search_term`에 넘기는 값만 달라져요.

| 문법 | 검색어 | 뜻 | 실제 결과 |
|---|---|---|---|
| 그냥 띄어쓰기 | `love story` | love **또는** story (기본이 OR) | A Love Story 두 편에, Toy Story와 Love Actually까지 |
| 구문 | `"love story"` | 이 순서 그대로 붙어 있는 것 | Wristcutters: A Love Story, Capitalism: A Love Story |
| 논리 연산자 | `love AND story NOT horror` | 둘 다 있고 horror는 없는 것 | 위와 같은 두 편 |
| 퍼지 | `afair~0.7` | 철자가 조금 달라도 | Vanity Fair, The Thomas Crown Affair, My Fair Lady 등 |
| 와일드카드 | `star*` | star로 시작하는 단어 | Star Wars, The Fault in Our Stars, 20 Feet from Stardom 등 |
| 가중치 | `love^4 story` | love를 4배 중요하게 | love만 있는 짧은 제목들이 맨 위로 올라와요 |

표에서 첫 줄이 가장 놓치기 쉬워요. 검색어에 단어를 띄어 쓰면 기본이 **OR**라서 생각보다 많이 걸려요. 그래서 붙어 있는 구문을 원하면 따옴표로 감싸고, 둘 다 있어야 하면 `AND`를 써야 해요. 파이썬 문자열 안에 따옴표를 넣어야 하니까 `"\"love story\""`처럼 이스케이프해요.

나머지 문법에서 알아 둘 점은 이 정도예요.

- **퍼지**는 편집 거리, 즉 글자를 몇 번 고치면 같아지는지로 비슷함을 판단해요. `afair`는 a 하나를 지우면 `fair`, f 하나를 넣으면 `affair`가 되니까 둘 다 한 번 고치면 같아져서 걸려요. 강의는 `~0.7`을 "70% 유사도"라고 설명했지만, 실제로는 내부에서 편집 횟수로 바뀌어요. 그래서 `afair~1`로 바꿔 돌려 봐도 결과가 똑같아요. 편집 횟수는 최대 2까지라서 `~1`이나 `~2`로 쓰는 게 더 명확해요.
- **와일드카드** 결과는 점수가 전부 1.0이에요. 그래서 와일드카드 검색에서는 점수로 순위를 매기는 게 의미가 없어요.
- **가중치**는 순위를 크게 흔들어요. `love^4 story`로 검색하면 love만 있는 짧은 제목들(9.1점)이 love와 story가 둘 다 들어간 긴 제목보다 위로 올라가요.

## 4. 전문 검색에서 그래프 탐색으로

전문 검색만 쓰면 일반 검색 엔진과 다를 게 없어요. 그런데 Neo4j에서는 찾은 노드가 곧 그래프의 출발점이 되니까, 거기서 관계를 따라 정보를 넓혀 갈 수 있어요. 강사가 "검색 범위를 확장한다"고 한 게 이 부분이에요.

```mermaid
flowchart LR
    Q["검색어 love"] --> F["전문 검색<br/>movie_title_fulltext"]
    F --> M["찾은 영화들<br/>(movie, score)"]
    M --> G["그래프 탐색<br/>movie ← ACTED_IN ← actor"]
    G --> R["영화 제목 + 점수<br/>+ 배우 목록"]
```

```cypher
CALL db.index.fulltext.queryNodes("movie_title_fulltext", $search_term)
YIELD node AS movie, score
MATCH (movie)<-[:ACTED_IN]-(actor:Person)
RETURN movie.title AS MovieTitle, score AS SearchRelevance,
       collect(actor.name) AS Actors
ORDER BY SearchRelevance DESC
LIMIT 5
```

`YIELD node AS movie`로 이름을 붙여 두면, 그다음 줄부터는 평소 쓰던 `MATCH` 패턴 그대로 이어 쓸 수 있어요. 영화 하나에 배우가 여러 명이니까 `collect`로 영화당 한 행에 배우 리스트로 모아요. `love`로 돌려 보면 Love Stinks, Love & Basketball이 각각 배우 목록과 함께 나와요. 같은 방식으로 `ACTED_IN` 대신 감독이나 장르 관계를 따라가면 "이 영화의 감독", "같은 장르의 다른 영화"로도 넓힐 수 있어요.

노트북에는 한 단계 더 나간 예시도 있어요. 인덱스 두 개를 동시에 써서 "제목에 star가 있는 영화" 중 "이름에 harrison이 있는 배우"가 출연한 영화를 찾는 쿼리예요.

```cypher
CALL () {
    CALL db.index.fulltext.queryNodes("movie_title_fulltext", $movie_term)
    YIELD node AS movie, score AS movieScore
    RETURN movie, movieScore
}
CALL () {
    CALL db.index.fulltext.queryNodes("person_name_fulltext", $actor_term)
    YIELD node AS actor
    RETURN actor
}
MATCH (movie)<-[:ACTED_IN]-(actor)
RETURN movie.title, movieScore, actor.name, movie.rating
```

`CALL () { ... }`는 서브쿼리라서, 영화 검색과 사람 검색을 각각 따로 돌린 다음 마지막 `MATCH`로 둘 사이에 `ACTED_IN` 관계가 있는 쌍만 남겨요. 실제로 돌려 보면 Star Wars와 Harrison Ford 한 쌍만 나와요. `star*`가 아니라 `star`로 검색해서 Stars나 Stardom 같은 단어는 빠졌기 때문이에요.

## 5. 정리하며 기억할 것

- 기본 검색은 문자열을 그대로 비교하고, 전문 검색은 단어로 쪼갠 다음 대소문자 없이 비교해요. 그리고 결과에 관련성 점수가 붙어요.
- 전문 검색은 풀텍스트 인덱스가 있어야 쓸 수 있고, `CALL db.index.fulltext.queryNodes(인덱스, 검색어) YIELD node, score`로 검색해요.
- 퍼지, 와일드카드, 구문, AND/NOT, 가중치는 전부 **검색어 문자열** 쪽 문법이에요. 그리고 띄어쓰기는 기본이 OR예요.
- `YIELD node AS movie`로 받은 노드에서 `MATCH`로 이어 가면, 검색 결과를 그래프 탐색으로 넓힐 수 있어요.

## 다음이랑 어떻게 이어지나

전문 검색은 결국 **같은 단어**가 있어야 찾아요. 그래서 "인공지능 영화"로 검색하면 `artificial intelligence`라고 적힌 영화는 못 찾죠. 다음 [벡터 검색](../KG_P1_03_neo4j_movie_04_vector_search)에서는 텍스트를 임베딩으로 바꿔서 단어가 아니라 **의미**로 찾아요. 그리고 인덱스를 만들고 `CALL db.index.<종류>.queryNodes`로 검색한 뒤 `YIELD node, score`로 받는 모양은 거의 그대로라서, 이번에 익힌 틀이 거기서도 그대로 쓰여요.
