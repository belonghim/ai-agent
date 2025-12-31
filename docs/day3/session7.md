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

### Step 1: 승인 요청 이메일 보내기 (SendAndWait email Node)

**이메일(SMTP)** 노드를 추가합니다(또는 Slack). HTML을 지원하므로 예쁜 버튼을 만들 수 있습니다.

* **Node 추가** `Send Email` (n8n 기본 노드)
* **From Email:** 본인 이메일
* **To Email:** 본인 이메일 (테스트용)
* **Subject:** `[n8n] {{ $json['Product code'] }} 도면 처리 승인 요청건`
* **HTML Message:** (Expression)

```
<div style="font-family: sans-serif; border: 1px solid #ddd; padding: 20px; border-radius: 10px;">
  <h2 style="color: #333;">📋 승인 요청</h2>
  <p>AI가 분석한 도면의 결과는 아래와 같습니다. DB에 저장을 승인하시겠습니까?</p>

  <ul style="background-color: #f9f9f9; padding: 15px; list-style: none;">
    {{ Object.entries($json).map(([key, value]) => `<li><b>${key}:</b> ${value}</li>`).join('') }}
  </ul>

  <hr style="border: 0; border-top: 1px solid #eee; margin: 20px 0;">

  <a href="{{ $execution.resumeUrl }}/?action=approve&id={{ $('Loop Over Items').item.json.id }}" 
     style="background-color: #28a745; color: white; padding: 12px 24px; text-decoration: none; border-radius: 5px; font-weight: bold;">
     ✅ 승인 및 저장
  </a>
  
  &nbsp;&nbsp;&nbsp;

  <a href="{{ $execution.resumeUrl }}/?action=reject" 
     style="background-color: #dc3545; color: white; padding: 12px 24px; text-decoration: none; border-radius: 5px; font-weight: bold;">
     ❌ 반려
  </a>
</div>
```

> **💡 핵심 포인트:** `{{ $execution.resumeUrl }}`은 n8n이 자동으로 만들어주는 **"이 워크플로우를 깨우는 주소"**입니다. 뒤에 `/?action=approve`와 `/?action=reject`를 붙여서 동작을 구분합니다.


### Step 2: 사람 기다리기 (Wait Node)

이메일을 보내고 나면, n8n은 사람이 버튼을 누를 때까지 멈춰 있어야 합니다.

* **Node 추가:** `Wait`
* **Resume Strategy:** `On Webhook Call` (웹훅 신호가 오면 깨어남)
* **Respond to Webhook:** `Node Output` (브라우저에 "처리되었습니다" 메시지 표시)
* **Authentication:** `None` (실습 편의상)
* **Suffix:** `/`

> **⚠️ 중요 (로컬 사용자 필독):**
> n8n을 `localhost`에서 실행 중이므로, 이메일에 적힌 링크(`http://localhost...`)는 외부(스마트폰 등)에서 클릭하면 안 열립니다. 실습할 때는 **같은 PC의 브라우저**에서 클릭해야 합니다.

### Step 3: 승인/반려 판단하기 (Switch Node)

사람이 `approve` 링크를 눌렀는지, `reject` 링크를 눌렀는지 판단합니다.

* **Node 추가:** `Switch`
* **Mode:** `Expression`
* **Number of Outputs:** `2`
* **Output Index:** 에서 승인 여부를 검사합니다.
    ```
    {{ $json.query.action == "approve" && $json.query.id == $('Loop Over Items').item.json.id }}
    ```


### Step 4: Code 노드
* **Switch 노드** 의 **출력 1(승인)**에 `Code` 노드를 연결합니다.
* LLM 이 출력했던 내용을 다시 가져오는 역할을 합니다.
* **JavaScript:**
  ```
  return $items("Basic LLM Chain");
  ```


### Step 5: DB 저장 (Google Sheets)

* `Code` 노드 다음에 `Google Sheets` 노드를 연결합니다.
* **Append or update row in sheet** 노드에서 `Document` 와 `Sheet` 를 도면용으로 만든 파일과 시트를 선택합니다.
    * **Mapping Column Mode** 는 `Map Automatically` 로 선택합니다.
    * **Column to match on** 는 `Drawing link` 로 선택합니다.

* **출력 0(반려)**에는 아무것도 연결하지 않거나, `Email` 노드를 연결해 "취소되었습니다" 알림을 보낼 수도 있습니다.


### Step 6: Respond to Webhook 노드(브라우저에 "처리되었습니다" 메시지 표시)

* **Respond to Webhook**노드를 **출력 1(승인)** 과 **출력 2(반려)** 에 각각 만들어 줍니다.
* **출력 1(승인) 의 Respond to Webhook** 노드에는 아래와 같이 입력합니다. (Expression)
    ```
    <div style="text-align: center; margin-top: 50px; font-family: sans-serif;">
    <h1 style="color: #4CAF50;">✅ 처리되었습니다.</h1>
    <p>{{ $('Code in JavaScript').item.json['Drawing link'] }}</p>
    <p>데이터가 성공적으로 구글 시트에 저장되었습니다.</p>
    <button onclick="window.close()" style="padding: 10px 20px; cursor: pointer;">창 닫기</button>
    </div>
    ```
    * **Respose Header** 를 추가하고, `Content-Type` 은 `text/html` 로 설정합니다.

* **출력 0(반려) 의 Respond to Webhook1** 노드에는 아래와 같이 입력합니다. (Expression)
    ```
    <div style="text-align: center; margin-top: 50px; font-family: sans-serif;">
    <h1 style="color: #4CAF50;">❌ 반려 되었습니다.</h1>
    <p>{{ $('Basic LLM Chain').item.json['Drawing link'] }}</p>
    <button onclick="window.close()" style="padding: 10px 20px; cursor: pointer;">창 닫기</button>
    </div>
    ```
    * **Respose Header** 를 추가하고, `Content-Type` 은 `text/html` 로 설정합니다.


---

## 4. [옵션] Google Chat으로 알림 받기

이메일 대신 메신저인 **Google Chat**도 가능합니다.

1. **Google Chat 스페이스(방)** 생성.
2. 스페이스 이름 클릭 -> **앱 및 통합** -> **Webhook 관리**.
3. **이름:** `n8n-bot` 입력 -> **저장**.
4. 생성된 **URL** 복사 (이게 비밀키입니다).
5. n8n에서 `Google Chat` 노드 추가.
* **Credential:** 불필요.
* **Webhook URL:** 복사한 URL 붙여넣기.
* **Message:**
```text
[승인요청] {{ $json.project_name }}
승인: {{ $execution.resumeUrl }}?action=approve&id={{ $('Loop Over Items').item.json.id }}
반려: {{ $execution.resumeUrl }}?action=reject

```


* *Google Chat은 HTML 버튼을 지원하지 않으므로, 텍스트 링크 형태로 보냅니다.*



---

## 5. 최종 실습 체크리스트

1. [ ] **앱 비밀번호**를 발급받아 n8n SMTP Credential에 넣었나요?
2. [ ] 이메일 본문(HTML)의 `href` 링크에 `{{ $execution.resumeUrl }}` 변수가 잘 들어갔나요?
3. [ ] **Wait 노드**가 `On Webhook Call` 상태인가요?
4. [ ] 메일을 받고 링크를 클릭했을 때, n8n 워크플로우가 **초록색(Success)**으로 바뀌며 진행되나요?

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
