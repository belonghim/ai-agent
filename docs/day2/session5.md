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
  * **Tools:** 에이전트는 다양한 도구들을 사용할 수 있습니다.
      * *예: Calculator(계산), Wikipedia(검색), Custom n8n Workflow(사내 API)*
  * **System Prompt 심화:** 에이전트에게 페르소나와 '생각하는 법(Chain of Thought)'을 주입합니다.

기본 제공 도구(계산기 등) 외에, 우리가 만든 로직도 도구로 쓸 수 있습니다.

### 3\. [실습] 똑똑한 '기업 분석 & 리포트' 에이전트 만들기

**시나리오:** 사용자가 "엔비디아(Nvidia)의 최근 1주간 주가 흐름과 주요 뉴스를 찾아서 투자 의견을 써줘"라고 하면, 에이전트가 **웹 검색 → 뉴스 읽기 → 분석 → 리포트 작성**을 수행합니다.

---

#### Step 1: 도구(Tools) 준비 (Agent의 손과 발)

에이전트 노드에 연결할 2가지 도구를 준비합니다. 2가지 모두 `Call n8n Workflow` 도구를 사용하여 서브 워크플로우를 호출합니다.

1.  **google_search (검색):**
      * `Google Search` 노드 연결.
      * *용도:* 최신 정보를 실시간으로 긁어오는 역할.
2.  **web_scraper (읽기):**
      * `HTTP Request` 와 `HTML` (HTML -\> Text 변환) 노드 연결.
      * *용도:* 검색된 URL의 상세 본문을 읽는 역할.
* 2가지 도구 모두 별도의 서브 워크플로우(Slack 전송용)를 통해 만들어지고, 이를 메인 에이전트가 **함수처럼 호출**하게 됩니다.

##### 💡 팁: 왜 굳이 서브 워크플로우로 만드나요?

1.  **재사용성:** 서브 워크플로우는 다른 에이전트나 자동화에서도 계속 불러다 쓸 수 있습니다.
2.  **복잡도 관리:** 메인 워크플로우가 너무 길어지는 것을 방지합니다.
3.  **보안:** 슬랙 API 키나 채널 ID 같은 민감한 정보를 에이전트 프롬프트에 직접 노출하지 않고 숨길 수 있습니다.
4.  **가공성:** 복잡한 데이터를 가공하여 의도하는 목적에 맞게 AI 에게 쥐어 줄 수 있습니다.

---

## 🛠️ [사전 준비] Google Search API 연동 가이드

n8n의 `google_search` 워크플로우를 통해 실제 구글 검색 결과를 가져오기 위해서는 **1. GCP(Google Cloud Platform) 프로젝트 설정**과 **2. 검색 엔진(CSE) 생성** 두 가지 작업이 모두 필요합니다.

### 1단계: Google Cloud Platform (GCP) 설정

API Key를 발급받기 위한 과정입니다.

1. **GCP 콘솔 접속:** [https://console.cloud.google.com/](https://console.cloud.google.com/) 에 접속하여 로그인합니다.


2. **API 라이브러리 활성화:**
* 좌측 메뉴(햄버거 버튼)에서 **[API 및 서비스] > [라이브러리]**로 이동합니다.
* 검색창에 `Custom Search API`를 검색합니다.
* **[Google Custom Search API]**를 클릭하고 **[사용(Enable)]** 버튼을 누릅니다.


3. **API Key 발급:**
* 좌측 메뉴에서 **[API 및 서비스] > [사용자 인증 정보(Credentials)]**로 이동합니다.
* 상단의 **[+ 사용자 인증 정보 만들기]** > **[API 키]**를 선택합니다.
* 생성된 **`API Key`**를 복사하여 메모장에 저장해 둡니다.

> **주의:** GCP는 사용 패턴에 따라 결제 계정(신용카드) 등록을 요구할 수도 있습니다. (Custom Search API는 하루 100회 무료입니다.)

---

### 2단계: Programmable Search Engine (검색 엔진 ID) 만들기

"무엇을 검색할지" 정의하는 검색 엔진을 만들고 ID(CX)를 얻는 과정입니다.

1. **설정 사이트 접속:** [https://programmablesearchengine.google.com/](https://programmablesearchengine.google.com/) 에 접속합니다.
2. **검색 엔진 추가:**
* **[추가]** 또는 **[검색엔진 만들기]** 버튼을 클릭합니다.
* **이름:** 원하는 이름 입력 (예: `n8n-agent-search`).
* **검색할 대상:** **'전체 웹 검색'** 옵션을 선택합니다.
* **[만들기]**를 완료합니다.


3. **검색 엔진 ID (CX) 확인:**
* 생성 완료 화면 혹은 좌측 메뉴의 **[개요]**에서 **`검색엔진 ID`** (cx 값)를 복사하여 메모장에 저장해 둡니다.

> **Tip (전체 웹 검색 설정):**
> 간혹 '특정 사이트 검색'으로만 생성되는 경우가 있습니다. 이 경우 생성 후 **[설정] > [검색 기능]** 메뉴에서 **'전체 웹 검색'** 토글을 켜야 Agent가 인터넷 전체를 뒤질 수 있습니다.

---

#### Step 1.1: Sub_Google_Search 서브 워크플로우 만들기

AI가 호출할 '심부름센터(Sub Workflow)' **Sub_Google_Search** 를 만듭니다.

1.  **새 워크플로우 생성:**

* n8n 대시보드에서 `Add workflow`를 눌러 새 창을 엽니다.
* 이름을 명확하게 짓습니다. `Sub_Google_Search`
* 아래 **3개 노드**를 연결합니다.

> **[전체 흐름]**
> `Execute Workflow Trigger` → `HTTP Request` → **`Code` (다이어트)**

2.  **Trigger 설정 (받는 곳):**

* 노드 검색창에 `Execute Workflow Trigger`를 검색하여 추가합니다.
* 이 노드는 \*\*"누군가 나를 부르면(Call) 실행된다"\*\*는 뜻입니다.
* **[중요]** AI가 데이터를 넘겨줄 때 어떤 변수명을 쓸지 정해야 합니다. `keyword`라는 변수에 내용을 담아 보낸다고 가정합니다.

3.  **Action 설정 (하는 일):**

* **Node:** `HTTP Request`
* **Method:** `GET`
* **URL:** `https://www.googleapis.com/customsearch/v1`
* **Authentication:** `None` (API Key를 파라미터로 보낼 것이므로 여기선 끕니다.)

##### Query Parameters 설정 (핵심)

**`Send Query Parameters`** 스위치를 켜고, 아래 3가지 값을 추가합니다.

| Name | Value | 설명 |
| --- | --- | --- |
| **`q`** | `{{ $json.keyword }} -site:finance.yahoo.com` | 검색어 (이전 노드에서 받아온 값 매핑) |
| **`cx`** | `0123456789...` | **검색 엔진 ID** (Programmable Search Engine에서 복사한 값) |
| **`key`** | `AIzaSy...` | **GCP API Key** (Google Cloud Platform에서 발급받은 키) |
| **`num`** | `10` | **검색 결과 개수를 제한** (기본값이 10) |


4. **Code 노드:**
* `HTTP Request` 노드 바로 뒤에 붙여서, 복잡한 JSON을 심플하게 바꿉니다.
* 아래 코드를 복사해서 넣으세요.

```javascript
const items = $input.first().json.items || [];

// 검색 결과가 없을 경우
if (items.length === 0) {
  return { result: "검색 결과가 없습니다. 키워드를 변경해서 다시 시도하세요." };
}

// JSON을 AI가 읽기 쉬운 '텍스트'로 변환
const textOutput = items.map((item, index) => {
  return `[정보 ${index + 1}]\n- 제목: ${item.title}\n- 요약: ${item.snippet}\n- URL: ${item.link}`;
}).join("\n\n");

// 'result'라는 이름의 단순 텍스트로 반환
return { result: textOutput };
```

5.  **저장(Save) 및 출판(Publish):**

* 워크플로우를 반드시 **저장 및 출판(Publish)** 해야 다른 워크플로우에서 불러올 수 있습니다.

---

#### Step 1.2: Sub_Web_Scraper 서브 워크플로우 만들기

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

* **"사람인 척" 위장하기 (User-Agent 설정)**
    * Header (헤더) 추가
    * **Send Headers:** 스위치 **ON**
    * **Header Parameters** 추가시 **[Add Parameter]** 클릭:
    * **Name:** `User-Agent`
    * **Value:** `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/115.0.0.0 Safari/537.36`
    * **Name:** `Accept`
    * **Value:** `*/*`

* **Follow Redirects 켜기**
    * `Options > Directs > Follow Redirects` 스위치를 ON 로 킵니다.

4. **HTML Extract Content (알맹이만 꺼내기)**

* `HTTP Request`의 결과는 HTML 코드(`<div>...</div>`) 덩어리입니다. 여기서 글자만 발라내야 AI가 읽을 수 있습니다.
    * **Source Data:** `JSON`
    * **JSON Property:** `data` (HTTP Request가 가져온 내용이 담긴 변수명)
    * **Extraction Values (추출 설정):**
    * **Key:** `content` (결과를 담을 변수 이름)
    * **CSS Selector:** `p` (본문 단락만 가져오기)
    * **Return Value:** `Text` (**가장 중요!** HTML 태그 제거)


5.  **저장(Save) 및 출판(Publish):**

* **저장 및 출판(Publish)** 합니다.

-----

#### Step 2: AI Agent 노드 설정 (Brain)

이제 다시 **AI Agent가 있는 메인 워크플로우**로 돌아옵니다.

* **Node:** `AI Agent` (LangChain 기반)
* **Model:** `OpenAI Chat Model`
* **Prompt Instructions (System):** (Expression 모드)
  ```text
  당신은 '주니어 투자 애널리스트'. 주어진 종목에 대해 리포트를 작성.
  
  [핵심 목표: 팀장님의 승인]
  작성된 리포트는 까다로운 '편집장(Supervisor)'에게 심사 예정.
  
  [사용 가능한 도구]
  - google_search: 키워드 검색 (최신 뉴스, 티커 확인)
  - web_scraper: URL 내용 읽기 (상세 데이터 확보)
  
  [현재 시각]
  {{ $now.format('yyyy년 MM월 dd일 HH시 mm분 ss초') }}
  
  [행동 지침]
  - 기존 지식을 믿지 말고, 반드시 도구를 통해 검증된 정보만 작성.
  - 같은 키워드로 반복 탐색 금지.
  - 보고서는 1024 context를 넘지 않도록 짧게 작성.
  {{
    $json.reason
      ? `
  [입력 처리 지침]
  - 이전 리포트가 반려된 상황. 피드백 내용을 최우선으로 반영하여 내용을 수정.
  - 완성된 리포트만 출력.
  `
      : `
  [행동 순서]
  * 1단계. 탐색 (google_search)
    - 키워드: "... market cap stock price text", "... stock news analysis forecast text"
    - 주가 데이터는 탐색 요약 내용에서 추출.
    - 2단계 읽기 도구를 호출한 적 없다면 리포트를 작성 불가.
  
  * 2단계. 읽기 (web_scraper)
    - Javascript 나 Asp 동적 사이트(finance.yahoo, marketwatch)는 피하고, 텍스트 위주의 뉴스 기사를 선택.
    - 탐색 링크 중 가장 중요해 보이는 기사 1개를 찾아 내용을 읽기를 시도.
  
  * 3단계. 작성 (Report)
    - 수집된 정보만으로 최선을 다해 리포트를 작성.
    - 수집된 정보를 바탕으로 아래 포맷에 맞춰 작성.
    - 정보가 없는 항목은 지어내지 말고 "확인 불가"로 명기.
  
  [리포트 포맷]
  종목명/티커 주식 분석 리포트
  * 시각: (현재 시각)
  
  1. 기업 개요
  - 현재 주가: (오늘 기준)
  - 시가총액: (오늘 기준)
  - 변동률: (현재 시각 기준 1일 및 1주)
  
  2. 최신 주요 뉴스 (오늘 기준 2개만)
  - 뉴스 1: (제목, 게시 일자, 요약, URL링크)
  - ...
  
  3. 투자 매력도 및 리스크
  - 긍정적 요인: (짧게 한줄 요약)
  - 리스크 요인: (짧게 한줄 요약)
  
  4. 최종 투자의견
  - 결론: [매수 / 매도 / 보류]
  - 근거: (짧게 한줄 요약)
  ` }}
  ```

* **OpenAI Chat Model**
    * **Credential & Model:** `ibm-granite/granite-4.0-tiny-GGUF` ( AI Lab 에서 추가 서비스 구성 )
    * **Use Responses API:** Off
    * **Options:**
        * **Maximum Number of Tokens:** `4095`
        * **Timeout:** `300000`
        * **Response Format:** `Text`
        * **Sampling Temperature:** `0.1`
        * **Top P:** `0.2`

---

##### Step 2.1: 메인 워크플로우에 google_search 연결하기 (도구 쥐여주기)


* `AI Agent` 노드의 **Tools** 항목에서 `+` 버튼을 누릅니다.
1.  **도구 추가:**
* `Call n8n Workflow Tool` 을 선택합니다.

2.  **도구 설정:**
이 부분 설정이 AI가 도구를 잘 쓰냐 못 쓰냐를 결정합니다.

* **Source:** `Database` 선택 (저장된 워크플로우 불러오기)
* **Workflow:** 위에서 만든 `Sub_Goole_Search`를 선택합니다.
* **Workflow Inputs:** `keyword` 오른쪽의 반짝이는 별 아이콘을 누릅니다. (AI 가 알아서 입력을 넣게 됩니다.)
* **(왼쪽 상단)Name:** `google_search` (AI가 인식할 도구의 이름입니다. 영문 소문자 권장)
* **Description (설명서):** **여기가 핵심입니다.** AI에게 이 도구를 언제, 어떻게 써야 하는지 자연어로 설명해줘야 합니다.
    ```text
    이 도구를 사용하여 Google Search 에서 탐색하여 요약을 얻을 수 있습니다.
    입력은 "keyword" 필드를 포함하는 JSON 객체여야 합니다.
    예시: { "keyword": "Nvidia stock price market cap" }
    ```

3. 작동 원리 (설명용)
    아래와 같이 비유하면 이해가 빠릅니다.

    1.  **상황:** 에이전트(인턴 사원)가 리포트 작성을 마쳤습니다.
    2.  **판단:** 에이전트는 본인의 도구 상자를 봅니다.
          * *"어? `google_search`라는 도구가 있네? 설명서를 보니 '실시간 정보 검색에 써라'고 되어있고, `keyword`라는 봉투에 내용을 담아주면 된다고 하네."*
    3.  **실행:** 에이전트는 `Sub_Goole_Search` 워크플로우를 호출하면서 `{ "keyword": "오늘 엔비디아 주가 뉴스" }` 라는 데이터를 던집니다.
    4.  **결과:** 서브 워크플로우가 대신 구글 검색을 전달하고, 에이전트에게 결과를 넘겨줍니다.

---

##### Step 2.2: 메인 워크플로우에 web_scraper 연결하기 (도구 쥐여주기)

1.  **도구 추가:**

* `AI Agent` 노드의 **Tools** 항목에서 `+` 버튼을 누릅니다.
* `Call n8n Workflow Tool` 을 선택합니다.

2.  **도구 설정:**

* **Source:** `Database` 선택
* **Workflow:** `Sub_Web_Scraper`를 선택합니다.
* **Workflow Inputs:** `url` 오른쪽의 반짝이는 별 아이콘을 누릅니다.
* **Name:** `web_scraper`
* **Description (설명서):**
    ```text
    이 도구를 사용하여 특정 웹페이지의 상세 내용을 읽을 수 있습니다.
    링크에서 상세 내용 정보를 얻어야 할 때 유용합니다.
    입력은 "url" 필드를 포함하는 JSON 객체여야 합니다.
    예시: { "url": "https://www.bbc.com/news/..." }
    ```


-----

#### Step 3: 피드백 루프(Feedback Loop) 구현

이 부분이 '단순 자동화'와 '에이전트'의 차이점입니다.

  * **자율적 판단:** 에이전트는 한 번 검색 후 정보가 부족하다고 판단하면(예: 검색 결과가 광고뿐임), 사용자가 시키지 않아도 **스스로 검색어를 수정하여 다시 도구를 실행**합니다.
  * **설정:** AI Agent 노드의 옵션 중 `Max Iterations`(최대 반복 횟수)를 10회로 넉넉히 주어, 에이전트가 충분히 생각하고 시도할 시간을 줍니다.

#### Step 4: 결과 확인 및 디버깅

실행 로그(Executions)를 뜯어보며 에이전트의 \*\*"생각 과정(Thought Process)"\*\*을 확인하는 것이 중요합니다.

  * **Log 예시:**
    1.  *Thought:* 사용자가 엔비디아 정보를 원하네. 검색 도구를 써야지.
    2.  *Action:* Google Search ("Nvidia stock price 1 week chart news")
    3.  *Observation:* 검색 결과 3개를 찾음. 근데 내용이 좀 부실해.
    4.  *Thought:* 상세 내용 파악을 위해 1번 기사 URL을 스크랩해야겠어.
    5.  *Action:* Web Scraper (URL...)
    6.  *Observation:* 내용을 확인함. 근데 내용이 부실해.
    7.  *Thought:* 다른 키워드로 재검색해야 겠어.
    8.  *Action:* Google Search ("Nvidia stock price chart recent week")
    9.  *Observation:* 검색 결과 3개를 찾음. 이제 리포트 써 보자.
    10. *Final Answer:* 리포트 작성 완료.

-----

### 📝 2일차 요약 및 과제

**요약**

1.  **Dynamic Prompt:** 데이터(`{{ $json }}`)와 지시문(System Prompt)을 결합하여 AI에게 일을 시킨다.
2.  **JSON Output:** 자동화를 위해 AI의 출력은 반드시 정형화된 데이터여야 한다.
3.  **Agent:** 정해진 길이 아니라, 도구(Tools)를 판단하여 사용하는 자율적인 AI다.


**내일(3일차) 예고**

  * 이제 텍스트를 넘어 \*\*이미지(도면, 문서)\*\*를 처리합니다.
  * 복잡한 데이터를 AI 를 통해 확인하고 DB에 쌓고, \*\*사람이 중간에 승인(Human-in-the-loop)\*\*하는 고도화된 워크플로우를 만듭니다.

-----
