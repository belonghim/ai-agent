
## 🧩 [Session 5 심화 실습] 에이전트가 쓴 글을 '감수'하는 Supervisor 만들기

앞서 만든 '주식 분석 에이전트(Junior Analyst)'가 쓴 리포트를 무조건 신뢰하는 것이 아니라, **'팀장 에이전트(Senior Editor)'**가 한 번 더 검토하고, **합격(Approve)**일 때만 발송하거나 **불합격(Reject)**이면 수정을 지시하는 고급 패턴입니다.

> **실습 목표**
> * **Multi-Agent System:** 하나의 AI가 아니라, '작성자(Writer)'와 '검토자(Reviewer)' 두 개의 AI가 협업하는 구조를 만듭니다.
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

4.  **Send Email**

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

### Step 3: User Message (사용자 입력) **(추가)**

> **역할:** 처음 실행인지, 피드백을 받고 재실행하는 것인지 **동적으로 입력**을 넣어주는 부분입니다.
> 메인 워크플로우로 돌아와서 AI-Agent 노드를 수정합니다.
> n8n의 User Message 를 **Define Below** 로 변경하고 아래 코드를 입력하세요.

이 코드는 **"피드백이 있으면 피드백을 포함해서 명령하고, 없으면 원래 질문만 던지는"** 로직입니다.

```javascript
// Expression 모드. 한 줄 주의.
{{ $json.reason ? "**⚠️ 긴급: 수정 요청 사항 (Supervisor Feedback)**\n   - 이전 리포트가 반려되었습니다. 다음 피드백을 반영하여 리포트를 **처음부터 다시** 완벽하게 작성하세요.\n   - **반려 사유(피드백):** " + $json.reason + "\n\n[이전 리포트] (아래 리포트를 수정하세요)\n" + $('Save Report').last().json.output : $json.chatInput }}
```

---

### Step 4: 메인 워크플로우에 email_sender 연결하기 (도구 쥐여주기)

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
