## Session 6: n8n으로 구현하는 이미지 데이터 정형화

-----

### 1\. 비정형 데이터란?

  * **정형 데이터:** 엑셀, DB처럼 행과 열이 딱 떨어지는 데이터.
  * **비정형 데이터:** 이미지, PDF, 도면, 계약서 등 컴퓨터가 바로 읽기 힘든 데이터.
  * **목표:** 이미지 속의 정보를 **OCR(광학 문자 인식)** 기술로 추출해 \*\*JSON(정형 데이터)\*\*으로 바꾸는 것.

-----


### 2\. 최신 트렌드: Vision LLM의 등장
최근에는 **GPT-4o**나 **Claude 3.5 Sonnet** 같은 **Vision 기능이 탑재된 LLM**을 사용하는 것이 더 정확하고 쉬워졌습니다. 단순 글자 판독을 넘어 "이게 무슨 의미인지" 해석까지 가능하기 때문입니다.

-----

### 3\. 왜 HTTP Request로 OCR을 호출하나요?

* **전문성:** 도면 기호, 한국어 손글씨, 복잡한 표는 범용 LLM(GPT-4o)보다 **전문 문서 파싱 API(Upstage Document Parse, Naver Clova OCR 등)**가 훨씬 정확할 때가 많습니다.
* **비용 및 속도:** 대량의 문서를 처리할 때 전용 API가 더 저렴하고 빠를 수 있습니다.
* **범용성:** n8n에 전용 노드가 없는 서비스라도 **`HTTP Request`** 노드만 다룰 줄 알면 세상의 모든 API를 연결할 수 있습니다.

-----

### 4\. [실습] OCR API + LLM 하이브리드 워크플로우

**시나리오:** 구글 드라이브에 이미지가 올라오면, **외부 OCR 솔루션**으로 글자를 읽어내고, LLM을 통해 핵심 정보만 뽑아 DB에 저장합니다.


#### Step 1: 이미지 파일 받기 (Trigger + Download)

* 사전에 구글 드라이브에 디렉토리를 생성해둡니다: 예 n8n_image
* **Google Drive Trigger** 노드에서 관찰할 Folder 를 지정하고, 'Watch For` 는 **File Created** 로 지정
* **Google Drive Download** 노드에서 'Operation` 을 **Download** 를 선택합니다.
    * **File:** 은 `By ID` 로 선택한 뒤, {{ $json.id }} 표현식으로 File 을 다운로드 받게 됩니다.
    * `Add option > `*Put Output File in Field:* 는 `data` 라는 이름으로 바이너리를 전달합니다.

#### Step 2: Batch 사이즈를 고정 (Loop Over Items)

* **Loop Over Items** 노드에서 `Batch Size` 는 **1** 로 지정
* **Google Drive Trigger** 와 **Google Drive Download** 사이에 `Loop Over Items` 를 연결합니다.
    * **Loop Over Items** 의 `loop` 점에서 *Google Drive Download* 로 연결하면 됩니다.
    * **Loop Over Items** 의 입력선은 `Google Drive Trigger` 와 마지막 노드(`Google Sheets`)에서 연결됩니다.

#### Step 3: 외부 OCR API 호출하기 (HTTP Request) **(★핵심)**

* **Node:** `HTTP Request`
* **Method:** `POST`
* **URL:** 사용하려는 OCR 서비스의 API 주소 (예: `http://host.containers.internal:12222/ocr`)
* **Send Body:** `On` 켜기
* **Body Content Type:** `Form-Data` **(중요!)**
* API 서버로 파일을 전송할 때는 반드시 이 옵션을 써야 합니다.

* **Body Parameters:**
  * **Parameter Type:** `n8n Binary File`
  * **Parameter Name:** `file` (API 문서에서 요구하는 파일 필드명 확인 필수)
  * **Input Data Field:** Step 1에서 넘어온 바이너리 데이터 속성명: `data`

#### 3.1: OCR 컨테이너 설정
OCR 컨테이너를 12222 포트로 띄웁니다.

```bash
podman run -d -p 12222:8000 --name ocr quay.io/joopark/ocr:vector
```

-----

#### Step 4: 텍스트 구조화 및 요약 (LLM)

OCR API는 보통 방대한 양의 "원문 텍스트"나 "좌표 정보"를 줍니다. 이를 우리가 원하는 깔끔한 JSON으로 만듭니다.

* **Node:** `Basic LLM Chain`
* **Source for Prompt:** `Define Below`
* **Prompt (User Message):**
```text
아래 제공되는 [입력 텍스트]는 OCR을 통해 추출된 Raw Data야.

다음 규칙에 따라 데이터를 추출해줘:
1. **날짜 포맷 통일:** 모든 날짜는 `YYYY-MM-DD HH:MM:SS` 형태로 변환해. (예: `251216` -> `2025-12-16`)
2. **거래일자 추론:** 명시적인 '거래일자' 라벨이 없어도, `YYMMDD`와 `HH:MM:SS` 패턴이 보이면 날짜로 인식해.
3. **금액 포맷:** 모든 금액에서 '원', ','를 제거하고 숫자(Integer)로 변환해.
4. **불필요한 정보 제거:** 단순한 구분선이나 의미 없는 기호는 무시해.

[입력 텍스트]
"""
{{ $json.text }}
"""

```

#### Step 5: 데이터 정제 (Structured OutPut Parser)

* **Basic LLM Chain** 에서 `Require Specific Output Format` 를 키면 Structured Output Parser 를 연결할 수 있습니다.
* Structured Output Parser 는 AI가 뱉어낸 JSON 문자열을 n8n이 인식할 수 있는 JSON 객체로 변환합니다.
* *Define using JSON Schema:*
    ```
    {
      "type": "object",
      "properties": {
        "approval_number": {
          "type": "string",
          "description": "승인번호"
        },
        "total_amount": {
          "type": "number",
          "description": "합계금액"
        },
        "transaction_date": {
          "type": "string",
          "description": "거래일자"
        },
        "merchant_name": {
          "type": "string",
          "description": "가맹점명"
        }
      },
      "required": ["approval_number", "total_amount", "transaction_date", "merchant_name"]
    }

    ```

#### Step 6: DB 저장 (Google Sheets Append or update row in sheet)

  * **Node:** `Google Sheets Append or update row in sheet`
  * **Operation:** `Append or Update Row` (또는 Insert)
  * **Document** 와 **Sheet** 를 사전에 준비한 `Google Sheets Document` 와 `Sheet` 이름으로 선택 (자동 조회된 항목 중 선택)
  * **Mapping:**
      * **Mapping Column Mode** 는 `Map Each Column Manually` 로 선택
      * **Column to match on** 는 `승인번호` 로 선택 (자동 조회된 항목 중 선택)
      * `승인번호` 는 *{{ $json.output.approval_number }}* 표현식으로 입력
      * `거래일자` 는 *{{ $json.output.transaction_date }}* 표현식으로 입력
      * `가맹점명` 는 *{{ $json.output.merchant_name }}* 표현식으로 입력
      * `합계` 는 *{{ $json.output.total_amount }}* 표현식으로 입력

-----

