## Session 4: 생성형 AI n8n에 접목하기

---

### 1. LLM 연동의 기초 (Chain vs Agent)

n8n에서 AI를 다루는 방식은 크게 두 가지가 있습니다. 이번 세션에서는 **Chain**에 집중합니다.

* **Basic LLM Chain (이번 시간):** 입력 → 처리 → 출력의 일방통행 구조. (예: 요약하기, 번역하기)
* **AI Agent (다음 시간):** 도구를 사용하고 스스로 판단하여 반복 수행하는 구조. (예: 검색해서 정보 찾기)

**[필수 준비물]**
* **Model 노드:** `OpenAI Chat Model` (어제 만든 Model Service)
* **Chain 노드:** `Basic LLM Chain`

---

### 2. Prompt를 Node 안에 넣는 방법 (Dynamic Prompting)

고정된 명령어가 아닌, 들어오는 데이터에 따라 변하는 **동적 프롬프트**를 작성해야 합니다.

* **Expression 모드 활용:** 프롬프트 입력창에서 `Expression`을 켜고, 입력 데이터 변수를 `{{ }}` 안에 넣습니다.

**[예제] 이메일 감정 분석 프롬프트**

```text
당신은 고객의 문의 메일을 분석하는 CS 전문가입니다.
아래의 [이메일 내용]을 읽고, 고객의 감정을 '긍정', '부정', '중립' 중 하나로 분류하세요.

[이메일 내용]
{{ $json.email_body }}

```

> **Tip:** 데이터가 들어가는 자리에 `{{ ... }}` 변수를 드래그 앤 드롭으로 배치하세요.

---

### 3. 구조화된 출력(Structured Output) 만들기 **(★핵심)**

자동화의 성공은 **"뒷단의 노드가 읽도록 약속된 데이터(JSON)"**를 만드는 데 달려 있습니다. AI가 수필을 쓰게 두지 말고, 기계가 읽을 수 있는 포맷을 강제해야 합니다.

**[필수] 프롬프트 엔지니어링 전략**

프롬프트 끝에 반드시 아래 **제약 조건**을 포함해야 에러가 나지 않습니다.

```text
당신은 고객의 문의 및 리뷰를 분석하는 CS 전문가입니다.
아래에 입력되는 본문을 분석 대상 데이터로 사용합니다.

문의 유형은 요구 사항 위주로 ['배송', '상품', '기타'] 중 하나로 분류합니다.

출력은 반드시 아래와 같이 마크다운(```json) 없이 순수한 JSON 텍스트만 출력하세요.
{
  "category": "문의 유형",
  "priority": 1 (숫자 1~5),
  "summary": "3줄 요약"
}

```

> **Why?** LLM은 종종 `json ... ` 형태로 답을 주는데, 이는 n8n에서 바로 인식이 안 됩니다. **"순수 JSON만 줘"**라고 말해야 합니다.
> 그리고, 추가적으로 OpenAI Chat Model 옵션의 `Response Format` 을 JSON 으로 지정해주는 것이 좋습니다.

---

### 4. [실습] 문의 및 리뷰 요약 및 이슈 분류기 만들기

**시나리오:** 고객의 문의 및 리뷰 내용을 요약하고, 중요도를 판단하여 알림을 보냅니다.

**Step 1. Trigger 설정**

* `Chat Trigger` 생성.
* Chat Trigger 에 가상의 문의 및 리뷰 데이터 입력 (`chatInput` 필드로 전달된다).

**Step 2. AI 두뇌 연결**

* `Basic LLM Chain` 연결 + `OpenAI Chat Model`(Use Responses API: Off) 부착.
* 위에서 배운 **JSON 포맷 프롬프트** 를 System Message 로 입력.

**Step 3. Code 노드에서 ID "찍어내기"**

* 이슈별 고유 ID가 자동으로 따라오지 않으므로, n8n 안에서 직접 ID를 생성해서 붙여줍니다.
* `Basic LLM Chain` 에서 연결

```javascript
let result = $json;
result.id = `ISSUE-${new Date().toLocaleString('sv-SE', { timeZone: 'Asia/Seoul' }).replace(' ', 'T').slice(0, 23)}`;
return result;

```

**Step 4. Google Sheets Document 생성**

* 로그를 보관하기 위한 Document 를 생성합니다.
* drive.google.com 에 접속 후, 공유용 directory 및 Google Sheets Document 생성
  * 디렉토리 예: ~/n8n/n8n_save/n8n_feedback
* **n8n_feedback** Document 를 열고, sheet 명을 feedback 으로 수정합니다.
* 첫번째 행에 id 라는 열을 만들어 둡니다.


**Step 5. Google Sheets 노드 생성**

* 생성된 이슈들을 로그로 보관합니다.
* Code 노드 다음에 연결합니다. ( 준비 Part A 진행 필요합니다. )
* **Google Sheets** 의 `Append or update row in sheet` 노드를 선택합니다.
* **Document:** 의 `From list` 중에 사용할 Google Sheets Document 를 선택합니다. (예: n8n_feedback)
* **Sheet:** 의 `From list` 중에 사용할 Sheet 를 선택합니다. (예: feedback)
* **Mapping Column Mode:** 중에 `Map Automatically` 를 선택합니다.
* **Column to match on** 중에 `id` 를 선택합니다.



---

### 🚦 Switch 노드로 True/False 가르기 [실습 추가 가이드] 

**목표:**

* **긴급(Priority 4 이상) 이슈:** 알림으로 메시지 전송 (True)
* **모든 이슈:** Google Sheets (feedback) 에 기록


**Gmail 알림** 을 위한 상세 실습 가이드 입니다.

---
#### 1단계: Switch 노드 설정하기 (Rules)

Google Sheets 노드 다음에 `Switch` 노드를 생성니다.

1. **Mode:** `Rules`를 선택합니다. (기본)
2. **Routing Rules (규칙 정의):**
* **Value 1 (검사할 값):** 입력창 옆의 변수 아이콘을 누르거나 드래그하여 이전 노드의 `priority` 값을 넣습니다. (`{{ $json.priority }}`)
* **Operation (조건):** `Number` 중에 **is greater than or equal to** (>=, 크거나 같다) 를 선택합니다.
* *(주의: 여기서 String을 선택하면 숫자 5를 문자 "5"로 처리하여 크기 비교가 안 됩니다.)*
* **Value 2 (기준 값):** `4`를 입력합니다.

3. Subject: `[n8n 알림] 고객 불만 {{ $('Code in JavaScript').item.json.id }}`
4. Email Format: Text
  ```
  아이디: {{ $('Code in JavaScript').item.json.id }}
  카테고리:  {{ $('Code in JavaScript').item.json.category }}
  우선순위:  {{ $('Code in JavaScript').item.json.priority }}
  요약:  {{ $('Code in JavaScript').item.json.summary }}
  ```

---

#### 2단계: 선 연결하기 (가장 중요!)

설정을 마치고 캔버스로 나오면 `Switch` 노드 오른쪽에 **작은 동그라미(Output)**가 **1개(또는 그 이상)** 생겨있을 것입니다.

1. **첫 번째 구멍 (Output 0):** **"조건 일치 (True)"**
* 설정한 규칙(4 이상)에 맞는 데이터만 이 구멍으로 나옵니다.
* 이 구멍에서 나온 선에 **`Send email`** 노드를 만들어 연결합니다.

> **Tip:** 준비 Part B 를 참고하여 Google email 을 준비해야 합니다.

---

#### 3단계: 테스트 시나리오 (검증)

수강생들이 실제로 두 경로가 다 작동하는지 눈으로 확인하게 해야 합니다.

**CASE A: 긴급 상황 테스트**

1. 앞단의 `Chat Trigger` 노드로 돌아갑니다.
2. 테스트 데이터를 전달합니다: (상품이나 배송에 대한 강한 불만을 표현하는 메세지를 작성하세요)
3. **Send (종이비행기)** 아이콘 클릭.
4. **결과:** `Switch` 노드의 **윗쪽 선(Slack)** 과 **아래쪽 선(Google Sheets)** 이 초록색으로 빛나고, Gmail 메시지가 와야 합니다.

**CASE B: 일반 상황 테스트**

1. 데이터를 다시 수정합니다: "일상적인 질문 또는 칭찬성 메세지 전달"
2.  **Send (종이비행기)** 아이콘 클릭.
3. **결과:** 이번엔 `Switch` 노드의 **아래쪽 선(Google Sheets)** 만 초록색으로 빛나고, 시트에 행이 추가되어야 합니다.

---

### 준비 Part A. Google Drive & Sheets 연동 준비 (Service Account 방식)

서버나 백그라운드 자동화에서 가장 안정적인 **서비스 계정(Service Account)** 방식입니다.

#### 1️⃣ Google Cloud 프로젝트 생성

1. [Google Cloud Console](https://console.cloud.google.com) 접속. (최초 접속시 개인 정보를 요구할 수 있습니다.) 


#### 2️⃣ API 활성화 (Drive & Sheets)

1. 좌측 메뉴 **[API 및 서비스]** → **[라이브러리]**.
2. 검색창에 **`Google Sheets API`** 검색 → **[사용 설정]** 클릭.
* `My First Project` 가 자동으로 할당됩니다
3. 다시 라이브러리로 돌아가서 **`Google Drive API`** 검색 → **[사용 설정]** 클릭.
* *(Drive API는 시트 파일 자체를 찾거나 접근할 때 필요합니다.)*
4. 다시 라이브러리로 돌아가서 **`Gmail API`** 검색 → **[사용 설정]** 클릭.


#### 3️⃣ Service Account(서비스 계정) 생성

1. 좌측 메뉴 **[API 및 서비스]** → **[사용자 인증 정보]**.
2. 상단 **[+ 사용자 인증 정보 만들기]** → **[서비스 계정]** 선택.
3. **이름:** 식별하기 좋은 이름 (예: `n8n-bot`) 입력 → **[만들기 및 계속]**.
4. **역할(Role):** 선택하지 않고 **[계속]** → **[완료]** (역할은 필수가 아닙니다).


#### 4️⃣ 키(JSON) 생성 및 이메일 확인 (★중요)

1. 생성된 **서비스 계정 이메일 주소** (예: `n8n-bot@project-id.iam.gserviceaccount.com`)를 **따로 복사해둡니다.**
2. 해당 서비스 계정 클릭.
3. 상단 **[키(Keys)]** 탭 클릭 → **[키 추가]** → **[새 키 만들기]**.
4. **JSON** 선택 → **[만들기]** (자동으로 파일이 다운로드됩니다).
* ⚠️ **주의:** 이 파일은 비밀번호와 같습니다. 절대 타인에게 공유하지 마세요.


#### 5️⃣ Google Drive 폴더를 Service Account에 공유

로봇(서비스 계정)은 내 구글 드라이브를 볼 수 없습니다. **봇을 내 드라이브에 공유 해줘야 합니다.**

1. 개인 Google Drive 접속
2. 접근시키고 싶은 **폴더 우클릭 → 공유**
3. 이메일에 **Service Account 이메일** 입력
   예:
   ```
   n8n-drive-bot@your-project-id.iam.gserviceaccount.com
   ```
4. 권한:
   * 업로드/수정 → Editor
5. **완료**
👉 이 폴더 안의 모든 파일/하위폴더에 접근 가능해진다.


#### 6️⃣ n8n에서 연결하기

1. n8n의 **Google Sheets** 노드 → Credentials.
2. **[Create New]** → **[Google Service Account]** 선택.
3. `Service Account Email`: (자동 입력되거나 비워둬도 됨).
4. `Service Account Key`: 다운로드 받은 **JSON 파일의 내용 전체**를 메모장으로 열어 복사 후 붙여넣기. (또는 파일 업로드)
5. **[Save]** → Connection Tested 확인.

---


### 준비 Part B. 메일 전송 준비 ( 알람 발송용. Slack 등으로 대체 가능)

"알람"을 보내는 목적으로 **Gmail SMTP(앱 비밀번호)** 방식을 사용합니다.

* **보내는 사람:** 본인의 Gmail 주소
* **장점:** 서비스 계정처럼 토큰 만료 걱정 없이 한 번 설정하면 계속 사용 가능합니다.

### 설정 순서

#### 1️⃣ Google 계정에서 '앱 비밀번호' 생성

1. [Google 계정 관리](https://myaccount.google.com/) 접속.
2. 좌측 **[보안]** 탭 클릭.
3. 'Google에 로그인' 항목에서 **[2단계 인증]**이 켜져 있어야 합니다. (안 켜져 있다면 켜주세요).
4. 검색창에 **`앱 비밀번호`** 검색 또는 [앱 비밀번호 페이지](https://myaccount.google.com/apppasswords) 직접 접속.
5. **앱 이름:** `n8n-email` 입력 후 **[만들기]**.
6. 생성된 **16자리 비밀번호(기기용 앱 비밀번호)**를 복사해 둡니다. (띄어쓰기 무시하고 사용)


#### 2️⃣ n8n에서 이메일 노드 설정 (Gmail Node 아님)

n8n에서는 `Gmail` 노드 대신 **`Send Email`** 노드(SMTP)를 사용하는 것이 훨씬 간편합니다.

1. n8n 캔버스에서 **`Send Email`** 노드 추가.
2. **Credentials** → **[Create New]** → **[SMTP]** 선택.
3. 설정 값 입력:
* **User:** 본인의 Gmail 주소 (예: `myname@gmail.com`)
* **Password:** 아까 복사한 **16자리 앱 비밀번호**
* **Host:** `smtp.gmail.com`
* **Port:** `465`
* **SSL/TLS:** 켜기 (On)

4. **[Save]** 클릭하여 연결 확인.


#### 3️⃣ 메일 발송 테스트

1. `Send Email` 노드 설정:
* **From Email:** 본인 Gmail 주소
* **To Email:** 알람 받을 주소
* **Subject:** 테스트 알람
* **Text:** 내용 입력


2. 실행하여 메일이 잘 오는지 확인.

---




