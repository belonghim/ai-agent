#  [Session 3 추가 실습] AI 뉴스 요약 봇 만들기

## 목표

* **NewsAPI**를 통해 매일 최신 뉴스를 자동으로 수집합니다.
* 수집된 뉴스(5건)를 **Code 노드**로 정리하여 하나의 텍스트로 합칩니다.
* **OpenAI(ChatGPT)**에게 "오늘의 뉴스 흐름을 요약해줘"라고 시킵니다.
* 요약된 결과를 **Google Chat** 스페이스로 전송합니다. (선택 사항)

---

## 전체 워크플로우 (Workflow)

```
graph LR
    A[Cron (매일)] --> B[HTTP Request (뉴스수집)]
    B --> C[Code (데이터 가공)]
    C --> D[OpenAI (요약)]
    D --> E[Google Chat (전송)]

```

---

## 0️⃣ 사전 준비

1. **NewsAPI 키 발급**: [https://newsapi.org](https://newsapi.org) 가입 후 API Key 복사
2. **OpenAI API 키**: 수업 시간에 사용한 키 준비
3. **Google Chat 스페이스 생성**:
* [Google Chat](https://mail.google.com/chat)
* **새 스페이스 만들기** (이름 예: `내 뉴스 알림방`)
* ⚠️ **중요:** 개인 계정에서는 '나와의 채팅'이 아닌 **'스페이스(Space)'**를 만들어야 웹훅을 쓸 수 있습니다.



---

## 1️⃣ Cron 노드 – 스케줄러

매일 아침 뉴스를 받아오기 위한 트리거입니다.

* **Node:** `Schedule` (구 Cron)
* **Trigger Interval:** `Every Day`
* **Hour:** `08` (오전 8시) / **Minute:** `00`
* *(실습 시에는 `Manual Trigger`를 연결해두고 테스트하세요.)*



---

## 2️⃣ HTTP Request – 뉴스 수집

최신 헤드라인을 가져옵니다.

* **Node:** `HTTP Request`
* **Method:** `GET`
* **URL:** `https://newsapi.org/v2/top-headlines`
* **Query Parameters:**
* `country`: `kr`
* `apiKey`: `(여러분의 NewsAPI Key)`


* **Authentication:** `None` (파라미터에 키를 넣었으므로 없음)

---

## 3️⃣ Code 노드 – 데이터 하나로 합치기 (중요!)

가져온 뉴스 기사 5개를 **하나의 긴 텍스트**로 묶어서 AI에게 전달해야 합니다. (따로 보내면 AI가 5번 실행됩니다.)

* **Node:** `Code` (구 Function)
* **Language:** `JavaScript`
* **Code:** (아래 코드를 복사해서 붙여넣으세요)

```javascript
// 이전 노드(HTTP Request)에서 가져온 기사 목록
const articles = items[0].json.articles;

// 상위 5개만 자르기
const top5 = articles.slice(0, 5);

// 기사 제목과 링크를 하나의 문자열로 합치기
let newsText = "";
top5.forEach((article, index) => {
  newsText += `${index + 1}. ${article.title}\n   (링크: ${article.url})\n\n`;
});

// 다음 노드(OpenAI)로 넘겨줄 데이터 포맷팅
return [
  {
    json: {
      news_content: newsText
    }
  }
];

```

---

## 4️⃣ OpenAI 노드 – 요약하기

합쳐진 뉴스 데이터를 바탕으로 요약을 수행합니다.

* **Node:** `OpenAI Chat Model` (또는 Basic LLM Chain)
* **Credential:** OpenAI API Key 연결
* **Model:** `기존 모델 재사용`
* **System Message:**
```text
당신은 전문 뉴스 브리핑 비서입니다. 입력된 뉴스들을 종합하여 핵심만 간결하게 요약하세요.

```


* **User Message:** (Expression 모드 켜기)
```text
아래는 오늘 한국의 주요 뉴스 5가지입니다.

{{ $json.news_content }}

---
위 내용을 바탕으로:
1. 전체적인 오늘의 이슈를 한 문단으로 브리핑해주세요.
2. 각 기사의 핵심을 1줄씩 요약 리스트로 만들어주세요.

```



---

## 5️⃣ Google Chat – 결과 전송 (선택 사항 - 어려움)

개인 계정에서는 **Incoming Webhook(수신 웹훅)** 기능을 사용하여 메시지를 보냅니다.

### 5-1. 프로젝트 생성 및 API 활성화

1. [Google Cloud Console](https://console.cloud.google.com)에 접속합니다.
2. 새 프로젝트를 생성합니다 (예: `n8n-chat-personal`).
3. **[API 및 서비스]** > **[라이브러리]**로 이동합니다.
4. **`Google Chat API`**를 검색하여 **[사용 설정(Enable)]**을 클릭합니다.

### 5-2. OAuth 동의 화면 구성

1. **[API 및 서비스]** > **[OAuth 동의 화면]**으로 이동합니다.
2. **User Type:** `외부(External)` 선택 > **[만들기]**.
3. **앱 정보:** 앱 이름(예: `n8n-chat`), 사용자 지원 이메일, 개발자 연락처만 입력하고 저장합니다.
4. **범위(Scopes):** **[범위 추가 또는 삭제]** 클릭 > 필터에 `chat.messages` 검색 > `.../auth/chat.messages` (메시지 및 반응 만들기) 체크 > 업데이트 > 저장.
* *Tip: 목록에 없으면 수동으로 `https://www.googleapis.com/auth/chat.messages`를 입력하세요.*

5. **테스트 사용자(Test Users):** **[+ ADD USERS]** 클릭 > **본인의 구글 이메일** 입력 > 저장. (이 단계가 없으면 작동하지 않습니다!)

### 5-3. OAuth 클라이언트 ID 만들기

1. **[사용자 인증 정보(Credentials)]** > **[+ 사용자 인증 정보 만들기]** > **[OAuth 클라이언트 ID]**.
2. **애플리케이션 유형:** `웹 애플리케이션`.
3. **승인된 리디렉션 URI:**
* n8n의 **[Google OAuth2 API]** 자격 증명 설정 창에 있는 URL을 복사해 넣습니다.
* 보통: `https://YOUR-N8N-URL/rest/oauth2-credential/callback`
* (로컬 실행 시: `http://localhost:5678/rest/oauth2-credential/callback`)

4. **[만들기]** 클릭 > **클라이언트 ID**와 **클라이언트 보안 비밀**을 복사해 둡니다.

---

### 5-4. n8n에서 Credential 설정

Google Chat 전용 노드 대신, 범용적인 **HTTP Request**와 **Google OAuth2** 자격 증명을 사용합니다.

1. n8n에서 **Credentials** > **[Create New]** > **`Google OAuth2 API`** 선택.
2. **Client ID / Secret:** 복사해둔 값 붙여넣기.
3. **Scope (중요):** 아래 URL을 정확히 입력합니다.
```text
https://www.googleapis.com/auth/chat.messages

```

4. **Auth URI / Token URI:** 기본값 유지.
5. **[Sign in with Google]** 클릭 > 본인 계정 선택 > **[고급] > 이동(unsafe)** > **[허용]**.

---

### 5-5. 내 스페이스 ID 찾기

메시지를 보낼 채팅방(스페이스)의 주소를 알아야 합니다.

1. 웹브라우저로 Google Chat(Gmail)의 해당 스페이스에 들어갑니다.
2. URL을 확인합니다.
* 예: `https://mail.google.com/chat/u/0/#chat/space/AAAA1234abc`


3. 맨 뒤의 **`AAAA1234abc`** 부분이 바로 **Space ID**입니다.

### 5-6. HTTP Request 노드 설정

마지막 단계인 **5번 노드**를 아래와 같이 설정합니다.

* **Node:** `HTTP Request`
* **Authentication:** `Predefined Credential Type` > **`Google OAuth2 API`** (방금 만든 것 선택)
* **Method:** `POST`
* **URL:** (아래 URL의 `SPACE_ID`를 본인 것으로 교체)
```text
https://chat.googleapis.com/v1/spaces/SPACE_ID/messages

```


* *예: `https://chat.googleapis.com/v1/spaces/AAAA1234abc/messages*`


* **Send Body:** `ON`
* **Body Content Type:** `JSON`
* **Body Parameters:**
* **Name:** `text`
* **Value:** `{{ $json.content }}` (OpenAI의 요약 결과 변수)




---

## ✅ 최종 결과 확인

1. **Test Workflow** 버튼 클릭
2. Google Chat 스페이스에 요약된 뉴스가 봇 이름으로 도착했는지 확인

---
