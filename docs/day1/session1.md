## Session 1: 왜 지금 Agent 인가?

-----

### 1. 목표

* 생성형 AI를 넘어 '행동하는 AI(Agent)'의 개념을 이해합니다.
* 비용 효율적이고 보안에 강한 Local LLM 환경(Podman AI Lab)을 구축합니다.
* 노코드/로우코드 자동화 툴인 n8n의 환경을 구축하고 Local LLM과 연결할 준비를 마칩니다.
* n8n의 핵심 작동 원리(Node, Flow, JSON)를 마스터합니다.

-----

### 2. 왜 지금 Agent인가?

#### 2.1. 생성형 AI의 진화 (Chat vs Agent)

지금까지의 ChatGPT나 Claude는 똑똑한 **비서(Brain)** 였지만, 팔다리가 없어 직접 업무를 수행할 수는 없었습니다. AI Agent는 이 두뇌에 **도구(Tools)** 를 쥐여주어 실제로 일을 수행하게 만드는 단계입니다.

* Generative AI (LLM): 텍스트, 이미지 생성, 요약, 번역 (생각하고 말하기)
* AI Agent: LLM + 외부 도구 연동 + 자율적 판단 (생각하고 행동하기)

* **Agent vs LLM** 쉬운 비교

| 개념    | 쉽게 풀어쓴 설명                           |
| ----- | ----------------------------------- |
| LLM   | **똑똑한 비서**: 질문에 답은 하지만 직접 일은 못 함.   |
| Agent | **비서 + 도구(손발)**: 스스로 도구를 골라 일을 수행함. |


#### 2.2. Agent가 일하는 방식 (ReAct)

Agent는 단순히 명령을 따르는 것이 아니라, 목표를 달성하기 위해 **추론(Reasoning)** 하고 **행동(Acting)** 합니다.

1. 지각 (Perception): 사용자 요청 및 환경 인식
2. 계획 (Planning): 어떤 도구를 어떤 순서로 쓸지 결정
3. 행동 (Action): API 호출, 검색, 데이터베이스 조회
4. 피드백 (Feedback): 결과 확인 및 수정

* *ReAct(Reasoning + Acting)*는 “생각 → 행동 → 결과 확인 → 다시 생각”의 **사이클**

#### 2.3. 왜 n8n 인가?

AI Agent를 구현하기 위해서는 LLM과 외부 시스템(Slack, Gmail, DB 등)을 연결하는 파이프라인이 필요합니다. n8n은 이를 학습하기 위한 최적의 도구입니다.

* 유연성: 1,000개 이상의 연동 가능한 앱
* 보안: 자체 호스팅(Self-hosted) 가능 (사내 데이터 보안 유리)
* AI 특화: LangChain 노드 등 강력한 AI 기능 내장

---

### 3. [실습] 나만의 AI Brain 구축하기 (Local LLM)

n8n Agent의 '두뇌' 역할을 할 LLM을 내 PC에 직접 설치합니다. OpenAI API 비용 절감과 데이터 보안을 위해 **Podman AI Lab**을 사용합니다.

#### 3.1. Podman Desktop 설치

Docker의 대안인 오픈소스 컨테이너 엔진 Podman을 설치합니다.
* 인텔 기준 4세대 이상 프로세서 필요
* **다운로드:** [Podman Desktop 공식 홈페이지](https://podman-desktop.io/)
* **설치:** OS(Windows/Mac)에 맞는 설치 파일 실행 및 기본 설정 완료
* ( 윈도우인 경우 WSL 2 필요: https://learn.microsoft.com/en-us/windows/wsl/install-manual )

#### 3.2. AI Lab 확장 프로그램 구성

Podman Desktop 내에서 LLM을 쉽게 다운로드하고 실행할 수 있는 `AI Lab` 확장을 활성화합니다.

1. Podman Desktop 실행 -> 좌측 메뉴 **Extensions (퍼즐 아이콘)** 클릭
2. Catalog 탭에서 **'Podman AI Lab'** 검색 후 설치 (Install)

#### 3.3. Model 다운로드 및 로드

실습에 사용할 경량화된 오픈소스 모델을 로드합니다.

* **Target Model:** `google/gemma-3n-E4B (Unsloth quantization)`



1. AI Lab 메뉴 -> **Catalog** 선택
2. 모델 검색창에 `google/gemma-3n-E4B (Unsloth quantization)` 입력
* *Tip: Unsloth로 파인튜닝된 특정 모델 파일(GGUF)을 가지고 있다면 'Import Model' 기능을 통해 직접 파일을 불러올 수 있습니다.*



#### 3.4. AI Model Service 실행 (Local API Server)

다운로드한 모델을 n8n이 접속할 수 있도록 API 서버로 띄웁니다.

1. AI Lab -> **Services** (또는 Local Server) 탭 클릭
2. **New Model Service** 클릭 -> 다운로드한 Gemma 모델 선택
3. **Container port** 임의 지정 (예: 11000 포트)
4. **Create Service** 클릭
5. **Open Service Details** 클릭 후, **Endpoint 확인:** `http://localhost:11000/v1`
* * n8n 'OpenAI Chat Model' 노드를 위한 설정 시에는 컨테이너 내에서 호출되므로, http://host.containers.internal:11000/v1 로 바꿔서 기록해놓으세요.*
6. 예제 Client code 나 브라우저에서 Endpoint 주소를 사용하여, Service 응답을 테스트 해 볼 수 있습니다.



---

### 4. LLM vs Agent 차이점 (요약)

* **LLM:** 텍스트 생성에 특화 (생각)
* **Agent:** 도구(Tool)를 사용하여 실제 행동을 수행 (생각 + 행동)

---

### 5. Local LLM 주의점

* **Local LLM:** 내 노트북의 자원에 따라 성능이 달라집니다. 자원이 부족한 경우 동작이 불가능할 수 있습니다.
  * Podman 으로 LLM 실행하기 위해서 최소 16GB의 메모리와 최소 4개의 CPU를 권장합니다. 
* 클라우드 LLM: 외부 서비스 제공자에 의해서 관리되는 LLM 입니다. 유료 서비스이므로 가입 및 구독 절차가 필요합니다.

---
