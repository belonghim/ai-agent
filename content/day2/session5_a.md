
## 🧩 [Session 5 심화 실습] 에이전트가 쓴 글을 '감수'하는 Supervisor 만들기

앞서 만든 '주식 분석 에이전트(Junior Analyst)'가 쓴 리포트를 무조건 신뢰하는 것이 아니라, **'팀장 에이전트(Senior Editor)'**가 한 번 더 검토하고, **합격(Approve)**일 때만 발송하거나 **불합격(Reject)**이면 수정을 지시하는 고급 패턴입니다.

> **실습 목표**
> * **Multi-Agent System:** 하나의 AI가 아니라, '작성자(Writer)'와 '검토자(Reviewer)' 두 개 이상의 AI가 협업하는 구조를 만듭니다.
> * **Quality Control:** AI의 환각(Hallucination)이나 부적절한 말투를 2차 검증으로 걸러냅니다.
> * **Conditional Logic:** 검토 결과에 따라 전송할지, 다시 쓰게 할지 분기 처리합니다.
> 
> 

### 🏗️ 전체 워크플로우 구조

`User Input` → **`[Agent 1] 작성자`** → **`[LLM Node] 팀장(감수)`** → `Switch(판단)` → `(합격 시) Slack 전송` / `(불합격 시) 피드백 루프`

1. **Agent 1 (주니어):** 리포트 작성 -\> 출력.
2. **LLM Node (팀장):**
     * 입력: Agent 1의 글.
     * 프롬프트 예: "이 글의 팩트 체크와 톤앤매너를 점검해. 만약 부족하면 'REJECT', 괜찮으면 'APPROVE'라고 출력해."
3. **Switch Node (분기):**
     * `REJECT`일 경우 → 다시 Agent 1에게 수정 요청(Loop back).
     * `APPROVE`일 경우 → Slack 전송.

-----

### Step 1.1: Sub_Send_Email_Report 서브 워크플로우 만들기

AI가 호출할 '심부름센터(Sub Workflow)' **Sub_Send_Email_Report** 를 만듭니다.

1. **새 워크플로우 생성:**

* n8n 대시보드에서 `Add workflow`를 눌러 `Sub_Send_Email_Report` workflow 를 만듭니다.
**[노드 구성]**
`Execute Workflow Trigger` → **`Basic LLM Chain`** → **`Send Email`**

2. **Trigger 설정:**

* 노드 검색창에 `Execute Workflow Trigger`를 검색하여 추가합니다.
* **Fields:**
* `text` (본문 내용)

3. **Basic LLM Chain**

* 노드 검색창에 `Basic LLM Chain`을 검색하여 추가합니다.
* **Source for Prompt:** `Define below`
* **Prompt:** `{{ $json.text }}`

* **Chat Messages**
    * `Type Name:` `System`
    * `Message:`
    ```
    당신은 전문 '금융 뉴스레터 디자이너'입니다.
    입력받은 [주식 분석 리포트]를 분석하여 이메일 전송용 데이터로 변환하세요.
    
    [작업 1: 제목 생성]
    - 리포트의 핵심 결론(투자의견, 종목명, 주요 이슈)이 담긴 클릭하고 싶은 제목을 한 줄 작성하세요.
    - 예시: [매수] 삼성전자 - 반도체 업황 턴어라운드 본격화
    
    [작업 2: HTML 본문 변환]
    - 마크다운 텍스트를 깔끔한 HTML 스타일로 변환하세요.
    - <html>, <body> 태그는 제외하고 <div> 태그부터 시작하세요.
    - 주요 수치(주가, 등락률)는 빨간색 또는 굵게(Bold) 강조하세요.
    - 가독성을 위해 적절한 CSS(font-size: 14px, line-height: 1.6 등)를 인라인 스타일로 적용하세요.
    
    [출력 형식]
    반드시 마크다운 태그(```json) 없이 아래 순수 JSON 형식으로만 출력하세요.
    {
    "subject": "작성된 이메일 제목",
    "html_body": "변환된 HTML 코드 전체"
    }
    ```

3.1. **Model**

* `Basic LLM Chain` 창 하단의 Model 아이콘을 눌러서 `Model` 추가합니다.
* **Credential:** AI Agent 와 분리하기 위해 `gemma` 모델의 credential 을 선택합니다.
* **Model:** `/models/hf.google.gemma-3n-E4B`
* **Options:**
    * **Response Format:** `JSON`
    * **Timeout:** `300000`

4.  **Send Email**

* `Send Email` 노드를 추가하고 Trigger와 연결합니다.
    * `From` 과 `To` Email 을 모두 자신의 이메일 주소를 입력한다.
    * **Subject:** (Expression 모드)
      ```javascript
      {{ $json.subject }}
      ```
    * **HTML:** (Expression 모드)
      ```javascript
      {{ $json.html_body }}
      ```

5.  **저장(Save):**

* **저장** 합니다.

### Step 2: User Message (사용자 입력) **(추가)**

> **역할:** 처음 실행인지, 피드백을 받고 재실행하는 것인지 **동적으로 입력**을 넣어주는 부분입니다.
> 메인 워크플로우로 돌아와서 AI-Agent 노드를 수정합니다.
> n8n의 User Message 를 **Define Below** 로 변경하고 아래 코드를 입력하세요.

이 코드는 **"피드백이 있으면 피드백을 포함해서 명령하고, 없으면 원래 질문만 던지는"** 로직입니다.

```javascript
// Expression 모드. 한 줄 주의.
{{ $json.reason ? "**⚠️ 긴급: 수정 요청 사항 (Supervisor Feedback)**\n   - 이전 리포트가 반려되었습니다. 다음 피드백을 반영하여 리포트를 **처음부터 다시** 완벽하게 작성하세요.\n   - **반려 사유(피드백):** " + $json.reason + "\n\n[이전 리포트] (아래 리포트를 수정하세요)\n" + $('Save Report').last().json.output : $json.chatInput }}
```

---

### Step 3: 팀장(Supervisor) 노드 추가하기

작성자 에이전트(Agent 1) 뒤에 검토 역할을 할 새로운 LLM 노드를 붙입니다. 여기서는 도구를 쓸 필요 없이 텍스트만 판단하면 되므로 `Basic LLM Chain`을 사용합니다.

1. **Node 추가:** `Basic LLM Chain` (또는 `OpenAI Chat Model`)
2. **연결:** `[Agent 1] 작성자` 노드의 뒤에 연결합니다.
3. **Prompt (System):** 팀장의 페르소나를 부여합니다.
```text
당신은 까다로운 금융 리포트 편집장(Senior Editor)입니다.
입력된 '주식 분석 리포트'를 읽고 다음 기준을 엄격히 심사하세요.

[심사 기준]
1. 팩트(구체적인 수치)가 포함되어 있는가?
2. 투자의견(매수/매도/보류)이 명확한가?
3. 비속어나 불확실한 표현("~일 수도 있다")이 없는가?

출력은 반드시 아래 JSON 형식으로만 하세요. (마크다운 없이)
{
  "status": "APPROVE" 또는 "REJECT",
  "reason": "승인 이유 또는 거절 시 구체적인 피드백",
  "refined_content": "수정이 필요하다면 수정한 본문, 아니면 원문 유지"
}

```


4. **Model 설정:** `gpt-4o` 또는 `gpt-3.5-turbo` (검토는 3.5도 잘합니다).
5. **Output Parsing:** 설정에서 `JSON Output` 옵션을 켜거나, 노드 뒤에 `Structured Output Parser`를 붙여 결과를 JSON 객체로 만듭니다.

#### Step 4: 심사 결과에 따른 분기 (Switch)

팀장의 평가(`APPROVE` vs `REJECT`)에 따라 길을 나눕니다.

1. **Node 추가:** `Switch`
2. **Mode:** `Rules`
3. **Rule 1 (합격):**
* Condition: `String`
* Value 1: `{{ $json.status }}` (팀장 노드의 출력값)
* Operation: `Equal to`
* Value 2: `APPROVE`


4. **Rule 2 (불합격):**
* Value 1: `{{ $json.status }}`
* Operation: `Equal to`
* Value 2: `REJECT`



#### Step 5: 합격 라인 (Approve) - 전송

합격 판정을 받은 경우, 팀장이 다듬어준 글(`refined_content`)을 슬랙으로 보냅니다.

1. **연결:** `Switch` 노드의 **첫 번째 출력(Output 0)**에 `Slack` (또는 `Call n8n Workflow`) 노드 연결.
2. **Message:** `{{ $json.refined_content }}`
* *Tip:* "팀장 승인 완료" 뱃지를 달아주면 더 좋습니다.



#### Step 6: 불합격 라인 (Reject) - 피드백 루프 (Feedback Loop)

불합격 시, 팀장의 피드백(`reason`)을 가지고 다시 작성자 에이전트에게 일을 시킵니다. **(이 부분이 핵심 기술입니다)**

1. **연결:** `Switch` 노드의 **두 번째 출력(Output 1)**에서 선을 뽑아 **`[Agent 1] 작성자` 노드의 입력(Input)**으로 연결합니다. (뒤에서 앞으로 선을 연결)
2. **데이터 조작 (Prompt 수정):**
* 그냥 연결하면 똑같은 질문을 또 하게 됩니다.
* 작성자 에이전트의 프롬프트 입력창(`Expression` 모드)을 다음과 같이 수정해야 합니다.


```javascript
// 만약 팀장의 피드백(reason)이 있다면 그걸 포함해서 지시하고, 없으면(첫 실행이면) 원래 질문을 씁니다.
{{ $json.reason ? 
   "팀장님이 다음 이유로 반려했어: " + $json.reason + ". 이 피드백을 반영해서 리포트를 다시 작성해." : 
   $fromAI("TriggerNode").message }} 

```

---

### Step 7: 메인 워크플로우에 email_sender 연결하기 (도구 쥐여주기)

1.  **도구 추가:**

* `AI Agent` 노드의 **Tools** 항목에서 `+` 버튼을 누릅니다.
* `Call n8n Workflow Tool` 을 선택합니다.

2.  **도구 설정:**

* **Source:** `Database` 선택
* **Workflow:** `Sub_Send_Email_Report`를 선택합니다.
* **Workflow Inputs:** `text` 와 `subject` 오른쪽의 반짝이는 별 아이콘을 누릅니다.
* **Name:** `email_sender`
* **Description (설명서):**
    ```text
    Use this tool to send a final report via email.
    The input must be a JSON object with "subject" and "text" fields.
    Example: { "subject": "Nvidia Analysis Report", "text": "Here is the summary..." }
    ```

-----
