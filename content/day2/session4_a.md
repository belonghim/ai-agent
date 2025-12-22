## 🧩 [Session 4 추가 실습] 오디오 파일로 회의록 자동 생성하기

> **실습 목표**
>
>   * 단순 텍스트 처리를 넘어, **멀티모달(Audio -\> Text)** 처리 능력을 배웁니다.
>   * OpenAI의 \*\*Whisper 모델(STT)\*\*과 \*\*GPT 모델(LLM)\*\*을 연쇄적으로 활용합니다.
>   * 녹음 파일 하나만 던지면 [요약-할일-일정]이 정리되어 노션(Notion)이나 슬랙(Slack)으로 들어오는 흐름을 구현할 수 있습니다.

-----

### 1\. 워크플로우 전체 흐름 (Flow)

이 워크플로우는 \*\*듣고(Listen) -\> 쓰고(Transcribe) -\> 요약(Summarize)\*\*하는 3단계로 이루어집니다.

1.  **Trigger:** 오디오 파일 감지 (예: Telegram 음성 메시지, Google Drive 업로드)
2.  **Action 1 (STT):** OpenAI Whisper 모델이 음성을 텍스트로 받아쓰기
3.  **Action 2 (LLM):** 받아쓴 텍스트를 GPT가 요약 및 구조화
4.  **Action 3 (Output):** 결과 전송 (Slack, Email 등)

-----

### 2\. 단계별 설정 가이드

#### Step 0: Whisper 모델 구동하기

##### Model 다운로드 및 로드

실습에 사용할 오픈소스 whisper 모델을 로드합니다.

* **Target Model:** `ggerganov/whisper.cpp`

1. AI Lab 메뉴 -> **Catalog** 선택
2. 모델 검색창에 `ggerganov/whisper.cpp` 입력

##### whisper Model Service 실행 (Local API Server)

다운로드한 모델을 n8n이 접속할 수 있도록 API 서버로 띄웁니다.

1. AI Lab -> **Services** (또는 Local Server) 탭 클릭
2. **New Model Service** 클릭 -> 다운로드한 whisper 모델 선택
3. **Container port** 임의 지정 (예: 38888 포트)
4. **Create Service** 클릭
5. **Open Service Details** 클릭 후, **Endpoint 확인:** `http://host.containers.internal:38888/inference`
* *이 주소는 'OpenAI' 노드에서 사용됩니다.*


#### Step 1: 오디오 파일 (Trigger)

실습 편의를 위해 **Google Drive**나 **Telegram**을 많이 사용합니다. 지난 시간 연동해둔 **Google Drive**을 사용합니다. (또는 로컬 파일을 읽는 `Read Binary File` 노드도 사용 가능)

  * **Node:** `Google Drive` Trigger
  * **On changes involving a specific folder**
  * **Folder:** `From list` 에서 권한 있는 folder 선택. (오디오용 폴더 선택)
  * **Options:**
      * File Type: `Audio` 선택
      * **핵심:** 오디오 파일은 n8n 내부에서 **Binary Data**로 처리됨을 이해해야 합니다.

#### Step 2: 오디오 파일 가져오기 (Download file)
* **Google Drive Download** 노드에서 'Operation` 을 **Download** 를 선택합니다.
    * **File:** 은 `By ID` 로 선택한 뒤, {{ $json.id }} 표현식으로 File 을 다운로드 받게 됩니다.
    * *Put Output File in Field:* 는 `data` 라는 이름으로 바이너리를 전달합니다.
    * mp3 파일로 된 대화 레코딩 파일을 준비하세요.

#### Step 3: 음성을 텍스트로 변환 (Whisper API)

사람의 귀 역할을 하는 단계입니다.

* **Node:** `HTTP Request`
* **Method:** `POST`
* **URL:** `http://localhost:38888/inference` (Podman에서 확인한 주소)
* **Send Body:** `On` 켜기
* **Body Content Type:** `Form-Data` **(중요!)**
* API 서버로 파일을 전송할 때는 반드시 이 옵션을 써야 합니다.

* **Body Parameters:**
* **Parameter Type:** `n8n Binary File`
* **Parameter Name:** `file` (API 문서에서 요구하는 파일 필드명 확인 필수)
* **Input Data Field:** Step 1에서 넘어온 바이너리 데이터 속성명: `data`


#### Step 4: 회의록 요약 및 구조화 (LLM)

단순히 받아쓴 글은 줄글이라 읽기 힘듭니다. 이를 깔끔하게 정리합니다.

  * **Node:** `Basic LLM Chain`
  * **Prompt (System):**
    ```text
    너는 전문 속기사이자 프로젝트 매니저야.
    사용자가 제공하는 '회의 스크립트'를 읽고 아래 형식에 맞춰 Markdown으로 정리해줘.

    ## 1. 회의 개요
    - 주제: (내용 유추)
    - 분위기: (대화 톤을 보고 유추)

    ## 2. 3줄 요약
    -
    -
    -

    ## 3. Action Items (할 일)
    - [담당자] 할 일 내용 (기한)
    ```
  * **Prompt (User):** Expression 모드 사용
      * `{{ $json.text }}` (이전 Whisper 노드에서 나온 텍스트 결과)

  * **Node 추가 및 연결** `OpenAI Chat Model`


#### Step 5: 결과 내보내기

  * **Node:** `Send Email` or `Slack`
  * **Content:** Step 3(LLM)에서 나온 `text`를 본문에 넣습니다.

-----

### 3\. 실습 꿀팁 (Troubleshooting)

**Q. 오디오 파일이 너무 크면 어떻게 하나요?**

> A. Whisper API는 파일 크기 제한(약 25MB)이 있습니다. 실습 때는 1\~2분 내외의 짧은 녹음 파일이나 `m4a`, `mp3` 포맷(압축률 좋음)을 사용하는 것이 좋습니다.

**Q. 사투리나 전문 용어도 알아듣나요?**

> A. 네, 생각보다 매우 잘 알아듣습니다. 하지만 고유명사(회사 이름, 프로젝트 코드명)는 프롬프트에 미리 "이 회의에는 'n8n', 'LangChain' 같은 단어가 자주 등장해"라고 힌트를 주면 정확도가 올라갑니다.

-----
