## Session 7: 휴먼 개입(Human-in-the-loop) 워크플로우

---

### 1. 승인 시스템

> **오늘의 목표**
> * 100% 자동화가 불안한 업무(결제, 계약, 대량 등록)에 **안전장치(Safety Guard)**를 만듭니다.
> * **Gmail SMTP**를 이용해 승인 요청 메일을 발송합니다.
> * **Wait 노드**를 활용해 사람이 버튼을 누를 때까지 워크플로우를 일시 정지시킵니다.
> 
> 


---

## 2. 실습 시나리오 및 흐름도

**시나리오:**
AI가 도면에서 추출한 데이터를 바로 DB에 넣지 않고, 담당자에게 메일을 보냅니다. 담당자가 **[승인]** 버튼을 클릭하면 구글 시트에 저장하고, **[반려]**하면 저장하지 않고 종료합니다.

**Workflow Flow:**
* `Trigger` → `데이터 처리(AI)` → **`Email(승인 요청)`** → **`Wait(승인 대기)`** → **`Switch(승인/반려 판단)`** → `DB 저장 또는 종료`

---

## 3. 단계별 상세 가이드

### Step 0: 새 workflow 만들기

**도면** 을 분석하는 workflow 를 만듭니다. Session 6 에서 만든 workflow 를 복사한 뒤 수정합니다.

* **Duplicate** 이전 workflow 의 오른쪽 세점을 누른 뒤, `Duplicate` 를 수행합니다.
* **구글 시트** 도면 분석을 위한 새 구글 시트를 공유한 구글 드라이브 안에 준비합니다.
    * "Drawing link" 를 첫번째 행의 임의의 열에 입력합니다.
* **Basic LLM Chain** 노드를 수정합니다.
  * `Require Specific Output Format` 는 끄고, `Structured Output Parser` 는 제거합니다.
  * `system message` 는 `너는 도면 OCR 텍스트를 단일 계층(Flat)의 Key-Value JSON으로 변환하는 데이터 정제 엔진이야.` 라고 입력합니다.
  * `user message` 는 아래와 같습니다.
  ```
  # Task
  제공된 여러 들여쓰기로 분리된 텍스트에서 **Target Keys**에 해당하는 값만 찾아 JSON으로 추출해.

  # Rules
  1. "Drawing link" 의 값은 "https://drive.google.com/file/d/{{ $('Download file').item.json.id }}" 이다.
  2. Target Keys:
   - "Drawing link","Product code","Sectional area (mm²)","Approximate mass (kg/m)","Drawing title","File name","Drawing number","Date","Page","Scale".
  3. Noise Filter: 위 키에 해당하지 않는 단순 치수(예: 159.5 mm)나 라벨은 절대 포함하지 마.
  4. Key-Value 는 다른 들여쓰기 일 수 없음: Key와 Value가 서로 다른 줄인 경우, 반드시 Value는 Key의 바로 아랫줄이여야만 하고 같은 들여쓰기여야만 한다.

  # Input Text
  {{ $json.text }}
  ```
* **OpenAI Chat Mode**
  * `Timeout:` 300000
  * `Response Format:` JSON
 

### Step 1: 승인 요청 이메일 보내기 (SendAndWait email Node)

**SendAndWait email** 노드를 추가합니다.
* **Node 추가** `SendAndWait email`
* **From Email:** 본인 이메일
* **To Email:** 본인 이메일 (테스트용)
* **Subject:** `[n8n] {{ $json['Product code'] }} 도면 처리 승인 요청건`
* **Message:** (Expression)

   {% raw %}
   ```js
   <div style="font-family: sans-serif; border: 1px solid #ddd; padding: 20px; border-radius: 10px;">
     <h2 style="color: #333;">📋 승인 요청</h2>
     <p>AI가 분석한 도면의 결과는 아래와 같습니다. DB에 저장을 승인하시겠습니까?</p>
   
     <ul style="background-color: #f9f9f9; padding: 15px; list-style: none; text-align: left; margin: 0 auto; max-width: 100%;">
       {{ Object.entries($json).map(([key, value]) => `<li><b>${key}:</b> ${value}</li>`).join('') }}
     </ul>   
   </div>
   ```
   {% endraw %}

* **Type of Approval:** `Approve and Disapprove`

### Step 2: 승인/반려 판단하기 (Switch Node)

사람이 `approve` 링크를 눌렀는지, `reject` 링크를 눌렀는지 판단합니다.

* **Node 추가:** `Switch`
* **Mode:** `Expression`
* **Number of Outputs:** `2`
* **Output Index:** 에서 승인 여부를 검사합니다.
    ```
    {{ $json.data.approved == true }}
    ```


### Step 3: Code 노드
* **Switch 노드** 의 **출력 1(승인)**에 `Code` 노드를 연결합니다.
* LLM 이 출력했던 내용을 다시 가져오는 역할을 합니다.
* **JavaScript:**
  ```
  return $items("Basic LLM Chain");
  ```


### Step 4: DB 저장 (Google Sheets)

* `Code` 노드 다음에 `Google Sheets` 노드를 연결합니다.
* **Append or update row in sheet** 노드에서 `Document` 와 `Sheet` 를 도면용으로 만든 파일과 시트를 선택합니다.
    * **Mapping Column Mode** 는 `Map Automatically` 로 선택합니다.
    * **Column to match on** 는 `Drawing link` 로 선택합니다.

* **출력 0(반려)**에는 아무것도 연결하지 않습니다.




---

## 4. 최종 실습 체크리스트

1. [ ] **앱 비밀번호**를 발급받아 n8n SMTP Credential에 넣었나요?
2. [ ] 메일을 받고 승인(Approve) 클릭했을 때, n8n 워크플로우가 **초록색(Success)**으로 바뀌며 진행되나요?

---

이 가이드로 진행하시면 복잡한 인증 절차 없이 **"메일 발송 -> 승인 대기 -> 처리"**라는 고급 워크플로우를 구현하실 수 있습니다.


---

### 🎓 3일간의 교육 과정 마무리 (Wrap-up)

**과정 요약**

1. **1일차 (Foundation):** n8n 설치, 노드/워크플로우 개념, 데이터 흐름(JSON)의 이해.
2. **2일차 (Intelligence):** LLM(ChatGPT) 연동, 프롬프트 엔지니어링, 도구를 사용하는 Agent 구현.
3. **3일차 (Advanced):** 비정형 데이터(이미지) 처리, DB 적재, 그리고 **사람의 결정이 포함된(HITL) 완벽한 파이프라인** 구축.

**수강생들에게 전하는 말**

> "오늘 실습한 '승인 버튼' 하나가 여러분의 자동화를 **신뢰할 수 있는 시스템**으로 바꿔줍니다.
> 이제 여러분은 **'보고(Vision)', '생각하고(LLM)', '사람과 소통하는(Gmail+Wait)'** 완전한 형태의 AI 동료를 만들 수 있게 되었습니다.
> 돌아가셔서 여러분 현업의 문제를 이 n8n 레고 블록들로 멋지게 해결해 보시기 바랍니다."

---
