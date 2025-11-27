# 📘 OpenSearch 벡터 검색 구축 기술 매뉴얼
> **쉽게 이해하고, 개발자도 바로 쓸 수 있는 실전 가이드**

이 문서는 Windows 환경에서 WSL, Docker를 활용하여 OpenSearch 벡터 검색 엔진(k-NN)을 구축하고, 텍스트를 벡터로 변환하여 유사한 문장을 검색하는(FAQ 검색기 등) 전체 과정을 설명합니다.

---

## 📑 목차 (Table of Contents)

1. [기초 개념: 도구 알아보기](#1-기초-개념-도구-알아보기)
   - WSL이란?
   - Ubuntu란?
   - Docker란?
2. [환경 구성: OpenSearch 실행하기](#2-환경-구성-opensearch-실행하기)
   - Docker Compose 설정
   - 실행 및 접속 확인
3. [핵심 기능: OpenSearch란?](#3-핵심-기능-opensearch란)
4. [ML 설정: 뇌 장착하기 (ML Commons)](#4-ml-설정-뇌-장착하기-ml-commons)
5. [모델 등록: HuggingFace 모델 가져오기](#5-모델-등록-huggingface-모델-가져오기)
6. [벡터 변환: 텍스트를 숫자로](#6-벡터-변환-텍스트를-숫자로)
7. [인덱스 생성: k-NN 인덱스](#7-인덱스-생성-k-nn-인덱스)
8. [데이터 저장: 텍스트와 벡터 저장](#8-데이터-저장-텍스트와-벡터-저장)
9. [검색: k-NN 유사도 검색](#9-검색-k-nn-유사도-검색)
10. [전체 흐름 정리](#10-전체-흐름-정리)
11. [⚠️ 운영 시 주의 사항 (Check Points)](#11-️-운영-시-주의-사항-check-points)

---

## 1. 기초 개념: 도구 알아보기

### 1.1 WSL (Windows Subsystem for Linux)이란?
> **"윈도우 안의 리눅스 통역사"**

- **개념**: 윈도우(Windows) 운영체제 위에서 리눅스(Linux) 프로그램을 실행할 수 있게 해주는 **가상 통역기** 같은 시스템입니다.
- **왜 필요한가요?**: 개발 서버나 도구(Docker, OpenSearch 등)는 리눅스 환경에서 가장 잘 작동합니다. 윈도우를 밀어버리지 않고도 리눅스의 힘을 빌려 쓰기 위해 사용합니다.

### 1.2 Ubuntu란?
> **"가장 친절한 리눅스 운영체제"**

- **개념**: 리눅스(Linux)의 여러 종류(배포판) 중 하나입니다.
- **필요한 이유**: 개발자들이 가장 많이 쓰는 표준적인 개발 환경입니다. WSL 위에서 돌아가는 '실제 작업 공간'이 됩니다.

### 1.3 Docker란?
> **"프로그램을 담는 마법의 컨테이너 박스"**

- **개념**: 복잡한 설치 과정 없이 프로그램을 **컨테이너**라는 상자에 담아 어디서든 똑같이 실행하게 해주는 도구입니다.
- **왜 필요한가요?**: OpenSearch를 설치하려면 자바 설치, 환경 변수 설정 등 복잡한 과정이 필요한데, Docker를 쓰면 명령어 한 줄로 "OpenSearch 상자"를 가져와서 바로 실행할 수 있습니다.

---

## 2. 환경 구성: OpenSearch 실행하기

Docker를 이용해 OpenSearch와 시각화 도구인 Dashboards를 실행합니다.

### 2.1 docker-compose.yml 작성
이 파일은 "어떤 컨테이너들을 어떻게 실행할지" 적어둔 **설계도**입니다.

```yaml
version: '3'
services:
  # 1. 검색 엔진 (OpenSearch)
  opensearch-node1:
    image: opensearchproject/opensearch:2.11.0
    container_name: opensearch-node1
    environment:
      - discovery.type=single-node  # 테스트용으로 서버 1대만 사용
      - bootstrap.memory_lock=true  # 메모리 성능 최적화
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" # 자바 메모리 설정 (최소/최대)
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=YourStrongPassword123! # 접속 비밀번호
    ports:
      - 9200:9200 # 외부 통신 포트
      - 9600:9600 # 성능 분석 포트
    ulimits:
      memlock:
        soft: -1
        hard: -1

  # 2. 시각화 도구 (OpenSearch Dashboards)
  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:2.11.0
    container_name: opensearch-dashboards
    ports:
      - 5601:5601 # 대시보드 접속 포트
    environment:
      OPENSEARCH_HOSTS: '["https://opensearch-node1:9200"]' # 연결할 엔진 주소
    depends_on:
      - opensearch-node1 # 엔진이 켜진 뒤에 실행
```

### 2.2 실행 명령어
터미널(Ubuntu/WSL)에서 파일이 있는 폴더로 이동 후 실행합니다.

```bash
# 컨테이너 실행 (백그라운드 모드)
docker-compose up -d
```

### 2.3 결과 확인
- **OpenSearch 상태 확인**: 브라우저에서 `https://localhost:9200` 접속 (로그인 필요)
- **대시보드 접속**: 브라우저에서 `http://localhost:5601` 접속
- **계정**: `admin` / `YourStrongPassword123!`

---

## 3. 핵심 기능: OpenSearch란?
> **"똑똑한 검색 엔진 & 데이터 저장소"**

- 단순한 단어 검색뿐만 아니라, **로그 저장**, **텍스트 분석**, **머신러닝(ML)**, 그리고 오늘 다룰 **벡터 검색(Vector Search)** 기능을 제공합니다.
- 우리가 만들 **FAQ 검색기**의 두뇌 역할을 합니다.

---

## 4. ML 설정: 뇌 장착하기 (ML Commons)

OpenSearch가 머신러닝 모델을 사용할 수 있도록 설정을 켜줘야 합니다.

### 4.1 설정 확인 및 활성화
Dev Tools(대시보드 내 콘솔) 또는 Curl 명령어로 설정을 변경합니다.

```json
PUT _cluster/settings
{
  "persistent": {
    "plugins.ml_commons.only_run_on_ml_node": false,
    "plugins.ml_commons.native_memory_threshold": 100,
    "plugins.ml_commons.model_access_control_enabled": true
  }
}
```
- **설명**: "ML 모델을 이 노드에서 실행해도 좋아"라고 허락해주는 과정입니다.

---

## 5. 모델 등록: HuggingFace 모델 가져오기

텍스트를 벡터(숫자)로 바꿔줄 **AI 모델**을 등록합니다.

### 5.1 모델 등록 (Register)
HuggingFace라는 AI 모델 저장소에서 모델을 가져옵니다.

```json
POST /_plugins/_ml/models/_register
{
  "name": "huggingface/sentence-transformers/all-MiniLM-L6-v2",
  "version": "1.0.1",
  "model_format": "TORCH_SCRIPT"
}
```
- **결과**: `task_id`가 반환됩니다. 이 ID로 모델이 잘 다운로드되었는지 확인합니다.

### 5.2 모델 배포 (Deploy)
등록된 모델을 메모리에 올려서 사용할 준비를 합니다.

```json
POST /_plugins/_ml/models/{model_id}/_deploy
```

---

## 6. 벡터 변환: 텍스트를 숫자로

### 6.1 벡터(Vector)란?
- 컴퓨터는 "사과"라는 글자의 의미를 모릅니다.
- 대신 `[0.1, -0.5, 0.8 ...]` 같은 숫자 리스트로 바꿔주면, 숫자가 비슷한 단어끼리 의미가 비슷하다고 이해합니다.
- 이를 **임베딩(Embedding)**이라고 합니다.

### 6.2 변환 테스트 (Predict API)
```json
POST /_plugins/_ml/_predict/text_embedding/{model_id}
{
  "text_docs":[ "오늘 날씨가 참 좋네요" ],
  "return_number": true,
  "target_response": ["sentence_embedding"]
}
```
- **결과**: 384개의 숫자로 이루어진 배열(Vector)이 나옵니다. (모델마다 차원 수가 다름, 예: 384차원)

---

## 7. 인덱스 생성: k-NN 인덱스

데이터를 저장할 **저장소(Index)**를 만듭니다. 이때 "여기는 벡터 검색을 할 거야"라고 알려줘야 합니다.

```json
PUT /my-faq-index
{
  "settings": {
    "index.knn": true  // k-NN 검색 활성화
  },
  "mappings": {
    "properties": {
      "my_vector": {
        "type": "knn_vector",
        "dimension": 384,  // 모델이 만드는 벡터 크기와 일치해야 함!
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

---

## 8. 데이터 저장: 텍스트와 벡터 저장

질문과 그 질문의 벡터 값을 함께 저장합니다.

```json
POST /my-faq-index/_doc/1
{
  "my_text": "비밀번호를 잊어버렸어요",
  "my_vector": [0.12, 0.34, -0.55, ...] // 실제로는 모델로 변환된 384개 숫자
}
```
> **Tip**: 실제 운영에서는 'Ingest Pipeline'을 써서 텍스트만 넣으면 자동으로 벡터로 변환되어 저장되게 구성합니다.

---

## 9. 검색: k-NN 유사도 검색

사용자가 "패스워드 분실"이라고 검색하면, 이를 벡터로 바꾼 뒤 가장 가까운(유사한) 질문을 찾습니다.

```json
GET /my-faq-index/_search
{
  "size": 2,
  "query": {
    "knn": {
      "my_vector": {
        "vector": [0.15, 0.31, -0.50, ...], // "패스워드 분실"의 벡터 값
        "k": 2
      }
    }
  }
}
```
- **원리**: 공간상에서 거리가 가장 가까운 데이터를 찾아냅니다. (유사도 기반)

---

## 10. 전체 흐름 정리

```mermaid
graph TD
    A[Windows 사용자] -->|WSL 실행| B(Ubuntu 리눅스 환경)
    B -->|Docker 실행| C{OpenSearch 컨테이너}
    C -->|1. 모델 로딩| D[ML Commons (AI 모델)]
    
    User[사용자 질문] -->|API 요청| C
    C -->|2. 텍스트를 벡터로 변환| D
    D -->|3. 벡터 값 반환| C
    C -->|4. k-NN 검색 (유사도 비교)| E[(k-NN 인덱스 데이터)]
    E -->|5. 가장 유사한 답변 찾기| C
    C -->|6. 결과 반환| User
```

1. **기반**: Windows 위에 **WSL/Ubuntu**를 깔고, 그 위에 **Docker**로 **OpenSearch**를 띄웁니다.
2. **준비**: OpenSearch에 **ML 모델**을 등록하고, 데이터를 **벡터화**해서 저장해둡니다.
3. **사용**: 사용자가 질문하면 → 벡터로 변환 → 저장된 데이터 중 가장 비슷한 것을 찾아(k-NN) → 답변을 줍니다.

---

## 11. ⚠️ 운영 시 주의 사항 (Check Points)

시스템을 운영할 때 다음 사항들을 반드시 체크하세요.

1.  **Docker 메모리 부족 (OOM)**
    *   **증상**: OpenSearch 컨테이너가 자꾸 죽음 (Exited with code 137).
    *   **해결**: WSL 자체 메모리 할당을 늘리거나(`.wslconfig`), Docker Desktop 설정에서 메모리를 4GB 이상으로 늘리세요.
2.  **Embedding Dimension 불일치**
    *   **주의**: 모델이 만드는 벡터 크기(예: 384)와 인덱스 설정(`dimension: 384`)이 **정확히 일치**해야 합니다. 다르면 저장 시 에러가 납니다.
3.  **모델 다운로드 실패**
    *   **원인**: OpenSearch 컨테이너가 인터넷에 연결되지 않았을 때 발생.
    *   **해결**: 기업 내부망이라면 프록시 설정을 확인하거나, 오프라인 모델 등록 방식을 써야 합니다.
4.  **디스크 용량**
    *   벡터 데이터는 일반 텍스트보다 용량을 많이 차지합니다. 디스크 공간을 넉넉히 확보하세요.
5.  **보안 설정 (Security)**
    *   예제에서는 `admin` 계정을 썼지만, 실제 서비스에서는 반드시 권한이 분리된 계정을 사용하고 HTTPS 인증서를 관리해야 합니다.

---
*작성일: 2025.11.27 | 작성자: Antigravity*
