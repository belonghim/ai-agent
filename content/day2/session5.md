## Session 5: 실전\! 에이전트 워크플로우 (Deep Dive)

> **목표**
>
>   * **ReAct (Reasoning + Acting) 패턴**을 이해합니다.
>   * **Custom Tool:** n8n의 다른 워크플로우를 '도구'로 만들어 에이전트에게 쥐여줍니다.
>   * **Looping:** 에이전트가 정보가 부족하면 스스로 다시 검색하는 '자율적 반복'을 경험합니다.

-----

### 1\. Chain vs Agent 차이점 이해

Session 4에서 만든 것이 \*\*Chain(체인)\*\*이라면, 이제 만들 것은 \*\*Agent(에이전트)\*\*입니다.

| 구분 | Chain (체인) | Agent (에이전트) |
| :--- | :--- | :--- |
| **작동 방식** | 정해진 순서대로 무조건 실행 | 목표를 달성하기 위해 **스스로 순서를 정함** |
| **도구 사용** | 개발자가 연결해 둔 것만 실행 | 필요하면 \*\*도구상자(Tools)\*\*에서 꺼내 씀 |
| **유연성** | 낮음 (경직됨) | 높음 (자율적) |

### 2\. AI Agent 노드 해부

n8n의 최신 `AI Agent` 노드는 LangChain 기반으로 동작합니다.

  * **Agent Type:** 보통 `Tools Agent` (OpenAI Functions 등)를 사용합니다.
  * **Tools:** 에이전트가 사용할 수 있는 도구들입니다.
      * *예: Calculator(계산), Wikipedia(검색), Custom n8n Workflow(사내 API)*
  * **Memory:** 대화의 맥락을 기억하기 위한 저장소. (`Window Buffer Memory` 등)
  * **System Prompt 심화:** 에이전트에게 페르소나와 '생각하는 법(Chain of Thought)'을 주입합니다.

기본 제공 도구(계산기 등) 외에, 우리가 만든 로직도 도구로 쓸 수 있습니다.

### 3\. [실습] 똑똑한 '기업 분석 & 리포트' 에이전트 만들기

**시나리오:** 사용자가 "엔비디아(Nvidia)의 최근 1주간 주가 흐름과 주요 뉴스를 찾아서 투자 의견을 써줘"라고 하면, 에이전트가 **웹 검색 → 뉴스 읽기 → 분석 → 리포트 작성**을 수행합니다.

#### Step 1: 도구(Tools) 준비 (Agent의 손과 발)

에이전트 노드에 연결할 3가지 도구를 준비합니다. 3가지 모두 `Call n8n Workflow` 도구를 사용하여 서브 워크플로우를 호출합니다.

1.  **google_search (검색):**
      * `Google Search` 노드 연결.
      * *용도:* 최신 정보를 실시간으로 긁어오는 역할.
2.  **web_scraper (읽기):**
      * `HTTP Request` 와 `HTML` (HTML -\> Text 변환) 노드 연결.
      * *용도:* 검색된 URL의 상세 본문을 읽는 역할.
3.  **email_sender (보고):**
      * `Send Email` 노드 연결.
      * *용도:* 보고서를 제출하는 역할.
* 3가지 도구 모두 별도의 서브 워크플로우(Slack 전송용)를 통해 만들어지고, 이를 메인 에이전트가 **함수처럼 호출**하게 됩니다.

##### Step 1.1: Sub_Google_Search 서브 워크플로우 만들기

AI가 호출할 '심부름센터(Sub Workflow)' **Sub_Google_Search** 를 만듭니다.

1.  **새 워크플로우 생성:**

* n8n 대시보드에서 `Add workflow`를 눌러 새 창을 엽니다.
* 이름을 명확하게 짓습니다. `Sub_Google_Search`

2.  **Trigger 설정 (받는 곳):**

* 노드 검색창에 `Execute Workflow Trigger`를 검색하여 추가합니다.
* 이 노드는 \*\*"누군가 나를 부르면(Call) 실행된다"\*\*는 뜻입니다.
* **[중요]** AI가 데이터를 넘겨줄 때 어떤 변수명을 쓸지 정해야 합니다. `keyword`라는 변수에 내용을 담아 보낸다고 가정합니다.

3.  **Action 설정 (하는 일):**

* **Node:** `HTTP Request`
* **Method:** `GET`
* **URL:** `https://www.googleapis.com/customsearch/v1`
* **Authentication:** `None` (API Key를 파라미터로 보낼 것이므로 여기선 끕니다.)

###### Query Parameters 설정 (핵심)

**`Send Query Parameters`** 스위치를 켜고, 아래 3가지 값을 추가합니다.

| Name | Value | 설명 |
| --- | --- | --- |
| **`q`** | `{{ $json.keyword }}` | 검색어 (이전 노드에서 받아온 값 매핑) |
| **`cx`** | `0123456789...` | **검색 엔진 ID** (Programmable Search Engine에서 복사한 값) |
| **`key`** | `AIzaSy...` | **GCP API Key** (Google Cloud Platform에서 발급받은 키) |

> **팁:** `num` 파라미터를 추가하고 값을 `3`이나 `5`로 주면 검색 결과 개수를 제한할 수 있습니다. (기본값은 10개)

4.  **저장(Save):**

* 워크플로우를 반드시 **저장**해야 다른 워크플로우에서 불러올 수 있습니다.

##### Step 1.2: Sub_Web_Scraper 서브 워크플로우 만들기

AI가 호출할 '심부름센터(Sub Workflow)' **Sub_Web_Scraper** 를 만듭니다.

1.  **새 워크플로우 생성:**

* n8n 대시보드에서 `Add workflow`를 눌러 `Sub_Web_Scraper` workflow 를 만듭니다.
* 아래 3개 노드를 순서대로 연결하세요.

**[Flow 구조]**
`Execute Workflow Trigger` → `HTTP Request` → `HTML Extract`

2.  **Trigger 설정:**

* 노드 검색창에 `Execute Workflow Trigger`를 검색하여 추가합니다.
* `url`라는 변수에 내용을 담아 보낸다고 가정합니다.

3.  **HTTP Request (접속하기)**

* **Method:** `GET`
* **URL:** `{{ $json.url }}` (Expression 모드)
* *(설명: 트리거로 들어온 url 주소로 접속합니다.)*


* **Authentication:** `None` (일반 웹페이지는 인증 불필요)

4. **HTML Extract (알맹이만 꺼내기)**

* `HTTP Request`의 결과는 지저분한 HTML 코드(`<div>...</div>`) 덩어리입니다. 여기서 글자만 발라내야 AI가 읽을 수 있습니다.
* **Source Data:** `JSON`
* **JSON Property:** `data` (HTTP Request가 가져온 내용이 담긴 변수명)
* **Extraction Values (추출 설정):**
* **Key:** `content` (결과를 담을 변수 이름)
* **CSS Selector:** `p` (본문 단락만 가져오기)
* **Return Value:** `Text` (**가장 중요!** HTML 태그 제거)

* **"사람인 척" 위장하기 (User-Agent 설정)**
* Header (헤더) 추가
* **Send Headers:** 스위치 **ON**
* **Header Parameters** 아래 **[Add Parameter]** 클릭:
* **Name:** `User-Agent`
* **Value:** `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/115.0.0.0 Safari/537.36`

* **[하나 더 추가]**
* **Name:** `Accept`
* **Value:** `text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8`



6.  **저장(Save):**

* **저장** 합니다.

##### Step 1.3: Sub_Web_Scraper 서브 워크플로우 만들기

AI가 호출할 '심부름센터(Sub Workflow)' **Sub_Web_Scraper** 를 만듭니다.

1.  **새 워크플로우 생성:**

* n8n 대시보드에서 `Add workflow`를 눌러 `Sub_Web_Scraper` workflow 를 만듭니다.

2.  **Trigger 설정:**

* 노드 검색창에 `Execute Workflow Trigger`를 검색하여 추가합니다.
* `url`라는 변수에 내용을 담아 보낸다고 가정합니다.

3.  **Action 설정:**

      * `Slack` 노드를 추가하고 Trigger와 연결합니다.
      * **Operation:** `Post a Message`
      * **Message Text:** 입력창 옆의 설정(기어 아이콘 또는 Expression)을 켜고, 아래와 같이 수식으로 입력합니다.
        ```javascript
        {{ $json.text }}
        ```
      * *(해설: Trigger로 들어온 데이터 중 `text`라는 이름의 값을 슬랙 메시지로 보내겠다는 뜻입니다.)*

4.  **저장(Save):**

* **저장** 합니다.

-----

#### Step 2: AI Agent 노드 설정 (Brain)

  * **Node:** `AI Agent` (LangChain 기반)
  * **Model:** `OpenAI Chat Model` (gpt-4o 권장 - 추론 능력이 좋아야 함)
  * **Prompt Instructions (System):**
    ```text
    당신은 월가 출신의 노련한 투자 애널리스트입니다.
    사용자의 요청을 수행하기 위해 아래 도구들을 스스로 판단하여 사용하세요.

    [사용 가능한 도구]
    - google_search: 최신 뉴스나 주가 정보를 검색할 때 사용
    - web_scraper: 특정 URL의 상세 본문을 읽어야 할 때 사용
    - email_sender: 최종 리포트를 이메일로 전송할 때 사용

    [행동 지침]
    1. 먼저 'google_search'를 사용하여 관련 정보를 광범위하게 수집하세요.
    2. 검색 결과 중 더 깊이 읽어봐야 할 중요한 기사가 있다면, 링크를 추출하여 'web_scraper'로 내용을 읽으세요.
    3. 수집된 정보를 종합하여, 통찰력 있는 '투자 분석 리포트'를 작성하세요.
    4. 마지막으로 'email_sender' 도구를 사용하여 작성된 리포트를 전송하세요. (매력적인 제목(subject)과 본문(text)을 구성할 것)

    [중요한 규칙]
    1. 도구의 이름은 정확히(google_search, email_sender 등) 호출해야 합니다.
    2. 'google_search' 사용 시 똑같은 키워드로 2번 이상 검색하지 마세요. (정보가 부족하면 키워드를 다르게 변경)
    3. 검색 결과가 다소 부족하더라도 무한 루프에 빠지지 말고, 확보된 정보만으로 최선을 다해 결론을 내리고 리포트를 발송하세요.
    4. 이메일 전송에 성공하면 "이메일 전송을 완료했습니다"라고 말하고 작업을 종료하세요.

    ```

##### Step 2.1: 메인 워크플로우에 연결하기 (도구 쥐여주기)

이제 다시 **AI Agent가 있는 메인 워크플로우**로 돌아옵니다.

1.  **도구 추가:**

      * `AI Agent` 노드의 **Tools** 항목에서 `+` 버튼을 누릅니다.
      * `Call n8n Workflow Tool` (또는 `Execute Workflow`)을 선택합니다.

2.  **도구 설정 (가장 중요 ★):**
    이 부분 설정이 AI가 도구를 잘 쓰냐 못 쓰냐를 결정합니다.

      * **Source:** `Database` 선택 (저장된 워크플로우 불러오기)
      * **Workflow:** 방금 만든 `Sub_Send_Slack_Report`를 선택합니다.
      * **Name:** `slack_sender` (AI가 인식할 도구의 이름입니다. 영문 소문자 권장)
      * **Description (설명서):** **여기가 핵심입니다.** AI에게 이 도구를 언제, 어떻게 써야 하는지 자연어로 설명해줘야 합니다.
        ```text
        Use this tool when you need to send a final report or message to Slack.
        The input must be a JSON object with a "text" field containing the message content.
        Example: { "text": "Here is the summary..." }
        ```

3. 작동 원리 (설명용)
    아래와 같이 비유하면 이해가 빠릅니다.

    1.  **상황:** 에이전트(인턴 사원)가 리포트 작성을 마쳤습니다.
    2.  **판단:** 에이전트는 본인의 도구 상자를 봅니다.
          * *"어? `slack_sender`라는 도구가 있네? 설명서를 보니 '리포트 보낼 때 쓰라'고 되어있고, `text`라는 봉투에 내용을 담아주면 된다고 하네."*
    3.  **실행:** 에이전트는 `Sub_Send_Slack_Report` 워크플로우를 호출하면서 `{ "text": "오늘 엔비디아 주가는..." }` 라는 데이터를 던집니다.
    4.  **결과:** 서브 워크플로우가 대신 슬랙을 보내고, 에이전트에게 "성공했어\!"라고 신호를 줍니다.

##### 💡 팁: 왜 굳이 서브 워크플로우로 만드나요?

1.  **재사용성:** 이 '슬랙 보내기' 워크플로우는 다른 에이전트나 자동화에서도 계속 불러다 쓸 수 있습니다.
2.  **복잡도 관리:** 메인 워크플로우가 너무 길어지는 것을 방지합니다.
3.  **보안:** 슬랙 API 키나 채널 ID 같은 민감한 정보를 에이전트 프롬프트에 직접 노출하지 않고 숨길 수 있습니다.

-----

#### Step 3: 피드백 루프(Feedback Loop) 구현

이 부분이 '단순 자동화'와 '에이전트'의 차이점입니다.

  * **자율적 판단:** 에이전트는 한 번 검색 후 정보가 부족하다고 판단하면(예: 검색 결과가 광고뿐임), 사용자가 시키지 않아도 **스스로 검색어를 수정하여 다시 도구를 실행**합니다.
  * **설정:** AI Agent 노드의 옵션 중 `Max Iterations`(최대 반복 횟수)를 5\~10회로 넉넉히 주어, 에이전트가 충분히 생각하고 시도할 시간을 줍니다.

#### Step 4: 결과 확인 및 디버깅

실행 로그(Executions)를 뜯어보며 에이전트의 \*\*"생각 과정(Thought Process)"\*\*을 확인하는 것이 중요합니다.

  * **Log 예시:**
    1.  *Thought:* 사용자가 엔비디아 정보를 원하네. 검색 도구를 써야지.
    2.  *Action:* Google Search ("Nvidia stock news this week")
    3.  *Observation:* 검색 결과 3개를 찾음. 근데 내용이 좀 부실해.
    4.  *Thought:* 상세 내용 파악을 위해 1번 기사 URL을 스크랩해야겠어.
    5.  *Action:* Web Scraper (URL...)
    6.  *Final Answer:* 리포트 작성 완료.

-----

### 4\. [심화] 에이전트가 쓴 글을 '감수'하는 Supervisor 만들기

(시간이 허락한다면 진행하는 고급 과정)

에이전트가 작성한 리포트를 바로 보내지 않고, \*\*두 번째 LLM(팀장 역할)\*\*이 검토하는 흐름입니다.

1.  **Agent 1 (주니어):** 리포트 작성 -\> 출력.
2.  **LLM Node (시니어/팀장):**
      * 입력: Agent 1의 글.
      * 프롬프트: "이 글의 팩트 체크와 톤앤매너를 점검해. 만약 부족하면 'REJECT', 괜찮으면 'APPROVE'라고 출력해."
3.  **Switch Node (분기):**
      * `REJECT`일 경우 → 다시 Agent 1에게 수정 요청(Loop back).
      * `APPROVE`일 경우 → Slack 전송.

-----

### 📝 2일차 요약 및 과제

**요약**

1.  **Dynamic Prompt:** 데이터(`{{json}}`)와 지시문(System Prompt)을 결합하여 AI에게 일을 시킨다.
2.  **JSON Output:** 자동화를 위해 AI의 출력은 반드시 정형화된 데이터여야 한다.
3.  **Agent:** 정해진 길이 아니라, 도구(Tools)를 판단하여 사용하는 자율적인 AI다.


**내일(3일차) 예고**

  * 이제 텍스트를 넘어 \*\*이미지(도면, 문서)\*\*를 처리합니다.
  * 복잡한 데이터를 DB에 쌓고, \*\*사람이 중간에 승인(Human-in-the-loop)\*\*하는 고도화된 워크플로우를 만듭니다.

-----
