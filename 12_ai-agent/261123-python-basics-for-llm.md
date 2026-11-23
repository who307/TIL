# Python 기초: 자료형·컬렉션·제어문·컴프리헨션·함수·JSON·예외 처리 · LLM 응답 다루기 · OpenAI API 첫 호출 (TIL)

> 실습 환경: Python 3, `python-dotenv`, `openai` 패키지
>
> AI Agent 과정의 첫 시간. Java·JavaScript로 배운 개념을 Python 문법으로 다시 정리하고, LLM에 보낼 **프롬프트(JSON)를 만들고 응답(JSON)을 검증**하는 연습까지 진행함. 마지막에 OpenAI API를 직접 호출하는 `bot.py`를 만듦. (`02_numpy`는 파일만 만들어 둔 상태)

## 0. 오늘의 흐름

| 파일 | 내용 |
| :--- | :--- |
| `01_basic_and_collections.py` | 변수·f-string, 리스트(`append`·`insert`·`pop`), 딕셔너리(`get`) |
| `02_string_and_control_flow.py` | 슬라이싱, 들여쓰기 블록, `if`·`elif`·`else`, `for in` |
| `03_comprehension.py` | 리스트 컴프리헨션 |
| `04_functions.py` | `def`, 타입 힌트, 기본값, 키워드 인자 |
| `05_json_and_exceptions.py` | `json.dumps`·`json.loads`, `try`·`except` |
| `06_python_for_llm.py` | LLM 프롬프트 만들기, 응답 JSON 검증 |
| `03_gpt/01_gpt-basic/bot.py` | `.env`로 API 키 읽기, OpenAI Chat Completions 호출 |

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
* 테스트용 응답을 `RESPONSE_EXAMPLES` 딕셔너리(정상 / 깨진 JSON / 필수 필드 누락)로 모아 둠

### `if __name__ == "__main__":`

```python
def main():
    prompt = build_prompt(FEEDBACKS)
    print(prompt)

if __name__ == "__main__":
    main()
```

* 파일을 **직접 실행**하면 `__name__`이 `"__main__"`이라 `main()`이 실행되고, 다른 파일에서 **`import`하면** 실행되지 않음
* Java의 `public static void main`처럼 시작점을 명시하는 관용구. 다른 파일이 `build_prompt`만 가져다 쓸 수 있게 해 줌
* 대문자 이름(`FEEDBACKS`, `SAMPLE_REPLY`)은 "바꾸지 않는 상수"라는 관례 (Java `static final`처럼 막아 주지는 않음)

---

## 7. OpenAI API 첫 호출 (`bot.py`)

```python
import os
from dotenv import load_dotenv
from openai import APIError, OpenAI

load_dotenv()                                    # .env → 환경변수
MODEL = os.getenv("OPENAI_MODEL", "gpt-5-mini")  # 없으면 기본 모델

def get_reply(client, messages):
    response = client.chat.completions.create(model=MODEL, messages=messages)
    return response.choices[0].message.content

def main():
    api_key = os.getenv("OPENAI_API_KEY")
    if not api_key or api_key == "sk-your-key-here":
        print("OPENAI_API_KEY 가 없습니다. .env 파일에 키를 넣어 주세요.")
        return

    client = OpenAI(api_key=api_key)
    messages = [{"role": "user", "content": "python 리스트가 뭐에요?"}]
    try:
        print("Bot:", get_reply(client, messages))
    except APIError as e:
        print("API 호출 실패:", e)
```

* **API 키는 코드에 쓰지 않고 `.env`에** 두고 `load_dotenv()` + `os.getenv()`로 읽음. Spring에서 `application.yml`에 `${JWT_SECRET}`으로 환경 변수를 쓴 것과 같은 이유
* `.env`는 `.gitignore`에 추가해서 저장소에 올라가지 않게 함 (오늘 `.gitignore`에 `.env`, `__pycache__/`, `*.pyc` 추가)
* 키가 없거나 예시 값(`sk-your-key-here`) 그대로면 호출 전에 안내하고 끝냄 → 의미 없는 401 에러 대신 원인을 바로 알려 줌
* `messages`: 대화 기록을 `{"role": ..., "content": ...}` 딕셔너리의 리스트로 보냄 (`user`·`assistant`·`system`). 6장에서 연습한 "딕셔너리의 리스트" 그대로
* 응답 텍스트 위치: `response.choices[0].message.content` (여러 후보 중 첫 번째)
* 네트워크·인증·한도 초과 같은 실패는 `APIError`로 잡음

---

## ⚠️ 코드에서 본 것

| 위치 | 내용 |
| :--- | :--- |
| `06_python_for_llm.py` `MISSING_FIELD_REPLY` | 이름은 "필수 필드 누락"인데 항목에 `id`·`sentiment`·`score`·`key_reason`이 **모두 들어 있음** → `parse_response`가 실패하지 않고 통과함. 예를 들어 `"key_reason": ...`을 빼야 의도대로 `ValueError("필수 필드가 없습니다: key_reason")`이 남 |
| `06_python_for_llm.py` `main()` | 지금은 프롬프트 출력만 하고 `RESPONSE_EXAMPLES`로 `parse_response`를 돌려 보는 부분은 아직 없음. `for name, reply in RESPONSE_EXAMPLES.items():` + `try`/`except (json.JSONDecodeError, ValueError)`로 세 경우를 확인할 수 있음 |
| `02_string_and_control_flow.py` | 출력 문구 오타 `점쉬` → `점수` |

---

## 📝 오늘의 정리

1. Python은 타입 선언 없이 값으로 타입이 정해지고, 블록은 **콜론 + 들여쓰기**로 구분함
2. 리스트는 `append`·`insert`·`pop`, 딕셔너리는 `d['key']`(없으면 `KeyError`)와 `d.get('key', 기본값)`(없으면 기본값)을 구분해서 씀
3. 슬라이싱 `[시작:끝]`은 끝을 포함하지 않고, 컴프리헨션 `[결과 for x in 목록 if 조건]`은 filter + map을 한 줄로 씀
4. 함수는 타입 힌트(`: str`, `-> str`)·기본값·키워드 인자를 지원함. 타입 힌트는 실행 시 검사하지 않음
5. `json.dumps`(→ 문자열, 한글은 `ensure_ascii=False`)·`json.loads`(문자열 →), 깨진 JSON은 `json.JSONDecodeError`로 잡음
6. LLM 응답은 그냥 문자열이므로 **JSON 문법 → 구조(`isinstance`) → 필수 키(`in`)** 순서로 검증하고, 틀리면 `raise ValueError`
7. `if __name__ == "__main__":`으로 직접 실행할 때만 `main()`을 돌림
8. OpenAI 호출은 `.env`의 키를 `load_dotenv` + `os.getenv`로 읽고, `messages` 리스트를 보내 `choices[0].message.content`로 답을 받음

---

## 느낀 점

* Java·JS에서 배운 개념이 거의 그대로라 문법만 바꿔 끼우는 느낌이었음. 특히 컴프리헨션은 처음엔 낯설었지만 for문 버전과 나란히 보니 읽는 순서가 바로 이해됨
* LLM이라고 하면 프롬프트만 잘 쓰면 될 줄 알았는데, 실제 코드에서는 **응답을 믿지 않고 검증하는 부분**이 더 중요해 보임. Spring에서 요청 값을 `@Valid`로 검사했던 것처럼, AI 응답도 "외부에서 들어온 입력"으로 다뤄야 한다는 점이 새로웠음
* API 키를 `.env`로 빼고 `.gitignore`에 넣는 건 지난 Security·JPA 때 DB 비밀번호를 다룬 것과 같은 습관이라, 언어가 바뀌어도 보안 원칙은 같다는 걸 느낌
