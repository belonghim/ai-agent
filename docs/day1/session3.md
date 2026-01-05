
## Session 3: n8n 이것은 알고 시작하자 (기본 다지기 & Local LLM 연동)

### 1. 핵심 3요소: Trigger, Node, Flow

n8n은 복잡한 코딩 대신 **3가지 요소**의 조합으로 작동합니다.

* **Trigger (방아쇠):** "언제 실행할까?"
* 자동화의 시작점입니다. (예: 매일 아침 9시, 메일이 도착했을 때, 버튼을 눌렀을 때)


* **Node (작업자):** "무엇을 할까?"
* 데이터를 가공하거나 외부 도구와 통신하는 단위입니다. (예: AI에게 요약 시키기, 슬랙 보내기)


* **Flow (연결선):** "데이터의 이동 경로"
* 왼쪽 노드가 뱉어낸 결과물(Output)이 선을 타고 오른쪽 노드의 재료(Input)로 들어갑니다.



---

### 2. 주요 Trigger 비교

| 구분 | Webhook (웹훅) | Schedule (스케줄) | Polling (폴링) |
| --- | --- | --- | --- |
| **비유** | 초인종 누르기 | 알람 시계 | 우편함 수시로 확인 |
| **방식** | 외부에서 신호를 줌 (URL) | 정해진 시간에 실행 | 주기적으로 변화 감지 |
| **특징** | 실시간 반응, 가장 빠름 | 주기적 업무에 적합 | 자원 소모가 있음 |

---

### 3. 데이터 흐름의 이해 (JSON & Item) **(★가장 중요)**

#### ① 컨베이어 벨트 위의 택배 상자 (Item)

n8n의 데이터는 **'Item'**이라는 택배 상자에 담겨 컨베이어 벨트(선)를 타고 흐릅니다.

* **Item 1개** = 택배 상자 1개 (`{ "product": "Laptop", "price": 1200 }`)
* **Item 10개** = 택배 상자 10개

#### ② 자동 반복 처리 (Looping)

만약 Trigger에서 **100개의 데이터(Item)**가 들어왔다면?

* 다음 노드는 **알아서 100번 작동**합니다.
* 반복문(For-loop)을 짤 필요가 없습니다. 이것이 n8n이 강력한 이유입니다.

#### ③ JSON 구조와 Binary(파일)

* **JSON:** 글자, 숫자 데이터 (대부분의 데이터)
* **Binary:** 이미지, PDF, 엑셀 파일 등 (3일차 OCR 실습 때 다룰 예정)

```json
[
  { "json": { "product": "Laptop", "price": 1200 } },
  { "json": { "product": "Mouse", "price": 20 } }
]

```

---

### 4. [실습 Part 1] 데이터 조작과 변수 매핑 (Mapping)

데이터를 변형하면서 n8n의 **'Expression(수식)'** 기능을 익혀봅시다.
‘Expression’ 기능은 고정된 값이 아닌, 변수를 사용할 수 있게 해줍니다.

**Step 1. 가상의 데이터 만들기**

1. 새로운 Workflow 를 생성
2. `Manual Trigger` 노드 생성
3. `Code` (javascript) 노드 연결 및 아래 코드 붙여넣기

```javascript
return [
  { json: { product: "Laptop", price: 1200 } },
  { json: { product: "Mouse", price: 20 } },
  { json: { product: "Keyboard", price: 80 } }
];

```

4. `Execute Node`를 눌러 데이터 3개가 생성된 것을 확인.

**Step 2. 가격 인상하기 (Edit Fields 노드)**

1. `Edit Fields` (구 Set 노드) 연결.
2. **변수 연결 (Drag & Drop):**
* 왼쪽 `Input` 패널에서 `product` 값을 마우스로 잡아서
* 오른쪽 `Value` 입력창에 끌어다 놓습니다. (`{{ $json.product }}` 생성됨)
* 왼쪽 `Input` 패널에서 `price` 값을 마우스로 잡아서
* 오른쪽 `Value` 입력창에 끌어다 놓습니다. (`{{ $json.price }}` 생성됨)


3. **수식 적용:**
* 입력된 price 값 뒤에 `* 1.1`을 덧붙입니다.
* 최종 수식: `{{ $json.price * 1.1 }}`

**Step 3. 결과 확인**

* 노드를 실행하고 오른쪽 `Output` 패널에서 가격이 10% 인상되었는지 확인합니다.


---

### 5. [실습 Part 2] Local LLM 연결: AI에게 업무 시키기

앞서 계산된 데이터를 **Local LLM(Podman)**에게 보내 판매 문구를 작성해봅시다.

**Step 1. Basic LLM Chain 노드 추가**

1. `Edit Fields` 노드 뒤에 `Basic LLM Chain` 노드를 연결합니다.
* *설명: 가장 기초적인 AI 대화 노드입니다.*


2. **Prompt(질문) 입력:**
* Source for Prompt (User Message) 를 `Define below` 로 수정하고, 다음 내용을 작성하여 `product` 변수를 매핑합니다.
* `너는 마케팅 전문가야. {{ $json.product }} 제품을 팔기 위한 매력적인 한 줄 광고 문구를 한국어로 작성해줘.`



**Step 2. Model 연결 (OpenAI Chat Model 노드 활용)**

1. `Basic LLM Chain` 노드의 `Model` 입력 단자에 붙은 더하기(+) 아이콘을 클릭합니다.
2. 메뉴에서 **`OpenAI Chat Model`**을 선택하여 연결합니다.
* **OpenAI Chat Model**
    * **Model:** `/models/hf.google.gemma-3n-E4B`
* *핵심:* 우리는 OpenAI를 쓰지 않지만, Podman의 Local LLM이 OpenAI와 통신 방식(규격)이 같으므로 이 노드를 빌려서 사용합니다.
3. `Use Responses API:` 비활성화
4. `Options:`
  * `Timeout:` 300000 (응답 시간을 늘립니다.)



**Step 3. Credential(자격 증명) 설정 - ★중요**

1. `OpenAI Chat Model` 노드를 구성합니다.
2. `Credential to connect with`에서 **Create New**를 선택합니다.
3. **설정 값 입력:**
  * **API Key:** `any-key` (아무거나 입력해도 됩니다. Local이라 검증하지 않음)
  * **URL (Base URL):** `http://host.containers.internal:11000/v1` (또는 Podman에서 확인한 주소)
  * OpenAi Credential 의 이름을 gemma 라고 바꿉니다.
  * **Save**를 누릅니다.
4. `Model:` 클릭하면 자동으로 해당 자격증명을 통해 확인되는 모델을 선택할 수 있습니다.



**Step 4. 결과 확인**

1. 전체 Flow를 실행(`Execute Workflow`)합니다.
2. **결과 확인:** Laptop, Mouse, Keyboard 각각에 대해 AI가 서로 다른 광고 문구를 작성했는지 확인합니다.
* *성공 예시:* "최고의 성능, 당신의 비즈니스를 위한 Laptop!"



---

### 📝 1일차 요약 및 과제

**오늘 배운 것**

1. **Agent**는 혼자 일하지 않고 도구(Node)를 쓴다.
2. **n8n**은 들어온 데이터 개수(Item)만큼 알아서 반복해서 일처리를 한다. (Loop 불필요)
3. **Local LLM 연결:** `OpenAI Chat Model` 노드의 주소(URL)만 바꾸면 내 PC의 AI를 무료로 쓸 수 있다.

**내일 예고**

* 이 기초 위에 **복잡한 문서 처리(RAG)**를 도전합니다.
* "긴 회의록을 던져주면, AI가 요약하고, 중요 이슈만 뽑아서 나에게 보고하는" 진짜 에이전트 구축 실습.

---

