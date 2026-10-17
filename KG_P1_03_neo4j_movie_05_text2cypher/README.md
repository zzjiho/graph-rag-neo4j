# KG_P1_03 Movie 05 Text2Cypher (섹션 4)

> 강의: 지식그래프(KG) 검색 - Text2Cypher 기법

## 이 강의의 학습 목표

이 강의를 마치면 다음을 할 수 있어요.

1. Text2Cypher가 무엇인지, 그리고 `GraphCypherQAChain`이 질문을 받아 답을 내기까지 어떤 단계를 거치는지 설명할 수 있어요.
2. LLM이 Cypher를 잘 짜도록 그래프 스키마를 준비할 수 있어요 (`enhanced_schema`, `refresh_schema`).
3. 체인 옵션(`verbose`, `top_k`, `return_intermediate_steps`, `return_direct`, `cypher_llm`과 `qa_llm`)을 목적에 맞게 고를 수 있어요.
4. AI가 짠 쿼리가 왜 실패했는지 읽어 내고, 커스텀 프롬프트(few-shot 예시)로 고칠 수 있어요.

읽기 전에 이 질문 하나를 머리에 두고 보면 좋아요. **"질문은 사람이 쓰는데, 쿼리는 누가 쓰고 데이터는 누가 꺼낼까?"**

## 한눈에 보기

Cypher를 능숙하게 짜는 건 전문 개발자가 아니면 쉽지 않아요. 그래서 사람은 평소 말로 질문하고, LLM이 그 질문을 Cypher로 바꿔 주게 하는 기법이 Text2Cypher예요. SQL 쪽에서 Text-to-SQL이라고 부르는 것과 같은 아이디어죠. 즉 텍스트로 질문하면, 지식 그래프를 쿼리한 결과로 대답을 듣는 지식 그래프 기반 RAG 시스템이에요.

LangChain에서는 `GraphCypherQAChain`이 이 과정을 한 번에 엮어 줘요. 다만 안을 들여다보면 역할이 넷으로 나뉘어 있어요.

```
① 사람        "Tom Hanks가 출연한 영화 알려줘"        질문은 사람이 써요
② LLM (1번)   질문 + 그래프 스키마 → Cypher 작성     쿼리는 LLM이 써요
③ Neo4j       Cypher 실행 → 결과 행                 데이터는 DB가 꺼내요 (LLM 아님)
④ LLM (2번)   질문 + 결과 행 → 자연어 답변           답변은 다시 LLM이 써요
```

그러니까 처음 질문의 답은 "쿼리는 LLM, 데이터는 Neo4j"예요. LLM은 두 번 일하고, 그 사이에 DB가 한 번 일하는 구조죠.

## 1. 준비: LLM에게 스키마를 보여 준다

LLM은 우리 DB가 어떻게 생겼는지 모르기 때문에, 그래프 스키마(노드 레이블, 속성, 관계)를 프롬프트에 함께 넣어 줘야 해요. 체인이 이걸 알아서 넣어 주는데, 그 전에 두 가지를 챙겨야 해요.

```python
graph = Neo4jGraph(
    url=os.getenv("NEO4J_URI"),
    username=os.getenv("NEO4J_USERNAME"),
    password=os.getenv("NEO4J_PASSWORD"),
    enhanced_schema=True,   # 속성의 예시 값까지 스키마에 담아요
)
graph.refresh_schema()      # 체인을 만들기 전에 최신 스키마로
print(graph.schema)
```

- **`enhanced_schema=True`**: 속성 이름뿐 아니라 예시 값까지 스키마에 들어가요. 예를 들어 `released`의 예시가 `'1927-01-10'`으로 보이면, LLM은 "개봉일이 날짜 문자열이구나" 하고 쿼리를 짤 단서를 얻어요.
- **`refresh_schema()`**: 데이터 구조를 바꾼 뒤에는 꼭 다시 불러야 해요. 그렇지 않으면 LLM은 옛날 구조를 보고 쿼리를 짜요.

## 2. 체인 만들고 질문하기

```python
llm = ChatOpenAI(model="gpt-4.1-mini", temperature=0.0)

cypher_chain = GraphCypherQAChain.from_llm(
    llm=llm,
    graph=graph,
    allow_dangerous_requests=True,
    verbose=True,
)
answer = cypher_chain.invoke({"query": "영화 'Apollo 13'에 대한 정보를 알려주세요."})
print(answer["result"])     # answer는 {"query": 질문, "result": 답변}
```

- **`temperature=0.0`**: 쿼리를 짤 때는 창의성보다 정확성이 중요하니까 0으로 둬요.
- **`allow_dangerous_requests=True`**: "LLM이 짠 쿼리를 검사 없이 내 DB에서 실행해도 된다"는 동의예요. 켜지 않으면 체인이 아예 만들어지지 않아요. 위험하다고 하는 이유는 LLM이 쿼리를 잘못 짜거나(틀린 답), 질문에 따라 `DELETE` 같은 쿼리를 짜거나(데이터 손상), 홉 제한 없는 무거운 쿼리를 짤 수(DB 멈춤) 있기 때문이에요. 혼자 실습할 때는 괜찮지만, 남이 질문을 입력하는 서비스라면 읽기 전용 계정으로 연결하는 식으로 막아야 해요.
- **`verbose=True`**: LLM이 짠 Cypher와 DB 검색 결과(Full Context)를 화면에 찍어 줘요. 개발하는 동안에는 항상 켜 두고 쿼리를 직접 읽어 보는 습관을 들이세요.

## 3. 옵션 고르기

| 옵션 | 건드리는 단계 | 이럴 때 써요 |
|---|---|---|
| `top_k=5` | ③ → ④ | LLM에게 넘기는 결과 수를 줄이고 싶을 때 |
| `return_intermediate_steps=True` | ②③ | 생성된 쿼리와 검색 결과를 반환값(`intermediate_steps`)으로 받아 후속 처리나 디버깅에 쓰고 싶을 때 |
| `return_direct=True` | ④ 생략 | 답변 문장 대신 DB 결과 그대로가 필요할 때 |
| `cypher_llm=`, `qa_llm=` | ②, ④ | 단계마다 다른 모델을 쓰고 싶을 때 |

몇 가지는 오해하기 쉬워서 짚고 넘어갈게요.

- **`top_k`는 Cypher에 `LIMIT`을 붙이지 않아요.** DB에서는 결과를 전부 가져오고, 그중 앞의 k개만 ④ 단계 LLM에게 넘겨요. 그래서 DB 부담을 줄이려면 쿼리 자체에 `LIMIT`이 들어가야 해요.
- **`return_direct`** 결과는 행 목록이라 표로 바꾸기 좋아요. 예를 들어 쿼리가 `RETURN m`이었다면 `pd.DataFrame([row["m"] for row in answer["result"]])`로 바로 DataFrame이 돼요.
- **`cypher_llm`과 `qa_llm`**을 나누면 쿼리는 GPT-4.1 mini가, 답변은 Gemini 2.0 Flash가 쓰는 식으로 섞을 수 있어요. 하나만 쓸 때는 `llm=` 하나로 두 역할을 다 맡기면 돼요.

## 4. 실패 사례에서 배우기 (이 강의의 핵심)

강의에서 "2010년 이후 개봉한 Drama 영화 중 평점 높은 5개"를 물었더니, 쿼리 모양은 그럴듯했는데 결과가 0개였고 답변도 "정보가 없다"였어요.

원인은 데이터 형식이에요. `released`는 연도 숫자가 아니라 `'2010-07-16'` 같은 **날짜 문자열**인데, LLM은 이걸 숫자처럼 비교했어요. 반대로 문자열 그대로 비교하면 잘 찾아져요. ISO 날짜 문자열은 문자열로 비교해도 순서가 맞거든요.

```cypher
MATCH (m:Movie)-[:IN_GENRE]->(g:Genre {name: 'Drama'})
WHERE m.released >= '2010-01-01'
RETURN m.title, m.rating
ORDER BY m.rating DESC
LIMIT 5
```

강의가 제시한 해결 방법은 세 가지예요.

1. 스키마를 더 명확하게 준다 (`enhanced_schema`처럼 값 예시까지)
2. few-shot 예시로 올바른 패턴을 보여 준다
3. 더 좋은 모델을 쓴다

이 중 가장 효과가 좋은 건 2번이고, 다음 절에서 다뤄요.

그리고 실패는 아니지만 같이 봐야 할 문제가 하나 더 있어요. LLM이 `RETURN m`처럼 노드를 통째로 돌려주면 줄거리(`overview`)나 임베딩까지 ④ 단계로 넘어가요. 그런데 임베딩은 숫자 덩어리라 답변에는 쓸모가 없고, 토큰만 많이 잡아먹고, 오히려 답변을 방해할 수 있어요. 그래서 필요한 속성만 `RETURN`하도록 예시를 주는 게 좋아요.

## 5. 커스텀 프롬프트 (few-shot)

프롬프트는 단계마다 하나씩, 두 개를 직접 써서 넘길 수 있어요.

- **Cypher 생성 프롬프트** (`{schema}`, `{question}`): 규칙("스키마에 있는 레이블과 속성만 써라", "쿼리만 출력해라")과 예시 쿼리를 담아요.
- **QA 프롬프트** (`{question}`, `{context}`): 전형적인 RAG 템플릿이에요. 검색 결과(context)를 근거로 답하라는 지시와 한국어 답변 형식을 담아요.

```python
cypher_chain = GraphCypherQAChain.from_llm(
    cypher_llm=cypher_llm,
    qa_llm=qa_llm,
    graph=graph,
    cypher_prompt=CYPHER_GENERATION_PROMPT,
    qa_prompt=QA_PROMPT,
    input_key="question",       # invoke할 때 넘기는 키
    output_key="result",        # 답변이 담기는 키
    allow_dangerous_requests=True,
    verbose=True,
)
answer = cypher_chain.invoke({"question": "배우 'Leonardo DiCaprio'와 감독 'Christopher Nolan'이 함께 작업한 영화가 있나요?"})
```

few-shot 예시는 아무거나 넣는 게 아니라, **모델이 자주 틀리는 패턴을 정답 쿼리로** 넣을 때 가장 효과가 커요. 예를 들어 날짜 비교, 배우와 감독을 동시에 묻는 V자 패턴 같은 것들이죠.

다만 예시가 틀리면 모델에게 틀린 패턴을 가르치게 돼요. 실제로 노트북의 예시 중 아래 쿼리는 이 데이터에서 0건이 나와요.

```cypher
WHERE m.released = 2000                 -- 숫자와 비교: 0건
WHERE m.released STARTS WITH '2000'     -- 문자열로 비교: 166건
```

그래서 예시 쿼리는 넣기 전에 직접 DB에서 한 번씩 돌려 보고 넣는 게 안전해요.

## 6. 정리하며 기억할 것

- LLM은 확률적으로 답을 내기 때문에, 같은 질문이어도 매번 쿼리가 다를 수 있고 가끔은 실패해요.
- 온톨로지를 어떻게 설계했는지(레이블 이름, `ACTED_IN` 같은 관계 이름)가 그대로 프롬프트에 들어가는 정보예요. 그래서 설계가 곧 Text2Cypher의 품질로 이어져요.
- 실패하면 `verbose`로 쿼리를 읽고, 스키마 → few-shot 예시 → 모델 순서로 손보면 돼요.

## 다음이랑 어떻게 이어지나

여기까지 영화 그래프를 기본 검색, 전문 검색, 벡터 검색, 그리고 Text2Cypher로 찾아봤어요. 다음 프로젝트 섹션들에서는 뉴스, 금융 상품, 기업 보고서, 법률 문서처럼 다른 도메인에 이 검색 방법들을 섞어서 graph RAG를 만들어요.
