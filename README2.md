# 📘 WSL + Ubuntu + Docker + OpenSearch + ML 모델 ZIP 등록 완전 정복 매뉴얼

이 문서는 **Windows 환경**에서 **WSL, Ubuntu, Docker**를 사용하여 **OpenSearch**를 구축하고, **외부 인터넷 없이(오프라인 환경 등)** **ML 모델(HuggingFace)**을 **ZIP 파일**로 직접 등록하여 사용하는 전체 과정을 다룹니다.

초등학생도 이해할 수 있는 개념 설명부터, 개발자가 바로 복사해서 쓸 수 있는 명령어까지 모두 포함되어 있습니다.

---

## 📑 목차
1. [전체 개요](#1-전체-개요)
2. [왜 이런 구조가 필요한가요? (초등학생도 이해하는 비유)](#2-왜-이런-구조가-필요한가요-초등학생도-이해하는-비유)
3. [전체 시스템 흐름도](#3-전체-시스템-흐름도)
4. [설치 단계 요약](#4-설치-단계-요약)
5. [환경 준비 (WSL, Ubuntu, Docker)](#5-환경-준비-wsl-ubuntu-docker)
6. [OpenSearch & Dashboards 실행](#6-opensearch--dashboards-실행)
7. [ML Commons 설정 확인](#7-ml-commons-설정-확인)
8. [ZIP 파일 준비 (HuggingFace)](#8-zip-파일-준비-huggingface)
9. [SHA256 해시 계산 및 이유](#9-sha256-해시-계산-및-이유)
10. [HTTP 서버로 ZIP 제공](#10-http-서버로-zip-제공)
11. [모델 등록 (Register)](#11-모델-등록-register)
12. [모델 배포 (Deploy)](#12-모델-배포-deploy)
13. [예측 테스트 (Predict)](#13-예측-테스트-predict)
14. [k-NN 인덱스 생성 및 검색](#14-k-nn-인덱스-생성-및-검색)
15. [에러 처리 (Troubleshooting)](#15-에러-처리-troubleshooting)
16. [운영 환경 팁](#16-운영-환경-팁)
17. [전체 요약](#17-전체-요약)

---

## 1. 전체 개요

우리는 **OpenSearch**라는 검색 엔진에 **인공지능(AI) 두뇌**를 달아줄 것입니다. 보통은 OpenSearch가 인터넷에서 AI 모델을 직접 다운로드하지만, **보안이 철저한 회사 내부망(폐쇄망)**이나 **인터넷이 느린 환경**에서는 이것이 불가능합니다.

따라서 우리는 AI 모델을 **미리 ZIP 파일로 다운로드** 받아두고, 내 컴퓨터를 **임시 파일 서버**로 만들어서 OpenSearch가 내 컴퓨터에서 파일을 가져가도록 만들 것입니다.

---

## 2. 왜 이런 구조가 필요한가요? (초등학생도 이해하는 비유)

> **🏫 비유: "전학 온 천재 친구와 교과서"**

상상해 보세요. 우리 반(OpenSearch)에 **엄청 똑똑한 전학생(AI 모델)**이 오기로 했습니다. 이 친구는 모든 질문에 답할 수 있는 능력이 있어요.

그런데 이 친구는 **"마법의 교과서(모델 파일)"**가 없으면 공부를 할 수 없습니다. 보통은 서점(인터넷/HuggingFace)에 가서 사오면 되는데, 우리 학교는 **산속에 있어서(폐쇄망/오프라인)** 서점에 갈 수가 없어요!

그래서 선생님(개발자)이 미리 도시에서 **마법의 교과서를 사서 가방(ZIP 파일)에 담아** 왔습니다. 그리고 교실 앞 탁자(내 컴퓨터의 HTTP 서버)에 올려두고 전학생에게 말합니다.

**"서점에 가지 말고, 여기 탁자에 있는 교과서를 가져가서 공부해!"**

이 과정이 바로 **ZIP 파일 등록 방식**입니다.
1.  **서점**: HuggingFace (인터넷)
2.  **가방**: ZIP 파일
3.  **탁자**: Python HTTP Server (내 컴퓨터)
4.  **전학생**: OpenSearch

---

## 3. 전체 시스템 흐름도

```mermaid
graph TD
    User[개발자 (Windows)] -->|1. ZIP 다운로드| HF[HuggingFace]
    User -->|2. HTTP 서버 실행| LocalServer[내 컴퓨터 (Python Server :8000)]
    User -->|3. 모델 등록 명령 (API)| OS[OpenSearch (Docker)]
    
    subgraph Docker Network
        OS -->|4. http://host.docker.internal:8000/model.zip 요청| LocalServer
    end
    
    LocalServer -->|5. ZIP 파일 전송| OS
    OS -->|6. 압축 해제 및 모델 로딩| OS_Mem[메모리]
```

---

## 4. 설치 단계 요약

1.  **WSL 설치**: 윈도우 안에 리눅스를 설치합니다.
2.  **Docker 설치**: 프로그램을 컨테이너라는 상자에 담아 실행하는 도구입니다.
3.  **OpenSearch 실행**: 도커를 이용해 검색 엔진을 켭니다.
4.  **모델 준비**: AI 모델을 ZIP으로 받고 해시값을 땁니다.
5.  **서버 띄우기**: ZIP 파일을 OpenSearch가 가져갈 수 있게 합니다.
6.  **API 호출**: 등록 -> 배포 -> 사용 순서로 명령을 내립니다.

---

## 5. 환경 준비 (WSL, Ubuntu, Docker)

### 5.1 WSL 및 Ubuntu 설치
윈도우 PowerShell을 **관리자 권한**으로 실행하고 아래 명령어를 입력하세요.

```powershell
# WSL과 Ubuntu를 한 번에 설치합니다.
wsl --install
```
*   설치 후 **재부팅**이 필요할 수 있습니다.
*   재부팅 후 Ubuntu 창이 뜨면 `username`과 `password`를 설정하세요.

### 5.2 Docker Desktop 설치
1.  [Docker Desktop 공식 홈페이지](https://www.docker.com/products/docker-desktop/)에서 Windows 버전을 다운로드 및 설치합니다.
2.  설치 후 설정(Settings) -> **Resources** -> **WSL Integration**에서 `Ubuntu` 스위치를 켜주세요.

---

## 6. OpenSearch & Dashboards 실행

WSL(Ubuntu) 터미널을 열고 아래 명령어로 OpenSearch와 Dashboards를 실행합니다.

### 6.1 `docker-compose.yml` 작성
```bash
# 작업 디렉토리 생성
mkdir -p ~/opensearch-lab && cd ~/opensearch-lab

# docker-compose.yml 파일 생성
nano docker-compose.yml
```

아래 내용을 붙여넣으세요. (ML 기능을 위해 메모리 제한을 해제하고 환경변수를 설정했습니다.)

```yaml
version: '3'
services:
  opensearch-node:
    image: opensearchproject/opensearch:2.11.0
    container_name: opensearch-node
    environment:
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g"
      - "DISABLE_INSTALL_DEMO_CONFIG=true"
      - "plugins.ml_commons.only_run_on_ml_node=false"
      - "plugins.ml_commons.native_memory_threshold=100"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    ports:
      - 9200:9200
    networks:
      - opensearch-net

  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:2.11.0
    container_name: opensearch-dashboards
    ports:
      - 5601:5601
    expose:
      - "5601"
    environment:
      OPENSEARCH_HOSTS: '["https://opensearch-node:9200"]'
      DISABLE_SECURITY_DASHBOARDS_PLUGIN: "true"
    networks:
      - opensearch-net
    depends_on:
      - opensearch-node

networks:
  opensearch-net:
```

### 6.2 실행
```bash
docker-compose up -d
```
*   약 1~2분 후 `https://localhost:9200` 접속 시 로그인 창(admin/admin)이 뜨면 성공입니다.
*   Dashboards는 `http://localhost:5601` 입니다.

---

## 7. ML Commons 설정 확인

OpenSearch Dashboards의 **Dev Tools** (주소창에 `http://localhost:5601/app/dev_tools#/console`)로 이동하여 아래 설정을 입력하고 실행(▶ 버튼)하세요.

```json
PUT _cluster/settings
{
  "persistent": {
    "plugins.ml_commons.only_run_on_ml_node": false,
    "plugins.ml_commons.native_memory_threshold": 100,
    "plugins.ml_commons.allow_registering_model_via_url": true,
    "plugins.ml_commons.allow_registering_model_via_local_file": true
  }
}
```
*   `allow_registering_model_via_url`: URL을 통해 모델을 등록할 수 있게 허용합니다.

---

## 8. ZIP 파일 준비 (HuggingFace)

우리가 사용할 모델은 문장을 벡터(숫자)로 바꿔주는 `huggingface/sentence-transformers/all-MiniLM-L6-v2` 모델입니다.

1.  HuggingFace 등에서 해당 모델의 **TorchScript 버전 ZIP 파일**을 다운로드합니다.
2.  또는 이미 가지고 있는 모델 ZIP 파일을 준비합니다.
3.  예시 파일명: `all-MiniLM-L6-v2_torchscript.zip`

**ZIP 파일 복사 (Windows -> WSL)**
PowerShell에서 실행:
```powershell
# C드라이브의 opensearch 폴더에 있는 zip 파일을 WSL 홈 디렉토리의 models 폴더로 복사
cp C:\opensearch\all-MiniLM-L6-v2_torchscript.zip \\wsl$\Ubuntu\home\username\models\
```

---

## 9. SHA256 해시 계산 및 이유

### 🧐 왜 해시(Hash)가 필요한가요?
OpenSearch는 보안을 위해 **"내가 다운로드하려는 파일이 해킹당하거나 변조되지 않은 원본 파일이 맞는지"** 확인합니다. SHA256 해시값은 파일의 **지문**과 같습니다. 파일 내용이 1비트만 바뀌어도 이 값은 완전히 달라집니다.

### 해시 계산 명령어

**Linux (WSL/Ubuntu)**
```bash
cd ~/models
sha256sum all-MiniLM-L6-v2_torchscript.zip
# 출력 예: a1b2c3d4...  all-MiniLM-L6-v2_torchscript.zip
```

**Windows (PowerShell)**
```powershell
Get-FileHash -Algorithm SHA256 .\all-MiniLM-L6-v2_torchscript.zip
```

> ⚠️ **중요**: 나온 해시값(긴 문자열)을 메모장에 복사해두세요!

---

## 10. HTTP 서버로 ZIP 제공

OpenSearch(도커 컨테이너)가 내 컴퓨터(호스트)에 있는 ZIP 파일을 가져가려면, 내 컴퓨터가 웹 서버 역할을 해야 합니다.

**WSL 터미널에서 실행 (ZIP 파일이 있는 폴더에서):**
```bash
cd ~/models
python3 -m http.server 8000
```
*   이제 `http://내IP:8000/파일명.zip` 주소로 파일에 접근할 수 있습니다.
*   **주의**: 이 터미널 창을 끄면 안 됩니다! (다운로드 끝날 때까지 유지)

---

## 11. 모델 등록 (Register)

이제 OpenSearch에게 "내 컴퓨터에 있는 ZIP 파일을 가져가서 등록해"라고 명령합니다.

**Dev Tools 입력:**
```json
POST /_plugins/_ml/models/_register
{
  "name": "huggingface/sentence-transformers/all-MiniLM-L6-v2",
  "version": "1.0.1",
  "model_format": "TORCH_SCRIPT",
  "model_config": {
    "model_type": "bert",
    "embedding_dimension": 384,
    "framework_type": "sentence_transformers",
    "all_config": "{\"architectures\":[\"BertModel\"],\"max_position_embeddings\":512,\"model_type\":\"bert\",\"num_attention_heads\":12,\"num_hidden_layers\":6}"
  },
  "url": "http://host.docker.internal:8000/all-MiniLM-L6-v2_torchscript.zip",
  "model_content_hash_value": "여기에_아까_복사한_SHA256_값을_붙여넣으세요"
}
```

### 💡 핵심 포인트: `host.docker.internal`
*   도커 컨테이너 안에서 `localhost`는 컨테이너 자신을 의미합니다.
*   내 컴퓨터(호스트)를 가리키려면 `host.docker.internal`이라는 특수 주소를 써야 합니다.

**응답 확인:**
```json
{
  "task_id": "AbCdEfG...",
  "status": "CREATED"
}
```
`task_id`를 복사하세요.

**상태 조회 및 Model ID 획득:**
```json
GET /_plugins/_ml/tasks/복사한_task_id
```
응답에서 `"model_id": "모델아이디_값"`을 찾아 복사하세요.

---

## 12. 모델 배포 (Deploy)

등록된 모델을 메모리에 올려서 실제로 사용할 수 있게 만듭니다.

**Dev Tools 입력:**
```json
POST /_plugins/_ml/models/복사한_model_id/_deploy
```

**상태 조회:**
```json
GET /_plugins/_ml/models/복사한_model_id
```
`"model_state": "DEPLOYED"`가 보이면 성공입니다!

---

## 13. 예측 테스트 (Predict)

모델이 문장을 잘 이해하는지 테스트해봅니다.

**Dev Tools 입력:**
```json
POST /_plugins/_ml/_predict/text_embedding/복사한_model_id
{
  "text_docs": [ "오늘 날씨가 참 좋네요", "OpenSearch is powerful" ],
  "return_number": true,
  "target_response": [ "sentence_embedding" ]
}
```
결과로 긴 숫자 배열(벡터)이 나오면 성공입니다.

---

## 14. k-NN 인덱스 생성 및 검색

이제 이 모델을 이용해 **의미 기반 검색(Vector Search)**을 해봅시다.

### 14.1 인덱스 생성 (저장소 만들기)
```json
PUT my_knn_index
{
  "settings": {
    "index.knn": true,
    "default_pipeline": "my-nlp-pipeline"
  },
  "mappings": {
    "properties": {
      "my_vector": {
        "type": "knn_vector",
        "dimension": 384,
        "method": {
          "name": "hnsw",
          "engine": "nmslib"
        }
      },
      "my_text": {
        "type": "text"
      }
    }
  }
}
```
*   **주의**: `default_pipeline`을 사용하려면 먼저 파이프라인(Ingest Pipeline)을 만들어야 합니다. 간단한 테스트를 위해 파이프라인 없이 직접 벡터를 넣거나, 파이프라인 생성 단계를 추가해야 합니다. (여기서는 핵심 흐름을 위해 생략하거나, 별도 파이프라인 생성 필요)

### 14.2 데이터 넣기 (Indexing)
*실제로는 텍스트를 모델에 넣어 벡터로 변환한 뒤 넣어야 합니다.*

### 14.3 검색 (Search)
```json
POST my_knn_index/_search
{
  "size": 2,
  "query": {
    "knn": {
      "my_vector": {
        "vector": [0.1, 0.2, ...], 
        "k": 2
      }
    }
  }
}
```
*(벡터 값은 `_predict` 결과에서 나온 값을 넣습니다)*

---

## 15. 에러 처리 (Troubleshooting)

### 🚨 1. Connection Refused (연결 거부)
*   **증상**: 모델 등록 시 URL 연결 실패 에러.
*   **원인**: `host.docker.internal`이 작동하지 않거나, 방화벽 문제.
*   **해결**:
    1.  Windows 방화벽에서 8000번 포트 허용.
    2.  Python 서버가 켜져 있는지 확인.
    3.  `http://내PC_IP주소:8000` 으로 변경해서 시도.

### 🚨 2. Hash Value Mismatch (해시 불일치)
*   **증상**: `model_content_hash_value` 에러.
*   **원인**: ZIP 파일이 손상되었거나, 해시값을 잘못 복사함.
*   **해결**: `sha256sum` 명령어로 다시 계산하여 정확히 입력.

### 🚨 3. Memory Issue (메모리 부족)
*   **증상**: 모델 배포 시 실패하거나 OpenSearch가 죽음.
*   **해결**: Docker Desktop 설정에서 메모리를 4GB 이상으로 늘려주세요. `docker-compose.yml`의 `OPENSEARCH_JAVA_OPTS`도 확인하세요.

---

## 16. 운영 환경 팁

1.  **방화벽**: 실제 서버에서는 8000번 포트가 외부에서 접근 가능한지, 혹은 내부망에서만 접근 가능한지 확인해야 합니다.
2.  **Nginx 사용**: Python `http.server`는 테스트용입니다. 운영 환경에서는 **Nginx**나 **Apache** 같은 정식 웹 서버를 사용하여 ZIP 파일을 호스팅하세요.
3.  **영구 저장**: Docker 컨테이너를 삭제하면 데이터가 날아갑니다. `volumes` 설정을 통해 데이터를 호스트에 저장하세요.

---

## 17. 전체 요약

1.  **환경**: Windows + WSL + Docker로 OpenSearch 구축.
2.  **문제**: 인터넷 없는 환경에서 AI 모델 등록 필요.
3.  **해결**:
    *   모델 ZIP 파일 준비 & 해시 계산.
    *   Python으로 임시 파일 서버 가동.
    *   `host.docker.internal` 주소로 OpenSearch가 내 컴퓨터의 파일을 가져가게 함.
4.  **결과**: 오프라인 상태에서도 최신 AI 모델을 활용한 벡터 검색 가능.

---


