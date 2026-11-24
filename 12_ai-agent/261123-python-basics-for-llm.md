# Python 기초: 자료형·컬렉션·제어문·컴프리헨션·함수·JSON·예외 처리 · LLM 응답 검증 · NumPy 벡터·코사인 유사도 · OpenAI 대화 봇 (TIL)

> 실습 환경: Python 3, `numpy`, `python-dotenv`, `openai` 패키지
>
> AI Agent 과정의 첫 시간. Java·JavaScript로 배운 개념을 Python 문법으로 다시 정리하고, LLM에 보낼 **프롬프트(JSON)를 만들고 응답(JSON)을 검증**하는 연습까지 진행함. 이어서 NumPy로 벡터와 **코사인 유사도**를 계산해 보고, 마지막에 OpenAI API로 **대화 이력을 기억하는 콘솔 챗봇**(`bot.py`)을 만듦

## 0. 오늘의 흐름

| 파일 | 내용 |
| :--- | :--- |
| `01_basic_and_collections.py` | 변수·f-string, 리스트(`append`·`insert`·`pop`), 딕셔너리(`get`) |
| `02_string_and_control_flow.py` | 슬라이싱, 들여쓰기 블록, `if`·`elif`·`else`, `for in` |
| `03_comprehension.py` | 리스트 컴프리헨션 |
| `04_functions.py` | `def`, 타입 힌트, 기본값, 키워드 인자 |
| `05_json_and_exceptions.py` | `json.dumps`·`json.loads`, `try`·`except` |
| `06_python_for_llm.py` | LLM 프롬프트 만들기, 응답 JSON 검증, 예외 종류별 처리, `dict.items()` |
| `02_numpy/01_vectors_for_gpt.py` | NumPy 배열 연산, 벡터, 내적·길이, 코사인 유사도 |
| `03_gpt/01_gpt-basic/bot.py` | `.env`로 API 키 읽기, `system`·`user`·`assistant` 메시지, 대화 이력, 응답 검증, 콘솔 대화 루프 |

---

## 1. 변수와 컬렉션

```python
team_name = '새싹'   # str
price = 123_456      # int, 밑줄로 자릿수 구분 (Java도 123_456 가능)

print(f"팀 이름: {team_name}")   # f-string
```

* 타입 선언 없이 값을 넣으면 그 값의 타입이 됨 (Java의 `String name = ...`이 필요 없음)
* f-string은 JS 템플릿 리터럴 `` `팀 이름: ${teamName}` ``과 같은 역할

### 리스트 (Java `ArrayList`, JS 배열)

```python
scores = [88, 90, 67]
scores.append(100)      # 끝에 추가        → [88, 90, 67, 100]
scores.insert(2, 75)    # 2번 인덱스에 삽입 → [88, 90, 75, 67, 100]
last = scores.pop()     # 맨 뒤를 꺼내고 제거 → last = 100
```

### 딕셔너리 (Java `HashMap`, JS 객체)

```python
user = {'name': '홍길동', 'age': 30}
user['job'] = '의적'    # 없는 키 → 추가
user['age'] = 33        # 있는 키 → 덮어쓰기

user['email']                      # ❌ KeyError
user.get('email', '정보 없음')      # ✅ 키가 없으면 기본값
```

| 조회 방법 | 키가 없을 때 | 언제 쓰나 |
| :--- | :--- | :--- |
| `d['key']` | `KeyError` 예외 | 반드시 있어야 하는 값 |
| `d.get('key', 기본값)` | 기본값 (생략하면 `None`) | 없을 수도 있는 값 |

* Java `map.getOrDefault(key, default)`와 같은 개념. JS는 없는 키를 읽으면 `undefined`가 나오고 에러가 나지 않는데, Python은 **대괄호로 읽으면 에러**가 난다는 점이 다름

```python
users = [{'name': '홍길동'}, {'name': '신사임당', 'job': '화가'}]
users[1]['name']                     # 신사임당
users[0].get('job', '정보 없음')      # 정보 없음
```

* "딕셔너리의 리스트"는 API 응답 JSON(`[{...}, {...}]`)과 모양이 같아서 LLM 응답을 다룰 때 계속 쓰임

---

## 2. 문자열 슬라이싱과 제어문

```python
txt = 'helloworld'
txt[:5]    # 'hello'  시작 생략 = 처음부터, 끝 인덱스(5)는 포함 안 함
txt[5:]    # 'world'  끝 생략 = 끝까지
```

* `[시작:끝]`은 Java `substring(시작, 끝)`처럼 끝을 포함하지 않음

```python
if score >= 90:
    grade = 'A'
elif score >= 80:      # else if → elif
    grade = 'B'
else:
    grade = 'F'

for food in ["햄버거", "피자", "보쌈"]:   # Java 향상된 for문(for (String food : foods))
    if food != "피자":
        print(f"오늘의 메뉴: {food}")
```

* 중괄호 `{}` 대신 **콜론(`:`) + 들여쓰기**로 블록을 구분함. 들여쓰기가 틀리면 문법 오류(`IndentationError`)
* 조건식에 괄호가 필요 없고, 조건 연결은 `&&`·`||`·`!` 대신 `and`·`or`·`not`

---

## 3. 리스트 컴프리헨션

다섯 글자 이상인 과일만 골라 대문자로 바꾸기:

```python
fruits = ["orange", "apple", "banana", "kiwi"]

# for문 버전
long_fruits = []
for fruit in fruits:
    if len(fruit) >= 5:
        long_fruits.append(fruit.upper())

# 컴프리헨션 버전
compact = [fruit.upper() for fruit in fruits if len(fruit) >= 5]
# ['ORANGE', 'APPLE', 'BANANA']
```

* 읽는 순서: `[ 결과 for 변수 in 목록 if 조건 ]` → "목록에서 조건에 맞는 것만 결과로 바꿔 담는다"
* JS의 `fruits.filter(f => f.length >= 5).map(f => f.toUpperCase())`, Java 스트림의 `filter().map().toList()`와 같은 일을 한 줄로 함

---

## 4. 함수와 타입 힌트

```python
def create_greeting(name: str, age: int = 20) -> str:
    return f"안녕하세요, {age}세 {name}님."

create_greeting("홍길동", 30)               # 위치 인자
create_greeting("유관순")                   # age 생략 → 기본값 20
create_greeting(age=45, name="신사임당")    # 키워드 인자: 순서 바꿔도 됨
```

| 문법 | 의미 |
| :--- | :--- |
| `name: str` | 매개변수 타입 힌트 |
| `age: int = 20` | 기본값이 있는 매개변수 (기본값 있는 것은 뒤쪽에) |
| `-> str` | 반환 타입 힌트 |
| `name="..."` | 키워드 인자. 이름으로 넘기므로 순서 무관 |

* 타입 힌트는 **실행할 때 검사하지 않음**. `create_greeting(123)`도 그대로 실행됨. 코드를 읽는 사람과 IDE·검사 도구를 위한 표시
* Java는 기본값이 없어서 오버로딩으로 해결했는데, Python은 기본값과 키워드 인자로 한 함수에서 처리함

---

## 5. JSON 변환과 예외 처리

| 함수 | 방향 | 대상 |
| :--- | :--- | :--- |
| `json.dumps(obj)` | Python → JSON **문자열** | 요청 본문 만들기 |
| `json.loads(str)` | JSON 문자열 → Python | 응답 읽기 |
| `json.dump(obj, f)` / `json.load(f)` | 위와 같음 | **파일**에 쓰기/읽기 |

* 외우는 법: 끝의 **s = string**

```python
feedback = {"id": 1, "text": "실습이 도움이 됐어요."}
json.dumps(feedback, ensure_ascii=False)
# '{"id": 1, "text": "실습이 도움이 됐어요."}'
```

* `ensure_ascii=False`가 없으면 한글이 `실습...`처럼 유니코드 이스케이프로 바뀜. 프롬프트에 넣을 때는 한글 그대로가 읽기 좋고 토큰도 적게 듦

```python
result = json.loads('{"id": 1, "score": 75}')
result["score"]   # 75 (dict로 바뀌어서 키로 조회)

try:
    json.loads('{"id": 1, "score": 75')       # 닫는 괄호 없음
except json.JSONDecodeError:
    print("응답을 JSON으로 읽을 수 없습니다.")
```

* `try`·`except`는 Java `try`·`catch`와 같음. 잡을 예외 타입을 구체적으로 적는 것도 같음 (`except Exception`으로 다 잡지 않기)

---

## 6. LLM에 쓰는 Python: 프롬프트 만들고 응답 검증하기

수강평 3개를 LLM에 보내 감정(`sentiment`)·점수(`score`)·이유(`key_reason`)를 받는 상황을 가정함. 아직 API는 호출하지 않고 **보낼 프롬프트**와 **받을 응답 처리**만 만듦.

### 프롬프트 만들기

```python
def build_prompt(feedbacks):
    feedbacks_json = json.dumps(feedbacks, ensure_ascii=False)
    return f"""
            아래의 수강평을 분석해 JSON 객체만 출력하세요. 최상위 키는 "items"입니다.
            각 항목은 id, sentiment, score, key_reason을 포함하세요.
            id는 원본과 동일하게, sentiment는 positive/negative/mixed중 하나,
            score는 0~100 정수, key_reason은 한 문장으로 작성하세요.
            수강평: {feedbacks_json}
            """
```

* `f"""..."""`: 여러 줄 f-string (Java 텍스트 블록 `"""` + 변수 삽입)
* 프롬프트에 **출력 형식(키 이름·값 범위·타입)을 구체적으로** 적어야 코드로 파싱할 수 있는 응답이 옴
* 입력 데이터도 `json.dumps`로 넣어서 모델이 항목 경계와 `id`를 정확히 구분하게 함

### 응답 검증하기

LLM 응답은 "문자열"일 뿐이라 **형식을 믿으면 안 됨**. 단계별로 확인함.

```python
def parse_response(reply):
    payload = json.loads(reply)                              # ① JSON인가 (아니면 JSONDecodeError)
    if not isinstance(payload, dict) or "items" not in payload:
        raise ValueError("응답은 items 키가 있는 JSON 객체여야 합니다.")   # ② 최상위 구조

    items = payload["items"]
    if not isinstance(items, list):
        raise ValueError("items는 리스트여야 합니다.")       # ③ items 타입

    required_keys = ["id", "sentiment", "score", "key_reason"]
    for item in items:
        if not isinstance(item, dict):
            raise ValueError("각 항목은 딕셔너리여야 합니다.")
        for key in required_keys:
            if key not in item:
                raise ValueError(f"필수 필드가 없습니다: {key}")   # ④ 필수 필드
    return items
```

| 검사 | 문법 | 실패하는 응답 예 |
| :--- | :--- | :--- |
| JSON 문법 | `json.loads` | `'{"items": []'` (닫는 괄호 없음) |
| 타입 | `isinstance(값, dict / list)` | `'[1, 2]'`, `'{"items": "없음"}'` |
| 키 존재 | `"키" in 딕셔너리` | `key_reason`이 빠진 항목 |

* `isinstance(x, dict)` = Java `x instanceof Map`
* `raise ValueError(...)` = Java `throw new IllegalArgumentException(...)`
* 잘못된 응답이면 **빨리 예외로 멈추고**, 호출하는 쪽에서 재요청하거나 사용자에게 알리게 함. 이상한 값이 화면·DB까지 흘러가는 것보다 나음
### 예외 종류별로 처리하기

```python
def show_result(reply):
    try:
        items = parse_response(reply)
    except json.JSONDecodeError:            # JSON 문법 자체가 깨짐
        print("응답을 JSON으로 읽을 수 없습니다")
        return
    except ValueError as error:             # JSON은 맞는데 형식이 틀림
        print("응답 형식 오류", error)
        return

    for item in items:
        print(f'{item["id"]}: {item["sentiment"]} / {item["score"]}점')
        print(f' 근거: {item["key_reason"]}')
```

* `except 예외 as error`: 예외 객체를 변수로 받아 메시지를 출력 (Java `catch (ValueError e)`)
* **`except` 순서가 중요함**: `json.JSONDecodeError`는 `ValueError`의 **하위 클래스**라서, `ValueError`를 먼저 쓰면 깨진 JSON도 거기서 잡혀 "형식 오류"로 나옴. Java에서 `catch`를 좁은 예외부터 쓰는 것과 같은 규칙
* 실패하면 `return`으로 바로 끝내고, 성공했을 때만 아래 출력으로 내려감 (조기 반환)

### 테스트 응답 3종 돌려 보기

```python
BROKEN_JSON = '{"items": []'      # 닫는 괄호 없음
MISSING_FIELD_REPLY = '{"items": [{"id": 1, "sentiment": "mixed", "key_reason": "..."}]}'   # score 없음

RESPONSE_EXAMPLES = {
    "정상 응답": SAMPLE_REPLY,
    "깨진 JSON": BROKEN_JSON,
    "필수 필드 누락": MISSING_FIELD_REPLY,
}

def main():
    print(build_prompt(FEEDBACKS))

    # 딕셔너리.items(): 키와 값을 한 쌍씩 꺼냄
    for name, reply in RESPONSE_EXAMPLES.items():
        print(f"\n[{name}]")
        show_result(reply)

if __name__ == "__main__":
    main()
```

```text
[정상 응답]
1: mixed / 75점
 근거: 진도는 빠르지만 실습이 도움이 됩니다.
2: negative / 35점
...
[깨진 JSON]
응답을 JSON으로 읽을 수 없습니다
[필수 필드 누락]
응답 형식 오류 필수 필드가 없습니다: score
```

* `for name, reply in d.items()`: Java `for (Map.Entry<K, V> e : map.entrySet())`, JS `Object.entries(obj)`와 같음. 튜플 `(키, 값)`을 두 변수로 나눠 받는 것(언패킹)
* 실제 API를 부르지 않고도 **정상·깨진 JSON·필드 누락** 세 경우의 처리를 미리 확인할 수 있음

### `if __name__ == "__main__":`

* 파일을 **직접 실행**하면 `__name__`이 `"__main__"`이라 `main()`이 실행되고, 다른 파일에서 **`import`하면** 실행되지 않음
* Java의 `public static void main`처럼 시작점을 명시하는 관용구. 다른 파일이 `build_prompt`만 가져다 쓸 수 있게 해 줌
* 대문자 이름(`FEEDBACKS`, `SAMPLE_REPLY`)은 "바꾸지 않는 상수"라는 관례 (Java `static final`처럼 막아 주지는 않음)

---

## 7. NumPy: 벡터와 코사인 유사도

NumPy는 숫자 배열을 만들고 **배열 전체를 한 번에 계산**하는 라이브러리. 임베딩(텍스트를 숫자 벡터로 바꾼 것)을 비교할 때 이 계산을 씀.

```python
import numpy as np

left, right = [1, 2], [3, 4]
left + right                        # [1, 2, 3, 4]  리스트의 +는 이어 붙이기
np.array(left) + np.array(right)    # [4 6]         배열의 +는 같은 위치끼리 계산
```

* `import numpy as np`: 모듈에 별칭을 붙여 `np.`로 씀 (관례)
* 파이썬 리스트로 같은 계산을 하려면 `[a + b for a, b in zip(left, right)]`처럼 반복문이 필요한데, NumPy는 연산자 하나로 끝남

### 벡터로 영화 표현하기

순서를 `[액션, 스릴러, 코미디, 감동]`으로 정해 두고 각 영화를 숫자로 표시함.

```python
moana        = np.array([0, 0, 0, 1])   # 감동
toy_story    = np.array([0, 0, 1, 1])   # 코미디 + 감동
action_movie = np.array([1, 0, 0, 0])   # 액션
```

* **벡터**: 같은 기준과 순서로 나열한 숫자 목록. 순서(기준)가 같아야 서로 비교할 수 있음

### 코사인 유사도

```python
def cosine_similarity(a, b):
    dot = np.dot(a, b)                # 내적: 같은 위치끼리 곱해서 모두 더함
    length_a = np.linalg.norm(a)      # 벡터의 길이 (원점에서의 거리)
    length_b = np.linalg.norm(b)
    return dot / (length_a * length_b)
```

$$\cos\theta = \frac{a \cdot b}{\lVert a \rVert \, \lVert b \rVert}$$

| 비교 | 내적 | 길이 곱 | 유사도 | 의미 |
| :--- | :---: | :---: | :---: | :--- |
| 모아나 ↔ 모아나 | 1 | 1 × 1 | **1.0** | 방향이 완전히 같음 |
| 모아나 ↔ 토이스토리 | 1 | 1 × √2 | **약 0.707** | "감동"이 겹쳐서 꽤 비슷함 |
| 모아나 ↔ 액션영화 | 0 | 1 × 1 | **0.0** | 겹치는 특징이 없음 |

* 두 벡터가 이루는 **각도(방향)**를 비교하는 것이라 값의 크기보다 "어떤 특징이 함께 있는지"를 봄. 1에 가까울수록 비슷함
* 길이로 나누는 이유: 특징 값이 커도 방향이 같으면 같은 점수가 나오게 하려고 (정규화)
* LLM에서는 문장을 임베딩 벡터로 바꾼 뒤 이 방식으로 **의미가 비슷한 문서를 찾음** (검색·추천·RAG의 기초)

---

## 8. OpenAI 대화 봇 (`bot.py`)

처음엔 질문 하나를 보내고 끝나는 코드였는데, **질문을 계속 받고 앞의 대화를 기억하는 콘솔 챗봇**으로 바꿈.

### 구조

```text
main() ─ load_dotenv, 키 확인, OpenAI 클라이언트 생성
  └─ chat_loop(reply_to)          입력 받기 · 메시지 조립 · 이력 관리
        └─ reply_to(messages)     = live_reply → request_reply(client, messages)  실제 API 호출
                                    (sample_reply로 바꾸면 API 없이 테스트)
```

### API 호출과 응답 검증

```python
def request_reply(client, messages):
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        max_completion_tokens=500,   # 한 답변에 쓸 수 있는 토큰 수 상한
        temperature=0.7,             # 답변의 다양성 (낮을수록 일정, 높을수록 다양)
    )
    choice = response.choices[0]
    reply = (choice.message.content or "").strip()
    if choice.finish_reason != "stop" or not reply:
        raise ValueError("완료된 텍스트 응답을 받지 못했습니다.")
    return reply
```

| 항목 | 의미 |
| :--- | :--- |
| `max_completion_tokens` | 답변 길이 상한. 비용과 응답 시간을 제한함 |
| `temperature` | 0에 가까우면 매번 비슷한 답, 높으면 표현이 다양해짐 |
| `finish_reason` | `"stop"` = 정상 종료. `"length"`면 토큰 상한에 걸려 **답이 중간에 잘린 것** |
| `content or ""` | `content`가 `None`일 수 있어서 빈 문자열로 바꾼 뒤 `.strip()` |

* 6장과 같은 원칙: **응답을 그대로 믿지 않고**, 잘린 답·빈 답이면 `ValueError`로 처리함

### 대화 루프와 이력

```python
def chat_loop(reply_to):
    history = []
    system = {"role": "system", "content": "당신은 친절한 강사입니다."}

    while True:
        try:
            question = input("You:").strip()
        except (EOFError, KeyboardInterrupt):    # Ctrl+D / Ctrl+C
            break
        if question.lower() in {"exit", "quit"}:
            break
        if not question:                         # 빈 문자열은 False
            continue

        user = {"role": "user", "content": question}
        messages = [system] + history + [user]
        try:
            reply = reply_to(messages)
        except (APIError, ValueError) as error:
            print("응답 실패", error)
            continue
        print("Bot:", reply)
        history = (history + [user, {"role": "assistant", "content": reply}])[-10:]
    print("종료합니다")
```

| role | 역할 |
| :--- | :--- |
| `system` | 모델의 역할·말투를 정하는 지시 ("친절한 강사"). 매 요청 맨 앞에 붙임 |
| `user` | 사용자의 질문 |
| `assistant` | 모델이 했던 답변. 다시 보내야 앞의 대화를 "기억"함 |

* **API는 이전 대화를 기억하지 않음(stateless)**. 그래서 매번 `system + 지금까지의 이력 + 새 질문`을 통째로 보냄. JWT를 매 요청에 실어 보낸 것처럼, 상태는 클라이언트가 들고 다님
* `[-10:]`: 슬라이싱으로 **최근 10개 메시지(5번의 문답)만** 남김. 이력이 길어질수록 토큰(비용)이 늘고 모델의 입력 한도를 넘을 수 있어서
* 실패했을 때는 `continue`로 넘어가 이력에 넣지 않음 → 답을 못 받은 질문이 이력에 남지 않음
* `in {"exit", "quit"}`: 중괄호에 값만 있으면 **set(집합)**. 포함 여부 확인용
* `except (EOFError, KeyboardInterrupt)`: 튜플로 여러 예외를 한 번에 잡음 (Java `catch (A | B e)`)

### `main()`과 함수를 값으로 넘기기

```python
def main():
    load_dotenv()
    if not os.getenv("OPENAI_API_KEY"):
        raise ValueError(".env에 OPENAI_API_KEY를 설정하세요.")

    with OpenAI(timeout=20.0, max_retries=0) as client:
        def live_reply(messages):
            return request_reply(client, messages)

        chat_loop(live_reply)
```

* **API 키는 코드에 쓰지 않고 `.env`에** 두고 `load_dotenv()`로 환경 변수에 올림. `OpenAI()`는 환경 변수 `OPENAI_API_KEY`를 자동으로 읽음. Spring에서 `${JWT_SECRET}`을 환경 변수로 뺀 것과 같은 이유
* `.env`는 `.gitignore`에 추가해서 저장소에 올라가지 않게 함 (`.gitignore`에 `.env`, `__pycache__/`, `*.pyc` 추가)
* `with ... as client:` = Java try-with-resources. 블록이 끝나면 클라이언트 연결을 자동으로 닫음
* `timeout=20.0`(20초 넘으면 포기), `max_retries=0`(실패 시 자동 재시도 안 함 → 실습에서 실패 원인을 바로 보기 위해)
* `chat_loop`는 **함수(`reply_to`)를 인자로 받음**. 실제 호출은 `live_reply`, 테스트는 `sample_reply`를 넘기면 되므로 API 키·비용 없이 대화 루프를 확인할 수 있음 (JS 콜백, Java 함수형 인터페이스·람다와 같은 개념)
* `live_reply`는 `main` 안에서 정의해 바깥의 `client`를 기억하는 **클로저**. `chat_loop`는 `client`를 몰라도 됨

---

## 9. 고친 것

어제 실습 코드에서 찾은 부분을 수정함.

| 위치 | 문제 | 수정 |
| :--- | :--- | :--- |
| `06_python_for_llm.py` `MISSING_FIELD_REPLY` | 이름은 "필수 필드 누락"인데 필드가 모두 있어서 검증을 통과함 | `score`를 빼서 `필수 필드가 없습니다: score`가 나오게 함 |
| `06_python_for_llm.py` `main()` | 프롬프트 출력만 하고 응답 검증은 실행하지 않음 | `show_result`와 `RESPONSE_EXAMPLES.items()` 반복 추가 |
| `02_string_and_control_flow.py` | 출력 문구 오타 `점쉬` | `점수` |

---

## 📝 오늘의 정리

1. Python은 타입 선언 없이 값으로 타입이 정해지고, 블록은 **콜론 + 들여쓰기**로 구분함
2. 리스트는 `append`·`insert`·`pop`, 딕셔너리는 `d['key']`(없으면 `KeyError`)와 `d.get('key', 기본값)`(없으면 기본값)을 구분해서 씀
3. 슬라이싱 `[시작:끝]`은 끝을 포함하지 않고, 컴프리헨션 `[결과 for x in 목록 if 조건]`은 filter + map을 한 줄로 씀
4. 함수는 타입 힌트(`: str`, `-> str`)·기본값·키워드 인자를 지원함. 타입 힌트는 실행 시 검사하지 않음
5. `json.dumps`(→ 문자열, 한글은 `ensure_ascii=False`)·`json.loads`(문자열 →), 깨진 JSON은 `json.JSONDecodeError`로 잡음
6. LLM 응답은 그냥 문자열이므로 **JSON 문법 → 구조(`isinstance`) → 필수 키(`in`)** 순서로 검증하고, 틀리면 `raise ValueError`. 받는 쪽에서는 하위 예외인 `JSONDecodeError`를 `ValueError`보다 먼저 잡음
7. `if __name__ == "__main__":`으로 직접 실행할 때만 `main()`을 돌리고, `dict.items()`로 키·값을 함께 반복함
8. NumPy 배열은 같은 위치끼리 계산하고, 코사인 유사도 = 내적 ÷ (길이 × 길이)로 두 벡터의 방향이 얼마나 비슷한지(1 = 같음, 0 = 무관) 구함. 임베딩 검색의 기초
9. OpenAI API는 대화를 기억하지 않으므로 `system + 이력 + 새 질문`을 매번 보내고, 이력은 `[-10:]`으로 최근 것만 유지함
10. 응답은 `finish_reason == "stop"`과 빈 문자열 여부까지 확인하고, 키는 `.env` + `load_dotenv`, 클라이언트는 `with`로 열고 닫음

---

## 느낀 점

* Java·JS에서 배운 개념이 거의 그대로라 문법만 바꿔 끼우는 느낌이었음. 특히 컴프리헨션은 처음엔 낯설었지만 for문 버전과 나란히 보니 읽는 순서가 바로 이해됨
* LLM이라고 하면 프롬프트만 잘 쓰면 될 줄 알았는데, 실제 코드에서는 **응답을 믿지 않고 검증하는 부분**이 더 중요해 보임. Spring에서 요청 값을 `@Valid`로 검사했던 것처럼, AI 응답도 "외부에서 들어온 입력"으로 다뤄야 한다는 점이 새로웠음
* 코사인 유사도가 "특징이 겹치는 정도"를 숫자 하나로 보여 준다는 게 직관적이었음. 영화 장르 4칸짜리 벡터가 실제로는 수백~수천 칸짜리 임베딩이 될 뿐이라고 생각하니 AI 검색이 어떻게 동작하는지 감이 옴
* 챗봇이 대화를 "기억"하는 게 사실은 이전 대화를 매번 다시 보내는 것이라는 점이 의외였음. 그래서 이력을 잘라 내야 비용과 한도를 관리할 수 있다는 것도 이해됨
* API 키를 `.env`로 빼고 `.gitignore`에 넣는 건 지난 Security·JPA 때 DB 비밀번호를 다룬 것과 같은 습관이라, 언어가 바뀌어도 보안 원칙은 같다는 걸 느낌
