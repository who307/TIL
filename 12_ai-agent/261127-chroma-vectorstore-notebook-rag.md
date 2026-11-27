# RAG 3: 노트북으로 RAG 중간값 확인 · 세션 메모리 후속 질문 · Chroma 영속 벡터 저장소·metadata 필터 · CSV 리뷰를 Document로 (TIL)

> 실습 환경: Python 3.12 (conda `pystudy_env`), Jupyter 노트북(VS Code + `ipykernel`), `langchain` 1.5, `langgraph` 1.2, `langchain-chroma` 1.2 (`chromadb` 1.5), `langchain-openai`, 임베딩 `text-embedding-3-small`, 모델 `gpt-4.1-mini`
>
> 어제 `.py`로 만든 RAG와 챗봇 메모리를 오늘은 **Jupyter 노트북에서 셀 하나씩 실행하며 중간값을 직접 확인**함. 오후에는 프로그램이 끝나면 사라지는 `InMemoryVectorStore` 대신 **디스크에 남는 Chroma**를 쓰고, metadata로 검색 범위를 좁힘. 마지막으로 와인 리뷰 CSV를 `Document`로 바꿔 저장 준비까지 함

## 0. 오늘의 흐름

| 노트북 | 내용 |
| :--- | :--- |
| `05_langchain_rag/06_RAG.ipynb` | TXT → Document → 조각 → 임베딩·저장 → 검색 → context → 프롬프트 → 답변, 단계별 변수 확인 |
| `06_chatbot_memory/01_History_And_Window.ipynb` | 이력 없는 요청 vs 있는 요청, `trim_messages` (어제 내용 노트북 복습) |
| `06_chatbot_memory/02_Sessions_And_Followup.ipynb` | `thread_id`별 기록, "그 둘의 차이" 후속 질문, 프로필 삭제 vs 대화 원문 |
| `07_vectorstore/01_VectorStore.ipynb` | Chroma 저장·재사용, 고정 ID, `get()`, metadata `filter` |
| `07_vectorstore/02_Wine_Review_RAG.ipynb` | `csv.DictReader`로 와인 리뷰 200개를 Document로 변환 (저장·검색·추천은 다음 시간) |

---

## 1. Jupyter 노트북 사용법

| 항목 | 내용 |
| :--- | :--- |
| 준비 | VS Code Jupyter 확장 설치, 오른쪽 위 **커널 선택 → `pystudy_env`**, 처음 실행 시 `ipykernel` 설치 |
| 실행 | 셀 왼쪽 ▶ 또는 **Shift+Enter** |
| 출력 | 셀의 **마지막 줄 값만** 자동 표시. 여러 값을 보려면 `print()` |
| 변수 | 실행한 셀의 변수가 **커널 메모리에 계속 남음** → 다음 셀에서 그대로 사용 |
| 작업 폴더 | `Path.cwd()` = 노트북 파일이 있는 폴더 |

* `.py`는 위에서 아래로 한 번에 실행되지만, 노트북은 **실행한 순서대로** 변수가 쌓임. 커널을 다시 켜면(Restart) 변수가 모두 사라지므로 **위 셀부터 다시** 실행해야 함
* 화면에 보이는 출력은 "마지막으로 실행했을 때"의 결과. 코드를 고쳐도 다시 실행하기 전까지 예전 에러가 그대로 보임

---

## 2. RAG 전 과정을 중간값으로 확인 (`06_RAG.ipynb`)

어제 `04_retrieval.py`에서 한 번에 돌던 과정을 셀마다 나눠, 각 단계의 **반환 타입이 어떻게 바뀌는지** 확인함.

| 단계 | 변수 | 타입 | 확인한 값 |
| :--- | :--- | :--- | :--- |
| TXT 읽기 | `documents` | `list[Document]` | 2개 (`center_policy.txt`, `course_guide.txt`) |
| 나누기 | `chunks` | `list[Document]` | 6개 (`chunk_size=220`, `overlap=40`), `source` 유지 |
| 임베딩·저장 | `document_ids` | `list[str]` | UUID 6개 |
| 벡터 | `vector` | `list[float]` | **1536차원** |
| 검색 | `results` | `list[Document]` | `k=2` → 2개 |
| 근거 묶기 | `blocks` → `context` | `list[str]` → `str` | `"\n\n".join(blocks)` |
| 답변 | `response` | `AIMessage` | `.content`로 본문 |

```python
first_record = store.store[document_ids[0]]   # InMemoryVectorStore 내부 기록
vector = first_record["vector"]
print("벡터 차원 수:", len(vector))           # 1536
print("벡터 앞 8개 값", vector[:8])            # [-0.0419, 0.0246, 0.0091, ...]
```

* `text-embedding-3-small`은 문장 하나를 **1536개의 숫자**로 바꿈. 숫자 한 칸이 단어 하나를 뜻하는 게 아니라 전체 패턴으로 비교함
* 질문을 "저녁에 공부할 공간은 몇 시까지 열려 있나요?"로 바꿨더니, "실습실" 조각이 1위로 검색되고 답변은 **"실습실은 평일 오후 9시까지 운영됩니다. 사용한 파일명: center_policy.txt"**. 질문에 "실습실"이라는 단어가 없어도 의미로 찾음
* 검색 결과(`results`)는 **답변이 아니라 근거 후보 목록**. 검색이 맞았는지와 모델이 맞게 답했는지는 따로 확인해야 함

### 노트북에서 겪은 에러

| 에러 | 원인 | 해결 |
| :--- | :--- | :--- |
| `ipykernel 패키지가 필요합니다` | 커널 환경에 `ipykernel` 없음 | `conda install -n pystudy_env ipykernel` |
| `NameError: DATA_DIR` | 커널 재시작 후 위 셀(경로)을 실행하지 않고 아래 셀부터 실행 | 위 셀부터 순서대로 실행 |
| 문서 수 0개 | 반복문에서 `Document`를 만들고 `documents.append(document)` 누락 | `append` 추가 |
| `NameError: os` | `os.getenv` 사용, `import os` 누락 | `import os` |
| `k=2`인데 결과 4개 | `search_kwags` 오타 → 설정이 **조용히 무시**되고 기본값 k=4 | `search_kwargs` |

* 마지막 오타는 에러가 나지 않아서 결과 개수를 보고서야 알았음. 노트북에서 `len(results)`처럼 중간값을 찍어 보는 이유

---

## 3. 세션 메모리와 후속 질문 (`02_Sessions_And_Followup.ipynb`)

어제 `03_session_memory.py`의 `create_agent` + `InMemorySaver` + `thread_id`를 노트북으로 다시 확인하고, **앞 대화를 가리키는 후속 질문**까지 해 봄.

### thread_id별 분리 결과

```text
student-a: human "내 이름은 은지야" → ai 인사 → human "내 이름이 뭐야?" → ai "은지님이세요!"
student-b: human "내 이름은 민수야" → ai 인사 → human "내 이름이 뭐야?" → ai "민수님이세요!"
```

* 같은 `agent` 객체를 쓰지만 `config`의 `thread_id`가 달라서 기록이 섞이지 않음. 질문은 매번 **하나만** 보냄

### 후속 질문 ("그 둘")

```python
QUESTIONS = [
    "내 이름은 민수야. Python 초보자이고 RAG를 배우고 있어.",
    "Document Loader의 역할을 두 문장으로 설명해줘",
    "Text Splitter의 역할도 두 문장으로 설명해줘",
    "그 둘의 차이를 Python 초보자가 이해할 수 있는 예시로 다시 설명해줘.",
]
followup_agent = create_agent(model=model, tools=[], system_prompt="...",
                              checkpointer=InMemorySaver())     # A/B와 다른 새 저장소
followup_config = {"configurable": {"thread_id": "followup-demo"}}

for question in QUESTIONS[1:3]:                                   # 인덱스 1, 2만
    result = followup_agent.invoke({"messages": [{"role": "user", "content": question}]},
                                   config=followup_config)
    print("누적 메세지:", len(result["messages"]))               # 4 → 6
```

| 시점 | 저장된 메시지 수 |
| :--- | :---: |
| 자기소개 후 | 2 |
| Loader 질문 후 | 4 |
| Splitter 질문 후 | 6 |
| "그 둘의 차이" 후 | **8** |

* 마지막 답변: "Document Loader는 `open('file.txt').read()`처럼 텍스트를 **가져오고**, Text Splitter는 `text.split('\n\n')`처럼 그 텍스트를 **쪼개는** 역할" → "그 둘"이 Loader·Splitter라는 걸 이력에서 찾아 답함. 자기소개의 "Python 초보자"도 반영해 파이썬 예시로 설명함
* `QUESTIONS[1:3]`: 슬라이싱으로 가운데 두 개만 반복, `QUESTIONS[-1]`: 마지막 질문
* `get_state(config).values["messages"]`로 마지막 질문 **전에** 저장 상태(6개)를 먼저 확인함 → 조회는 모델을 호출하지 않음

### 프로필 삭제 vs 대화 원문

```python
profile = {"name": "영희"}
history = [HumanMessage(content="내 이름은 영희야."), AIMessage(content="영희님, 반갑습니다.")]

removed_name = profile.pop("name", None)   # 키 제거 + 제거한 값 반환 (없으면 None)
print(profile)                              # {}
print(len(history))                         # 2 — 대화 속 "영희"는 그대로 남음
```

* 프로필에서 이름을 지워도 **대화 원문·체크포인트에 문장으로 남은 이름은 지워지지 않음**
* "개인정보 삭제" 요청을 처리하려면 프로필 DB, 대화 기록, 체크포인트처럼 **정보가 저장된 곳마다** 따로 지워야 함

### 겪은 에러

| 에러 | 원인 | 해결 |
| :--- | :--- | :--- |
| `ModuleNotFoundError: langchain_agents` | 모듈명 오타 | `from langchain.agents import create_agent` (점) |
| `OpenAIError: Missing credentials` | `load_dotenv`를 import만 하고 **호출 안 함** | `load_dotenv()` |
| 기억하지 못함 | `"configuable"` 오타, `invoke`에 `config` 미전달 | `{"configurable": {"thread_id": ...}}`, `config=config_a` |

---

## 4. Chroma: 디스크에 남는 벡터 저장소 (`01_VectorStore.ipynb`)

`InMemoryVectorStore`는 커널을 끄면 사라지고, 다시 켤 때마다 **임베딩 API를 다시 호출**해야 함(비용·시간). Chroma는 문서·벡터를 **폴더에 저장**해서 다음에 그대로 다시 열 수 있음.

```python
from langchain_chroma import Chroma

DATABASE_DIR = Path.cwd() / ".chroma-lab" / "center-policy-v1"

store = Chroma(
    collection_name="center-policy-v1",     # 저장소 안의 문서 묶음 이름
    embedding_function=embeddings,          # 문서·질문을 벡터로 바꿀 도구
    persist_directory=str(DATABASE_DIR),    # 저장 폴더 (없으면 생성)
)
```

| 구분 | `InMemoryVectorStore` | `Chroma` |
| :--- | :--- | :--- |
| 저장 위치 | 프로그램 메모리 | 디스크 폴더 (`persist_directory`) |
| 커널 재시작 후 | 사라짐 → 다시 임베딩 | **남아 있음 → 재사용** |
| 구분 단위 | 객체 하나 | 폴더 + `collection_name` |

### 고정 ID로 저장하고, 있으면 재사용

```python
DOCUMENTS = [
    Document(id="refund", page_content="수업 시작 24시간 전까지 취소하면 전액 환불됩니다.",
             metadata={"source": "center_policy.txt", "topic": "policy"}),
    Document(id="hours", page_content="토요일 실습실은 오전 10시부터 오후 5시까지 운영합니다.",
             metadata={"source": "center_policy.txt", "topic": "policy"}),
    Document(id="rag", page_content="RAG는 외부 문서를 검색해 모델 요청에 넣는 방식입니다.",
             metadata={"source": "course_guide.txt", "topic": "lesson"}),
]

stored = store.get(limit=1)                 # 최대 1건만 조회 (임베딩 호출 없음)
if stored["ids"]:
    print("상태: 저장한 문서와 벡터 재사용")
else:
    ids = [str(document.id) for document in DOCUMENTS]
    store.add_documents(DOCUMENTS, ids=ids)  # 이때만 임베딩 API 호출
    print("상태: 문서와 벡터를 처음 저장함")
```

| 실행 | `stored["ids"]` | 출력 |
| :--- | :--- | :--- |
| 처음 | `[]` | 문서와 벡터를 처음 저장함 |
| 커널 재시작 후 다시 | `['refund']` | **저장한 문서와 벡터 재사용** |

* 어제는 ID가 랜덤 UUID였는데, 여기서는 `"refund"`처럼 **고정 ID**를 줌. 같은 ID로 다시 넣으면 새로 쌓이지 않고 덮어써서 중복이 생기지 않음
* 재사용 조건이 "ID가 하나라도 있으면"이라 **문서를 고쳐도 다시 저장되지 않음**. 원본을 바꿨으면 컬렉션 이름에 `-v2`를 붙이거나 폴더를 지우고 다시 저장해야 함 (그래서 이름에 `v1`)
* `.chroma-lab/` 폴더는 실행하면 생기는 결과물이라 `.gitignore`에 넣는 게 좋음 (현재 lecture 저장소 `.gitignore`에는 아직 없음)

### 저장 내용 조회: `get()`

```python
saved = store.get(include=["documents", "metadatas"])
for item in zip(saved["ids"], saved["documents"], saved["metadatas"]):
    print(item)
# ('refund', '수업 시작 24시간 전까지 ...', {'source': 'center_policy.txt', 'topic': 'policy'})
# ('hours', ...), ('rag', ...)
```

* `get()`은 **딕셔너리**(`ids`·`documents`·`metadatas`가 같은 순서의 리스트)를 반환. 유사도 검색이 아니라 저장된 걸 그대로 꺼내는 조회
* `zip()`: 같은 위치의 값끼리 튜플로 묶어 반복

### metadata 필터로 검색 범위 좁히기

```python
all_retriever = store.as_retriever(search_kwargs={"k": 2})
policy_retriever = store.as_retriever(search_kwargs={"k": 2, "filter": {"topic": "policy"}})
```

| 검색 | 결과 (질문: "강의를 취소하면 결제한 돈을 돌려받을 수 있나요?") |
| :--- | :--- |
| 전체 | `refund`(환불), **`rag`(RAG 설명, topic=lesson)** |
| `filter={"topic": "policy"}` | `refund`(환불), `hours`(실습실 시간) |

* 전체 검색에서는 k=2를 채우려고 관계없는 수업 안내(`rag`)까지 들어옴. 필터를 걸면 **정책 문서 안에서만** 찾음
* `filter`는 "어디서 찾을지(대상)", `k`는 "그중 몇 개(개수)". 필터를 걸었다고 결과가 모두 정답 근거인 건 아님 (`hours`는 정책이지만 환불 질문과는 무관)
* 실무에서는 부서·문서 종류·작성일·사용자 권한 등을 metadata에 넣고 필터로 범위를 제한함

### 겪은 에러

| 에러 | 원인 | 해결 |
| :--- | :--- | :--- |
| `Missing credentials` | `load_dotenv()` 미호출 | 호출 추가 |
| 경고 `max_retires was transferred to model_kwargs` | `max_retries` 오타 → 에러 대신 **API로 보낼 옵션으로 넘어감** → 실제 임베딩 요청 때 API 에러 | `max_retries` |
| `AttributeError: 'NoneType' ... '__name__'` | `print(type(stored)).__name__` → `print()`의 반환값 `None`에서 속성 접근 | `print(type(stored).__name__, ...)` |
| `SyntaxError: cannot assign to literal` | 딕셔너리에 `{"k"=2}` | `{"k": 2}` (중괄호 안은 `:`) |

---

## 5. CSV 리뷰를 Document로 (`02_Wine_Review_RAG.ipynb`)

와인 리뷰 CSV(`title`, `description` 열)를 Document로 바꿔, 음식 설명으로 어울리는 와인 리뷰를 찾는 RAG를 준비함.

```python
import csv
from itertools import islice

with CSV_PATH.open(encoding="utf-8-sig", newline="") as file:
    reader = csv.DictReader(file)
    print(reader.fieldnames)            # ['title', 'description']
    sample_row = next(reader)           # 첫 데이터 행 (dict)
```

| 문법 | 의미 |
| :--- | :--- |
| `csv.DictReader` | 첫 줄(열 이름)을 키로 써서 각 행을 **딕셔너리**로 읽음 |
| `encoding="utf-8-sig"` | 파일 맨 앞의 BOM(`﻿`)을 제거. 안 하면 첫 열 이름이 `"﻿title"`이 되어 `row["title"]`에서 `KeyError` |
| `newline=""` | csv 모듈 권장 설정. 따옴표 안의 줄바꿈을 올바르게 처리 |
| `next(reader)` | 다음 행 하나를 꺼냄 (Java `Iterator.next()`) |

```python
def load_reviews(csv_path, limit=200):
    documents = []
    with csv_path.open(encoding="utf-8-sig", newline="") as file:
        for number, row in enumerate(islice(csv.DictReader(file), limit), start=1):
            title, review = row["title"].strip(), row["description"].strip()
            if title and review:                       # 빈 값은 건너뜀
                documents.append(Document(
                    page_content=review,               # 검색 대상: 리뷰 본문
                    metadata={"title": title, "review_number": number},
                ))
    return documents

documents = load_reviews(CSV_PATH)   # list 200개
# {'title': 'Nicosia 2013 Vulkà Bianco  (Etna)', 'review_number': 1}
# Aromas include tropical fruit, broom, brimstone and dried herb. ...
```

* `islice(반복자, 200)`: 앞에서 200행까지만 읽음. 리스트 슬라이싱(`[:200]`)과 달리 **파일 전체를 메모리에 올리지 않고** 필요한 만큼만 읽음
* **본문에는 검색할 내용(리뷰)**, **metadata에는 표시·필터용 정보(와인 이름, 번호)**를 나눠 담음. 와인 이름까지 본문에 넣으면 임베딩이 이름에 끌려갈 수 있음
* `a, b = x, y`: 두 값을 한 줄에 대입 (튜플 언패킹)
* CSV 파일이 노트북과 같은 폴더에 있어서 `FileNotFoundError`가 났음 → `07_vectorstore/data/`로 옮겨 코드의 `Path.cwd() / "data" / ...`와 맞춤
* 이후 단계(Chroma 저장 → 음식 설명으로 `k=3` 검색 → 리뷰 근거로 와인 추천)는 아직 작성 전

---

## ⚠️ 오늘 반복된 실수

| 유형 | 사례 | 교훈 |
| :--- | :--- | :--- |
| **조용히 무시되는 오타** | `search_kwags`, `configuable`, `max_retires` | 에러 없이 설정만 빠짐. 결과 개수·저장 개수 같은 **중간값을 찍어서** 확인 |
| `load_dotenv` 미호출 | 노트북 3개에서 반복 | import와 호출은 다름. 준비 셀에 `load_dotenv()` 습관화 |
| 셀 실행 순서 | 재시작 후 아래 셀부터 실행 → `NameError` | 커널 재시작 = 변수 초기화. 위부터 다시 |
| 노트북 파일 덮어쓰기 | 노트북을 연 채로 외부에서 고친 파일이 에디터 저장으로 덮어써짐 | 외부 수정 후에는 탭을 닫았다 열어 디스크 버전을 불러옴 |

---

## 📝 오늘의 정리

1. 노트북은 셀마다 변수가 커널에 쌓이고, 마지막 줄 값만 출력됨. 재시작하면 위부터 다시 실행
2. RAG 단계별 타입: `list[Document]`(원본) → `list[Document]`(조각) → 1536차원 벡터 → `list[Document]`(검색) → `str`(context) → `AIMessage`(답변)
3. `create_agent` + `thread_id`로 사용자별 기록이 분리되고, 같은 방에서는 "그 둘" 같은 후속 질문도 앞 대화로 해석함. 메시지는 질문·답변마다 2개씩 누적
4. 프로필에서 지운 정보도 대화 원문·체크포인트에는 남음 → 삭제는 저장 위치별로
5. Chroma는 `persist_directory`에 저장해 재시작 후에도 재사용. 고정 ID + "있으면 재사용" 패턴으로 임베딩 중복 호출을 막음
6. `get()`은 저장 내용 조회(딕셔너리), `as_retriever().invoke()`는 유사도 검색(`list[Document]`). `filter={"topic": "policy"}`로 검색 대상을 metadata로 제한
7. CSV는 `csv.DictReader` + `utf-8-sig` + `newline=""`로 읽고, `islice`로 행 수를 제한해 본문(리뷰)·metadata(제목·번호)를 나눈 Document로 만듦

---

## 느낀 점

* 어제 `.py`로 한 번에 돌릴 때는 안 보이던 중간 결과(1536개 숫자, 조각 6개, 검색 2개)를 셀마다 보니 RAG가 "데이터 타입이 바뀌어 가는 파이프라인"이라는 게 확실히 보였음
* 오늘 가장 시간을 많이 쓴 건 에러가 **안 나는** 오타였음. `search_kwags` 하나 때문에 결과가 4개가 나왔는데 에러 메시지가 없어서, 결과를 직접 세어 보는 습관이 중요하다고 느낌
* Chroma의 "있으면 재사용" 조건이 문서가 바뀌어도 그대로 재사용한다는 점이 캐시 무효화 문제처럼 느껴졌음. 실무에서는 버전 관리나 변경 감지가 필요할 것 같음
* 프로필 삭제 예제에서, 이름을 지워도 대화 기록에는 남는다는 게 개인정보 처리 관점에서 꽤 중요한 포인트라고 생각함
