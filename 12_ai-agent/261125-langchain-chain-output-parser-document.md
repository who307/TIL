# LangChain: 프롬프트 템플릿·체인·Output Parser · RAG 시작: Document·TXT/PDF 읽기 (TIL)

> 실습 환경: Python 3.12 (conda `pystudy_env`), `langchain-core` 1.8, `langchain-openai` 1.8, `langchain-community` 0.4, `pydantic` 2.14, `pypdf` 7.1, 모델 `gpt-4.1-mini`
>
> 어제까지는 `openai` 라이브러리로 프롬프트 문자열 만들기 → 모델 호출 → 응답 꺼내기 → JSON 검증을 직접 코드로 짰음. 오늘은 이 단계들을 **LangChain 부품으로 바꾸고 `|`로 이어 하나의 체인**으로 만듦. 오후에는 RAG의 첫 단계로 TXT·PDF 파일을 LangChain의 `Document` 객체로 읽어 들임

## 0. 오늘의 흐름

| 파일 | 내용 | 핵심 |
| :--- | :--- | :--- |
| `04_langchain/01_prompt_template.py` | 프롬프트 템플릿 | `PromptTemplate`·`ChatPromptTemplate`, `{변수}` 채우기 |
| `04_langchain/02_runnable_chain.py` | 체인 | `prompt \| model \| StrOutputParser()` |
| `04_langchain/03_output_parser.py` | 구조화된 출력 | Pydantic 클래스 + `PydanticOutputParser`로 응답 검증 |
| `04_langchain/04_quiz_generation.py` | 퀴즈 생성 | `@model_validator`로 "정답은 보기 중 하나" 같은 필드 간 규칙 검증 |
| `05_langchain_rag/01_loader_and_documents.py` | 문서 객체 | TXT → `Document(page_content, metadata)`, `TextLoader` |
| `05_langchain_rag/02_pdf_documents.py` | PDF 읽기 | `pypdf`로 페이지별 텍스트 추출 → 페이지 번호를 metadata에 저장 |

```text
어제 (openai 직접 사용)                         오늘 (LangChain)
f-string으로 프롬프트 조립          →  PromptTemplate / ChatPromptTemplate
client.chat.completions.create()    →  ChatOpenAI
response.choices[0].message.content →  StrOutputParser
json.loads + 키·타입 직접 검사      →  Pydantic 클래스 + PydanticOutputParser
                    ↓
         prompt | model | parser  (한 줄 체인)
```

---

## 1. LangChain과 프롬프트 템플릿 (`01_prompt_template.py`)

**LangChain**: LLM을 쓰는 프로그램에서 **프롬프트 작성 → 모델 호출 → 응답 처리**를 부품으로 나누고, 부품을 연결해 하나의 실행 흐름(체인)으로 만드는 라이브러리.

```powershell
pip install langchain-core langchain-openai python-dotenv
```

| 패키지 | 들어 있는 것 |
| :--- | :--- |
| `langchain-core` | 프롬프트 템플릿, Output Parser, `Document`, 체인 연결(`\|`) 같은 기본 부품 |
| `langchain-openai` | `ChatOpenAI` (OpenAI 모델 연결) |
| `langchain-community` | `TextLoader` 같은 외부 연동 모음 (5장) |

### PromptTemplate: 문자열 하나짜리 템플릿

```python
from langchain_core.prompts import ChatPromptTemplate, PromptTemplate

template = PromptTemplate.from_template("{audience}에게 {topic}에 대해 쉽게 설명하세요.")
prompt_value = template.invoke({
    "audience": "Java를 배운 Python 입문자",
    "topic": "LLM",
})
print("문자열 템플릿 :", prompt_value.to_string())
```

```text
문자열 템플릿 : Java를 배운 Python 입문자에게 LLM에 대해 쉽게 설명하세요.
```

### ChatPromptTemplate: 역할별 메시지 목록 템플릿

```python
template = ChatPromptTemplate.from_messages([
    ("system", "초보자에게 쉬운 말로 설명하세요"),
    ("user", "질문: {question}\n답변 형식:{answer_format}"),
])
prompt_value = template.invoke({
    "question": "LangChain에서 프롬프트템플릿을 만드려면 어떻게 하나요?",
    "answer_format": "한 줄 정의와 짧은 예시",
})
for message in prompt_value.to_messages():
    print(f"[{message.type}] {message.content}")
```

```text
[system] 초보자에게 쉬운 말로 설명하세요
[human] 질문: LangChain에서 프롬프트템플릿을 만드려면 어떻게 하나요?
답변 형식:한 줄 정의와 짧은 예시
```

* `from_template` / `from_messages`: 클래스 메소드로 템플릿을 만듦. `{이름}` 자리가 나중에 채울 변수가 됨
* `invoke(딕셔너리)`: 변수 자리를 채워 **`PromptValue`**를 돌려줌. `to_string()`은 문자열, `to_messages()`는 메시지 객체 목록
* 메시지는 `("역할", "내용")` 튜플로 적음. `"user"`로 적어도 LangChain 안에서는 **`human`** 타입(`HumanMessage`)으로 바뀜. 모델 응답은 `ai`(`AIMessage`)
* 어제는 f-string으로 프롬프트를 바로 만들었는데, 템플릿을 쓰면 **틀은 한 번 정의하고 값만 바꿔 재사용**할 수 있음. 같은 `invoke` 방식으로 다음 부품과 연결할 수 있다는 점이 더 중요함 (2장)

---

## 2. 체인으로 연결하기 (`02_runnable_chain.py`)

```python
from dotenv import load_dotenv
from openai import APIError
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI

def build_chain(model):
    prompt = ChatPromptTemplate.from_messages([
        ("system", "LLM 활용을 가르치는 강사입니다. 세 문장 이내로 설명하세요"),
        ("user", "{audience}에게 {topic}을 쉽게 설명하세요"),
    ])
    return prompt | model | StrOutputParser()

def main():
    load_dotenv()
    model = ChatOpenAI(
        model="gpt-4.1-mini",
        temperature=0.7,
        max_completion_tokens=300,
        timeout=20,
        max_retries=0,
        use_responses_api=False,
    )
    chain = build_chain(model)
    try:
        result = chain.invoke({"audience": "Python 입문자", "topic": "LangChain"})
    except APIError as error:
        print("요청 실패", type(error))
        return
    print("결과", result)
```

```text
결과 LangChain은 파이썬으로 AI 모델과 쉽게 연결해 대화형 프로그램을 만들 수 있는 도구입니다. 복잡한 작업을 작은 블록처럼 연결해 자동화하고 확장하는 데 도움을 줍니다. 입문자도 간단한 코드로 챗봇이나 문서 요약 같은 AI 기능을 구현할 수 있어요.
```

### `|` (파이프)로 연결되는 흐름

```text
{"audience": ..., "topic": ...}
   │ prompt.invoke()
   ▼
PromptValue (system + human 메시지)
   │ model.invoke()          ← 여기서 OpenAI API 호출
   ▼
AIMessage (content, 토큰 사용량 등 메타 정보)
   │ StrOutputParser().invoke()
   ▼
str  (응답 텍스트만)
```

* **Runnable**: LangChain 부품들이 공통으로 가진 인터페이스. 모두 `invoke(입력)`으로 실행되고 결과를 돌려줌. 그래서 서로 이어 붙일 수 있음
* `a | b | c`: 왼쪽 결과를 오른쪽 입력으로 넘기는 체인. 체인도 Runnable이라 `chain.invoke(...)` 한 번으로 전체가 실행됨. 리눅스 셸 파이프(`cat a.txt | grep ...`)와 같은 발상
* `StrOutputParser()`: `AIMessage`에서 텍스트(`content`)만 꺼내 문자열로 돌려줌. 어제의 `response.choices[0].message.content`를 대신함
* 각 부품은 단독으로도 `invoke`할 수 있어서, 체인이 이상하면 **한 단계씩 따로 실행해 보며** 어디서 틀렸는지 찾을 수 있음

### `ChatOpenAI` 설정

| 인자 | 의미 |
| :--- | :--- |
| `model="gpt-4.1-mini"` | 사용할 모델 |
| `temperature` | 응답의 무작위성. 설명문은 `0.7`, 분류·검증 실습은 결과가 일정하도록 `0` |
| `max_completion_tokens` | 모델이 생성할 최대 토큰 수 |
| `timeout=20` | 20초 안에 응답이 없으면 실패 처리 |
| `max_retries=0` | 실패 시 자동 재시도하지 않음 (실습에서 오류를 바로 보려고) |
| `use_responses_api=False` | Chat Completions API 사용. `True`면 Responses API 방식이며, OpenAI 내장 웹 검색·파일 검색이 필요할 때 검토 |

* API 키는 인자로 넘기지 않음. `load_dotenv()`가 `.env`의 `OPENAI_API_KEY`를 환경 변수로 올리면 `ChatOpenAI`가 알아서 읽음
* LangChain이 내부에서 `openai` 라이브러리를 쓰기 때문에 네트워크·인증 오류는 그대로 `openai.APIError`로 잡을 수 있음

---

## 3. Output Parser로 응답 검증 (`03_output_parser.py`)

어제는 `json.loads` 후 `set(item)`, `type(x) is int` 등으로 키·타입·허용 값을 하나씩 검사했음. 오늘은 **규칙을 클래스로 선언**하고 검증은 Pydantic에 맡김.

### Pydantic 클래스로 규칙 선언

```python
from typing import Literal
from pydantic import BaseModel, ConfigDict, Field

class SentimentAnalysis(BaseModel):
    model_config = ConfigDict(extra="forbid")
    label: Literal["긍정", "부정", "중립"] = Field(description="허용된 감정 분류")
    reason: str = Field(min_length=1, description="판단 근거")
```

* **Pydantic**: 클래스에 적은 **타입과 규칙으로 입력 데이터를 검증**하는 Python 라이브러리. `BaseModel`을 상속해 필드를 선언하면, 검증을 통과한 값만 객체로 만들어짐. Spring에서 DTO에 `@NotBlank`·`@Size`를 붙이고 `@Valid`로 검증하던 것과 비슷함
* `Literal["긍정", "부정", "중립"]`: 이 세 값만 허용 (어제 `if label not in ALLOWED`로 하던 일)
* `Field(min_length=1)`: 빈 문자열 금지. `description`은 모델에게 보내는 형식 안내에 들어감
* `ConfigDict(extra="forbid")`: 선언하지 않은 키가 오면 실패 (어제 `set(item) != EXPECTED_KEYS` 검사)

### PydanticOutputParser + 형식 안내 주입

```python
def build_chain(model):
    # JSON을 클래스 규칙으로 검증하고, 통과하면 그 클래스의 객체로 반환
    parser = PydanticOutputParser(pydantic_object=SentimentAnalysis)
    # 클래스의 필드와 규칙을 모델에게 전달할 JSON 형식 안내문으로 만듦
    format_instructions = parser.get_format_instructions()

    prompt = PromptTemplate.from_template(
        "아래 수강평의 감정을 분류하고 이유를 적으세요.\n"
        "{format_instructions}\n"
        "수강평: {sentence}"
    )
    prompt = prompt.partial(format_instructions=format_instructions)
    return prompt | model | parser
```

`get_format_instructions()`가 실제로 만드는 안내문 (프롬프트의 `{format_instructions}` 자리에 들어감):

```text
The output should be formatted as a JSON instance that conforms to the JSON schema below.
...
Here is the output schema:
{"additionalProperties": false,
 "properties": {"label": {"description": "허용된 감정 분류", "enum": ["긍정", "부정", "중립"], "type": "string"},
                "reason": {"description": "판단 근거", "minLength": 1, "type": "string"}},
 "required": ["label", "reason"]}
```

* 클래스 하나가 **두 가지 일**을 함: ① 모델에게 보낼 JSON Schema 안내 ② 돌아온 응답 검증. 규칙을 바꿀 때 클래스만 고치면 둘 다 바뀜
* `extra="forbid"` → `additionalProperties: false`, `Literal` → `enum`, `min_length` → `minLength`처럼 Pydantic 규칙이 JSON Schema로 그대로 옮겨짐
* `prompt.partial(format_instructions=...)`: 변수 일부를 **미리 채운 새 템플릿**을 만듦. 형식 안내는 매번 같으니 미리 넣고, 실행할 때는 `{"sentence": ...}`만 넘기면 됨

### 실행과 예외 처리

```python
try:
    result = build_chain(model).invoke({"sentence": "수업이 어렵지만 예제가 있어서 따라갈 만하다."})
except OutputParserException:
    print("응답 검증 실패: 감정 분류와 이유 형식을 확인하세요")
    return
except APIError as error:
    print("API 요청 실패", type(error).__name__)
    return
print("결과 타입: ", type(result).__name__)
print("감정", result.label)
print("이유:", result.reason)
```

```text
결과 타입:  SentimentAnalysis
감정 중립
이유: 수업이 어렵다는 부정적인 내용과 예제가 있어서 따라갈 만하다는 긍정적인 내용이 함께 포함되어 있어 ...
```

* 결과가 딕셔너리가 아니라 **`SentimentAnalysis` 객체**라서 `result["label"]` 대신 `result.label`로 읽음. 오타를 내면 IDE가 잡아 줌
* 규칙을 어긴 응답이 오면 `OutputParserException` (`from langchain_core.exceptions import OutputParserException`). 일부러 틀린 JSON을 넣어 보면 Pydantic이 **무엇이 왜 틀렸는지** 알려 줌

```text
OutputParserException Failed to parse SentimentAnalysis from completion {"label": "좋음", "reason": ""}.
Got: 2 validation errors for SentimentAnalysis
label
  Input should be '긍정', '부정' or '중립' [type=literal_error, input_value='좋음', ...]
reason
  String should have at least 1 character ...
```

### JsonOutputParser와 비교

```python
format_only = JsonOutputParser()
data = format_only.invoke(SAMPLE_REPLY)   # '{"label":"중립", "reason":"..."}'
print(data["label"])                      # dict로 읽기만 함
```

| | `JsonOutputParser` | `PydanticOutputParser` |
| :--- | :--- | :--- |
| 결과 | `dict` | 지정한 클래스의 객체 |
| 검증 | JSON 문법만 | 타입·허용 값·길이·추가 키까지 |
| 쓰임 | 형식이 자유로운 JSON을 읽기만 할 때 | 형식이 정해진 응답을 코드에서 바로 쓸 때 |

* Parser도 Runnable이라 모델 없이 **문자열을 바로 `invoke`**해서 테스트할 수 있음. API 호출 없이 검증 규칙만 확인할 때 유용

---

## 4. 퀴즈 생성과 필드 간 규칙 검증 (`04_quiz_generation.py`)

필드 하나하나의 규칙(타입·길이)만으로는 부족한 경우. "정답은 보기 중 하나여야 한다"는 **여러 필드를 함께 봐야** 확인할 수 있음.

```python
from pydantic import BaseModel, ConfigDict, Field, model_validator

class Quiz(BaseModel):
    model_config = ConfigDict(extra="forbid")
    question: str = Field(min_length=1, description="퀴즈")
    choices: list[str] = Field(min_length=4, max_length=4, description="보기 네 개")
    answer: str = Field(min_length=1, description="보기 중 정답 문자열")
    explanation: str = Field(min_length=1, description="정답 설명")

    @model_validator(mode="after")
    def check_answer(self):
        if any(not choice.strip() for choice in self.choices):
            raise ValueError("보기는 비어 있을 수 없습니다.")
        if len(set(self.choices)) != len(self.choices):   # set: 중복 제거
            raise ValueError("보기는 서로 달라야 합니다.")
        if self.answer not in self.choices:
            raise ValueError("정답은 보기 중 하나여야 합니다.")
        return self
```

* `list[str]`에 `min_length=4, max_length=4`를 주면 **리스트 길이**가 정확히 4개인지 검사함 (문자열이면 글자 수, 리스트면 항목 수)
* `@model_validator(mode="after")`: 필드별 검증이 **모두 끝난 뒤** 객체 전체를 보고 추가 검사. 여기서 `ValueError`를 던지면 Pydantic 검증 실패로 처리되고, 체인에서는 `OutputParserException`이 됨. 마지막에 `return self` 필수
* `any(조건 for ...)`: 하나라도 참이면 `True`. `" "`처럼 공백만 있는 보기는 `min_length`로는 못 막아서 `strip()`으로 따로 확인
* `len(set(리스트)) != len(리스트)`: 중복을 지운 개수와 원래 개수가 다르면 중복이 있다는 뜻
* 프롬프트에도 같은 규칙을 적음 ("서로 다른 문장 네 개", "정답은 보기 중 하나의 문장을 그대로"). **프롬프트로 지키게 하고, 클래스로 확인**하는 두 겹 구조

```python
def show_quiz(model, topic):
    quiz = build_quiz_chain(model).invoke({"topic": topic})
    print("문제", quiz.question)
    for number, choice in enumerate(quiz.choices, start=1):   # 번호와 값을 함께 꺼냄
        print(f"{number}. {choice}")
    print("정답:", quiz.answer)
    print("해설:", quiz.explanation)
```

```text
문제 Python 딕셔너리에서 키(key)에 해당하는 값에 접근하려면 어떻게 해야 하나요?
1. 딕셔너리 이름 뒤에 대괄호 [] 안에 키를 넣어 접근한다.
2. 딕셔너리 이름 뒤에 소괄호 () 안에 키를 넣어 접근한다.
3. 딕셔너리 이름 뒤에 중괄호 {} 안에 키를 넣어 접근한다.
4. 딕셔너리 이름 뒤에 점(.)을 찍고 키 이름을 문자열로 적어 접근한다.
정답: 딕셔너리 이름 뒤에 대괄호 [] 안에 키를 넣어 접근한다.
```

* `enumerate(목록, start=1)`: 기본은 0부터 세므로 화면 번호용으로 `start=1`
* 응답 JSON이 길어지는 작업이라 `max_completion_tokens=200`이면 잘릴 수 있음. 잘린 JSON은 파싱에 실패해 `OutputParserException`으로 떨어짐. 결과가 길어지면 한도를 늘려야 함

---

## 5. RAG 시작: Document 객체 (`01_loader_and_documents.py`)

**RAG(Retrieval-Augmented Generation)**: 질문과 관련된 문서를 먼저 **검색**하고, 검색 결과를 LLM의 Context로 넣어 답을 **생성**하는 방식. 모델이 학습하지 않은 사내 규정·수업 자료 같은 정보로 답할 때 씀. 어제 Function Calling에서 `search_lesson` 결과를 답에 넣었던 구조를 문서 전체로 넓힌 것.

```text
원본 파일(TXT·PDF) → [Loader] Document → [Splitter] 조각 → [Embedding] 벡터 → 검색 → LLM 답변
                     ^^^^^^^^^^^^^^^^^^
                       오늘 한 부분
```

### Document 직접 만들기

```python
from pathlib import Path
from langchain_core.documents import Document

DATA_DIR = Path(__file__).resolve().parent / "data"
FILENAMES = ["center_policy.txt", "course_guide.txt"]

documents = []
for filename in FILENAMES:
    path = DATA_DIR / filename
    document = Document(
        page_content=path.read_text(encoding="utf-8"),   # 본문
        metadata={"source": path.name},                   # 부가 정보(출처)
    )
    documents.append(document)
```

```text
타입: Document
출처: center_policy.txt
본문 앞부분: [새싹 AI 학습센터 운영 정책]
수강 취소는 수업 시작 24시간 전까지 신청해야 전액 환불된다. ...
```

* **`Document`**: LangChain에서 문서를 다루는 기본 단위. `page_content`(본문 문자열) + `metadata`(파일명·페이지·작성일 같은 부가 정보 딕셔너리)
* metadata에 출처를 남겨야 나중에 답변에 **"이 내용은 center_policy.txt에서 왔다"**고 근거를 붙일 수 있음. 사용자가 답을 다시 확인할 수 있게 하는 RAG의 핵심 정보
* `Path(__file__).resolve().parent`: **실행 중인 파이썬 파일이 있는 폴더**. 터미널을 어느 폴더에서 열고 실행하든 `data/` 경로가 같은 곳을 가리킴 (그냥 `"data/..."`라고 쓰면 현재 작업 폴더 기준이라 위치에 따라 `FileNotFoundError`)
* `Path / "문자열"`: `/` 연산자로 경로를 이어 붙임. `path.name`은 파일 이름만, `path.read_text(encoding="utf-8")`는 파일 전체를 문자열로 읽음

### TextLoader로 읽기

```python
from langchain_community.document_loaders import TextLoader

loader = TextLoader(str(DATA_DIR / filename), encoding="utf-8")
documents.extend(loader.load())    # load()가 list[Document]를 돌려줌
```

```text
TextLoader                                   ← TextLoader(...)만 하면 로더 객체
list 1 {'source': 'data/center_policy.txt'}  ← .load()를 불러야 Document 목록
```

* **Document Loader**: PDF·TXT·웹 페이지 같은 원본에서 텍스트와 metadata를 읽어 `Document`로 만들어 주는 부품. 직접 만든 코드와 결과는 같음
* `load()`는 Document **목록**을 돌려줘서 `append`가 아니라 `extend`로 붙임
* `TextLoader`는 `source`에 **넘겨준 경로 문자열을 그대로** 넣음. 그래서 출력 전에 `Path(document.metadata["source"]).name`으로 파일 이름만 남기도록 정리함
* 실행하면 `` `langchain-community` is being sunset `` 경고가 뜸. 커뮤니티 패키지는 유지보수가 끝나는 중이라 연동별 독립 패키지로 옮겨 가는 추세. 지금 실습에는 영향 없음

---

## 6. PDF를 페이지별 Document로 (`02_pdf_documents.py`)

```python
from pathlib import Path
from langchain_core.documents import Document
from pypdf import PdfReader

DATA_FILE = Path(__file__).resolve().parent / "data" / "course_handbook.pdf"

def main():
    reader = PdfReader(DATA_FILE)
    documents: list[Document] = []

    for page_number, page in enumerate(reader.pages, start=1):
        text = page.extract_text() or ""    # 페이지 안의 텍스트를 문자열로 추출
        if not text.strip():
            continue                        # 빈 페이지(이미지만 있는 페이지 등)는 건너뜀
        documents.append(Document(
            page_content=text.strip(),
            metadata={"source": DATA_FILE.name, "page": page_number},
        ))

    print("PDF 전체 페이지 수:", len(reader.pages))
    print("만들어진 Document 수:", len(documents))
    for document in documents:
        print(f"\n[{document.metadata['source']} / {document.metadata['page']}페이지]")
        print(document.page_content[:300])
```

```text
PDF 전체 페이지 수: 2
만들어진 Document 수: 2

[course_handbook.pdf / 1페이지]
새싹 AI 과정 수강 안내 | 1페이지
 새싹 AI 과정 수강 안내
1. 수업 일정
정규 수업은 매주 월요일부터 금요일까지 오전 9시부터 오후 6시까지 진행됩니다. 오전
수업 시작 10분 전까지 강의실에 입실하고 ...
구분
기준
정상 출석
수업 시작 전 입실
지각
수업 시작 10분 이후 입실
...
```

* `pip install pypdf`. `PdfReader(경로)` → `reader.pages`(페이지 목록) → `page.extract_text()`
* **페이지 하나 = Document 하나**로 만들고 `page` 번호를 metadata에 저장. 답변 근거를 "course_handbook.pdf 2페이지"처럼 **페이지 단위로** 제시할 수 있음
* `extract_text() or ""`: 텍스트가 없는 페이지에서 `None`·빈 값이 와도 다음 줄 `.strip()`에서 오류가 나지 않도록 빈 문자열로 바꿈
* 출력을 보면 PDF에서 뽑은 텍스트는 **원본 모양 그대로가 아님**
  * 문장 중간에 줄바꿈이 들어감 (`오전` / `수업 시작`): PDF의 화면상 줄바꿈이 그대로 옮겨짐
  * **표가 한 칸씩 줄로 풀림** (`구분` / `기준` / `정상 출석` / `수업 시작 전 입실`): 어느 값이 어느 열인지 관계가 사라짐
  * 머리글(`새싹 AI 과정 수강 안내 | 1페이지`)이 본문 앞에 섞임
* RAG 답변 품질은 결국 이 텍스트에 달려 있어서, PDF는 **추출 결과를 직접 눈으로 확인**하는 단계가 필요함. 다음 단계(Text Splitter)에서 이 텍스트를 조각으로 나눔

---

## 7. 실행하면서 고친 것

| 위치 | 문제 | 증상 | 수정 |
| :--- | :--- | :--- | :--- |
| `02_runnable_chain.py` 1줄 | `from openai import` 뒤가 비어 있음 | Pylance Error, `SyntaxError` | `from dotenv import load_dotenv` |
| `02_runnable_chain.py` | `from_messagses` | 실행 시 `AttributeError` | `from_messages` |
| `02_runnable_chain.py` | `"gtp-4.1-mini"` | OpenAI가 모델을 찾지 못해 API 오류 | `"gpt-4.1-mini"` |
| `03_output_parser.py` | `PromptTemplate.form_template` | 실행 시 `AttributeError`, **Pylance는 표시 안 함** | `from_template` |
| `03_output_parser.py` | `OutputParserException` import 누락 | 검증 실패로 `except`에 도달하는 순간 `NameError` | `from langchain_core.exceptions import OutputParserException` |
| `03_output_parser.py` | `Field(min_lenght=1)` | **오류 없이 규칙만 무시됨** (경고만 출력) | `min_length=1` |
| `01_loader_and_documents.py` | `Document` import 누락 | `NameError` | `from langchain_core.documents import Document` |
| `01_loader_and_documents.py` (주석) | `TextLoader(...)`를 바로 `extend` | 로더 객체는 Document 목록이 아님 | `TextLoader(...).load()` |

* **Pylance가 못 잡는 오타가 있음**: `PromptTemplate`·`ChatPromptTemplate`은 Pydantic 모델이라, 없는 클래스 속성(`form_template`)에 접근해도 빨간 줄이 안 뜸. 빨간 줄이 없다고 끝이 아니라 **실행해서 확인**해야 함
* **Pydantic은 모르는 인자를 받아도 멈추지 않음**: `min_lenght`처럼 철자가 틀리면 `PydanticDeprecatedSince20: Using extra keyword arguments on Field is deprecated` 경고만 나오고, 그 규칙은 **적용되지 않은 채** 실행됨. 검증 코드가 조용히 빠지는 거라 오히려 더 위험함. 실행 결과의 경고 메시지를 꼭 읽어야 함
* except 블록 안의 `NameError`는 그 예외가 실제로 발생하기 전까지 드러나지 않음. 정상 응답만 받을 때는 문제없이 돌아가다가 검증 실패 순간에 터짐

### 아직 남아 있는 것 (`04_quiz_generation.py`)

실행하면 동작은 하지만 고쳐야 할 부분. (4장 코드는 고친 버전으로 적음)

| 줄 | 현재 | 결과 | 고칠 내용 |
| :--- | :--- | :--- | :--- |
| `choices` | `max_legnth=4` | 경고만 나오고 **최대 4개 제한이 적용 안 됨** (보기 5개도 통과) | `max_length=4` |
| `answer` | `min_legnth=1` | 경고만 나오고 빈 정답 검사 안 됨 (단, `answer not in choices` 검사로 일부 막힘) | `min_length=1` |
| `show_quiz` | `print("정답:", {quiz.answer})` | `{...}`가 **set**을 만들어 `정답: {'딕셔너리 이름 뒤에 ...'}`로 출력 | `print("정답:", quiz.answer)` |

* f-string 안의 `{변수}`와 그냥 `{변수}`는 다름. f-string 밖에서 중괄호로 값을 감싸면 집합(set) 리터럴이 됨

---

## 📝 오늘의 정리

1. **LangChain**은 프롬프트 → 모델 → 응답 처리를 부품(Runnable)으로 나누고, 모든 부품을 `invoke(입력)`으로 실행함
2. `PromptTemplate.from_template`(문자열), `ChatPromptTemplate.from_messages`(역할별 메시지)로 `{변수}` 템플릿을 만들고, `invoke(딕셔너리)`로 채움. `"user"`는 내부에서 `human` 메시지가 됨
3. `prompt | model | parser`: 왼쪽 결과를 오른쪽 입력으로 넘기는 체인. `StrOutputParser`는 `AIMessage`에서 텍스트만 꺼냄
4. `ChatOpenAI(model, temperature, max_completion_tokens, timeout, max_retries)`로 모델 설정. API 키는 `.env`에서 자동으로 읽음
5. **Pydantic** `BaseModel` + `Literal`·`Field(min_length)`·`ConfigDict(extra="forbid")`로 응답 규칙을 클래스로 선언
6. `PydanticOutputParser`는 클래스로 ① 모델에게 줄 JSON Schema 안내(`get_format_instructions`)를 만들고 ② 응답을 검증해 객체로 돌려줌. 실패하면 `OutputParserException`
7. `prompt.partial()`로 매번 같은 변수(형식 안내)를 미리 채움
8. 필드 여러 개를 함께 봐야 하는 규칙은 `@model_validator(mode="after")`에서 `ValueError`로 검증
9. **RAG**: 관련 문서를 검색해 LLM의 Context로 넣고 답을 생성. 첫 단계는 원본을 **`Document`(`page_content` + `metadata`)**로 만드는 것
10. `TextLoader(...).load()`, `pypdf`의 `PdfReader`로 TXT·PDF를 읽고, 출처(`source`)·페이지(`page`)를 metadata에 남겨 답변 근거로 씀
11. PDF 추출 텍스트는 줄바꿈·표 구조가 깨지므로 결과를 직접 확인해야 함
12. Pylance가 못 잡는 오타(Pydantic 모델의 클래스 속성)와 경고만 남기고 무시되는 오타(`Field`의 잘못된 인자 이름)가 있음 → **실행하고 경고까지 읽기**

---

## 느낀 점

* 어제 `json.loads` 뒤에 `set(item)`, `type(x) is int`, 범위 검사를 줄줄이 적었던 코드가 Pydantic 클래스 몇 줄로 줄어든 게 가장 크게 와닿았음. Spring에서 DTO에 `@NotBlank`·`@Size`를 붙이던 방식과 거의 같아서 금방 익숙해짐
* 같은 클래스가 **모델에게 보낼 형식 안내**와 **응답 검증**을 동시에 맡는다는 점이 좋았음. 프롬프트의 형식 설명과 검증 코드가 따로 놀다가 어긋날 일이 없음
* `|` 체인은 처음엔 마법 같았는데, 결국 "각 부품의 `invoke` 결과를 다음 부품에 넘긴다"는 단순한 규칙이었음. 부품을 하나씩 따로 실행해 보면 중간 결과가 보여서 디버깅이 쉬움
* 오타 때문에 오늘 꽤 고생했는데, `form_template`은 빨간 줄이 안 떴고 `min_lenght`는 아예 에러 없이 돌아갔다는 게 충격이었음. 특히 검증 규칙이 **조용히 빠지는** 오타는 테스트로 일부러 틀린 값을 넣어 보지 않으면 모를 수 있음
* PDF에서 표가 한 줄씩 풀려 나오는 걸 보고, RAG는 모델보다 **문서를 깨끗하게 읽어 오는 단계**가 먼저라는 걸 느낌. 다음에 배울 문서 분할(Splitter)에서 이 텍스트를 어떻게 다루는지 궁금함
