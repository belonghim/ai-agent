## Session 2: Hello n8n (환경 설정 및 시작)

-----

### 1. 실행 환경 설정
n8n은 Podman을 통해 로컬에  쉽게 설치할 수 있습니다.

```bash
podman run -d --name n8n -p 5678:5678 -e GENERIC_TIMEZONE=Asia/Seoul -e TZ=Asia/Seoul -e N8N_RUNNERS_ENABLED=true -e N8N_PROTOCOL=http -e WEBHOOK_URL=http://localhost:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

-----

### 2. n8n 인터페이스 훑어보기

n8n의 캔버스(Editor UI)는 직관적입니다.

* Workflow: 하나의 자동화 작업 단위 (파일 하나라고 생각하면 됩니다).
* Nodes Panel: 사용할 수 있는 기능(앱)들을 검색하는 곳.
* Canvas: 노드들을 배치하고 선으로 연결하는 작업 공간.
* Executions: 실행 로그를 확인하고 디버깅하는 곳.

-----

### 3. [실습] 가장 기본 워크플로우 만들기

목표: "버튼을 누르면 'Hello World'를 출력한다."

1. Trigger 추가: Manual Trigger 노드 검색 및 추가.
2. Action 추가: Edit Fields 노드 추가.
3. 데이터 입력:
    * Name: message
    * Value: Hello n8n Agent!
4. 연결: 두 노드를 선으로 연결.
5. 실행: 하단의 Test Workflow 버튼 클릭.
