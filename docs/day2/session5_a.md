
## 🧩 [Session 5 심화 실습] 에이전트가 쓴 글을 '감수'하는 Supervisor 만들기

앞서 만든 '주식 분석 에이전트(Junior Analyst)'가 쓴 리포트를 무조건 신뢰하는 것이 아니라, **'팀장 에이전트(Senior Editor)'**가 한 번 더 검토하고, **합격(Approve)**일 때만 발송하거나 **불합격(Reject)**이면 수정을 지시하는 고급 패턴입니다.

> **실습 목표**
> * **Multi-Agent System:** 하나의 AI가 아니라, '작성자(Writer)'와 '검토자(Reviewer)' 등의 두 개 이상의 AI가 협업하는 구조를 만듭니다.
> * **Quality Control:** AI의 환각(Hallucination)이나 부적절한 말투를 2차 검증으로 걸러냅니다.
> * **Conditional Logic:** 검토 결과에 따라 전송할지, 다시 쓰게 할지 분기 처리합니다.
> 
> 

### 🏗️ 전체 워크플로우 구조

`User Input` → **`[AI Agent] 작성자`** → **`[LLM Node] 팀장(감수)`** → `Switch(판단)` → `(합격 시) Email 전송` / `(불합격 시) 피드백 루프`

1. **AI Agent (주니어):** 리포트 작성 -\> 출력.
2. **LLM Node (팀장):**
     * 입력: AI Agent의 글.
     * 프롬프트 예: "이 글의 팩트 체크와 톤앤매너를 점검해. 만약 부족하면 'REJECT', 괜찮으면 'APPROVE'라고 출력해."
3. **Switch Node (분기):**
     * `REJECT`일 경우 → 다시 AI Agent에게 수정 요청(Loop back).
     * `APPROVE`일 경우 → Slack 전송.

-----

### Step 1: Sub_Send_Email_Report 서브 워크플로우 만들기

AI가 호출할 '심부름센터(Sub Workflow)' **Sub_Send_Email_Report** 를 만듭니다.

1. **새 워크플로우 생성:**

* n8n 대시보드에서 `Add workflow`를 눌러 `Sub_Send_Email_Report` workflow 를 만듭니다.
* **[노드 구성]**
* `Execute Workflow Trigger` → **`Basic LLM Chain`** → **`Send Email`**

2. **Trigger 설정:**

* 노드 검색창에 `Execute Workflow Trigger`를 검색하여 추가합니다.
* **Fields:**
    * `text` (본문 내용)
    * `subject` (제목)

3.  **Send Email**

* `Send Email` 노드를 추가하고 Trigger와 연결합니다.
    * `From` 과 `To` Email 을 모두 자신의 이메일 주소를 입력한다.
    * **Subject:** (Expression 모드)
      ```javascript
      {{ $json.subject }}
      ```
    * **HTML:** (Expression 모드)
      ```javascript
      {{ $json.text }}
      ```

4.  **저장(Save) 및 출판(Publish):**

* **저장** 합니다.

---

### Step 2: AI Agent 의 User Message (사용자 입력) **(변경)**

> **역할:** 처음 실행인지, 피드백을 받고 재실행하는 것인지 **동적으로 입력**을 넣어주는 부분입니다.
> 메인 워크플로우로 돌아와서 AI-Agent 노드를 수정합니다.
> n8n의 User Message 를 **Define Below** 로 변경하고 아래 코드를 입력하세요.

이 코드는 **"피드백이 있으면 피드백을 포함해서 명령하고, 없으면 원래 질문만 던지는"** 로직입니다.

```javascript
// Expression 모드. 한 줄 주의.
{{ $json.reason ? "**⚠️ 긴급: 수정 요청 사항 (Supervisor Feedback)**\n   - 이전 리포트가 반려되었습니다. 다음 피드백을 반영하여 리포트를 **처음부터 다시** 완벽하게 작성하세요.\n   - **반려 사유(피드백):** " + $json.reason + "\n\n[이전 리포트] (아래 리포트를 수정하세요)\n" + $('Save Report').last().json.output : $json.chatInput }}
```

---

### Step 2.1: Edit Fields (Set) 노드 추가

> **역할:** 이 노드는 Agent AI 가 방금 뱉어낸 리포트를 "박제"하는 역할입니다.
* **Node Name:** `Save_Report`
* **Mode:** JSON
    * **JSON:** `{{ $json.output }}`

---

### Step 3: 팀장(Supervisor) 노드 추가하기

작성자 에이전트(AI Agent) 뒤에 검토 역할을 할 새로운 LLM 노드를 붙입니다. 여기서는 도구를 쓸 필요 없이 텍스트만 판단하면 되므로 `Basic LLM Chain`을 사용합니다.

1. **Node 추가:** `Basic LLM Chain` (또는 `OpenAI Chat Model`)
2. **연결:** `[Save_Report]` 노드의 뒤에 연결합니다.
3. **Prompt (User Message):** `{{ $json.output }}` 앞 노드의 output 을 입력받습니다.
4. **Prompt (System):** 팀장의 페르소나를 부여합니다.
    ```text
    당신은 까다로운 금융 리포트 편집장(Senior Editor)입니다.
    입력된 '주식 분석 리포트'를 읽고 심사 기준에 따라 엄격히 평가하세요.
    
    [심사 기준]
    1. 팩트 검증: 구체적인 수치(주가, 시가총액, 등락률 등)가 명시되어 있는가?
    2. 명확성: 투자의견(매수/매도/보류)이 결론에 확실히 드러나는가?
    3. 전문성: 비속어나 모호한 표현("~일 수도 있다", "~같다") 없이 전문적인 어조인가?
    
    [중요 행동 지침]
    - 절대로 팩트(숫자, 데이터)를 직접 수정하거나 지어내지 마세요.
    - 수치가 누락되었거나 틀려 보인다면, 직접 고치지 말고 반드시 반려(REJECT) 하세요.
    
    [출력 형식]
    반드시 마크다운(```json) 없이 순수한 JSON 텍스트만 출력하세요.
    
    {
      "status": "APPROVE" 또는 "REJECT",
      "reason": "승인 시, '적합'. 거절 시, 팀원에게 지시할 구체적인 수정 요청 사항 작성",
      "subject:" "거절 시, ''. 승인 시, 리포트의 핵심 결론(투자의견, 종목명, 주요 이슈)이 담긴 제목을 한 줄 작성 (예시: [매수] 삼성전자 - 반도체 업황 턴어라운드 본격화)"
    }
    ```

5. **Model 설정:** AI Agent 와 분리하기 위해 `qwen/Qwen3-4B-Thinking-2507-GGUF` 모델의 Credential 및 Model 을 추가 구성하여 선택합니다.
6. **Use Responses API:** Off
7. **Options:**
    * **Response Format:** `JSON`
    * **Timeout:** `300000`
    * **Sampling Temperature:** `0.2`
    * **Top P:** `0.3`

---

#### Step 4: 심사 결과에 따른 분기 (Switch)

팀장의 평가(`APPROVE` vs `REJECT`)에 따라 길을 나눕니다.

1. **Node 추가:** `Switch`
2. **Mode:** `Rules`
3. **Routing Rules:**
    * Value 1: `{{ $json.status }}` (팀장 노드의 출력값)
    * Operation: `is equal to`
    * Value 2: `APPROVE`

4. **Options:**
    * Fallback Output: `Extra Output`

---

#### Step 5: 합격 라인 (0) - 전송

합격 판정을 받은 경우, 원문을 Email 로 보냅니다.

1. **연결:** `Switch` 노드의 **첫 번째 출력(Output 0)**에 ``Call n8n Workflow` 노드 연결.
* **Source:** `Database` 
* **Workflow:** `Sub_Send_Email_Report`를 선택합니다.
* **Workflow Inputs:**
    * `text:` `{{ $('Save Report').last().json.output }}`
    * `subject:` `{{ $('Basic LLM Chain').last().json.subject }}`

---

#### Step 6: 불합격 라인 (Fallback) - 피드백 루프 (Feedback Loop)

불합격 시, 팀장의 피드백(`reason`)을 가지고 다시 작성자 에이전트에게 일을 시킵니다. **(이 부분이 핵심 기술입니다)**

1. **연결:** `Switch` 노드의 **두 번째 출력(Output Fallback)**에서 선을 뽑아 **`[AI Agent] 작성자` 노드의 입력(Input)**으로 연결합니다. (뒤에서 앞으로 선을 연결)

---

