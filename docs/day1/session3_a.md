# 🧩 [Session 3 추가 실습] AI 뉴스 요약 봇 만들기

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
* **URL:** `https://newsapi.org/v2/everything`
* **Query Parameters:**
* `q`: `한국`
* `language`: `ko`
* `sortBy`: `publishedAt`
* `pageSize`: `10`
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

## 4️⃣ Ollama 노드 – 요약하기

합쳐진 뉴스 데이터를 바탕으로 요약을 수행합니다.

* **Node:** `Ollama` (또는 Basic LLM Chain)
* **Credential:** Ollama API Key 생성 및 연결
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

* Simplify Output: Off


---

## 5️⃣ Google Chat – 결과 전송 (선택 사항 - 어려움)

Google Apps Script 기능을 사용하여 메시지를 보냅니다.

### 5-1. Google Apps Script 작성

1. [Google Apps Script](https://script.google.com/)에 접속하여 **새 프로젝트**를 만듭니다.
2. 아래 코드를 복사하여 붙여넣습니다. (이 코드는 n8n으로부터 데이터를 받아 내 계정 대신 메시지를 전달하는 역할을 합니다.)

```javascript
function doPost(e) {
  var params = JSON.parse(e.postData.contents);
  var spaceName = "spaces/XXXXXX"; // 여기에 스페이스 ID 입력
  var text = params.text;

  var message = {
    "text": text
  };

  // Google Chat API 직접 호출 (인증된 본인 권한 사용)
  var options = {
    "method": "post",
    "contentType": "application/json",
    "payload": JSON.stringify(message),
    "headers": {
      "Authorization": "Bearer " + ScriptApp.getOAuthToken()
    },
    "muteHttpExceptions": true
  };
  
  var response = UrlFetchApp.fetch("https://chat.googleapis.com/v1/" + spaceName + "/messages", options);
  return ContentService.createTextOutput(response.getContentText());
}

```

### 5-2. 스페이스 ID 확인법

* 웹 브라우저에서 Google Chat을 켭니다.
* 메시지를 보낼 스페이스에 들어갑니다.
* 주소창 URL을 보면 `https://mail.google.com/chat/u/0/#chat/space/XXXXXXX` 형태입니다.
* 여기서 `XXXXXXX` 부분이 스페이스 ID입니다. 코드의 `spaces/XXXXXX` 부분에 대입하세요.

### 5-3. 웹 앱으로 배포

1. 상단 **배포(Deploy)** 버튼 > **새 배포** 클릭.
2. 유형 선택에서 **웹 앱(Web App)** 선택.
3. 설정:
* **다음 사용자로 실행:** 나 (본인 계정)
* **액세스 권한이 있는 사용자:** 모든 사용자 (Anyone) — *n8n에서 접근하기 위함*


---

### 5-4. n8n에서 설정

* **HTTP Request 노드**를 만듭니다.
* **Method**: `POST`
* **URL**: 방금 복사한 **GAS 웹 앱 URL**
* **Body Parameters**: `text` : `전달할 메시지 내용`.


---

## ✅ 최종 결과 확인

1. **Test Workflow** 버튼 클릭
2. Google Chat 스페이스에 요약된 뉴스가 봇 이름으로 도착했는지 확인

---
