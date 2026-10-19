# 🏛️ AI 기반 공공조달 RFP 분석 RAG 시스템 (AI-based-RFP-RAG-System)

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-3776AB?style=flat-square&logo=python&logoColor=white"/>
  <img src="https://img.shields.io/badge/gemma4:e4b-Ollama_Local-000000?style=flat-square&logo=ollama&logoColor=white"/>
  <img src="https://img.shields.io/badge/ChromaDB-Vector_DB-FF6B35?style=flat-square"/>
  <img src="https://img.shields.io/badge/Qwen3-Embedding-412991?style=flat-square"/>
  <img src="https://img.shields.io/badge/RAGAS-Evaluation-009ACD?style=flat-square"/>
</p>
 
<p align="center">
  <b>AI8기 중급 프로젝트 | Team 1</b><br/>
  AI-based RFP RAG System for public procurement document analysis <br><br>
  유연정(PM) · 정설아(데이터처리) · 정봄(데이터처리) · 박찬영(Retrieval) · 박민성(Generation)
</p>

<p align="center">
  <a href="https://www.notion.so/AI8-_-_1-RFP-X-35e1fffab02c80d9ae52daa9a6b5fa5a">📋 협업노트/회의록 & 프로젝트 가이드</a>
  　<a href="./RFP-X_%EC%B5%9C%EC%A2%85%EB%B3%B4%EA%B3%A0%EC%84%9C_Team1_v9.pdf">📄 팀 보고서 PDF</a>
</p>

---

## 📌 프로젝트 개요

공공기관 발주 **RFP(제안요청서) 문서**는 방대한 분량과 복잡한 행정 언어로 구성되어 있어, 담당자가 핵심 조건을 빠르게 파악하기 어렵다는 실무적 문제가 있습니다.

본 프로젝트는 **RAG(Retrieval-Augmented Generation) 파이프라인**을 구축하여 공공조달 RFP 문서에 대한 정확한 질의응답 시스템을 개발하고, 이를 체계적인 **평가 데이터셋**으로 검증합니다.

| 항목 | 내용 |
|------|------|
| 데이터 출처 | 제공된 데이터셋 (HWP 665개 + PDF 25개) |
| 평가 데이터셋 | `pm_data.xlsx` (Q001–Q500, 5개 유형) |
| 평가 범위 | eval_batch 01–25 |
| 핵심 과제 | RAG 파이프라인 구축 + RAGAS 기반 성능 평가 |

---

## 🗂️ 파일 구조

```
AI-based-RFP-RAG-System/
│
├── final_code_com_03.ipynb         # ★ 최종 통합 파이프라인 — 검색·생성·평가 (박민성)
├── service/                        # ★ 최종 서비스 — Streamlit BidMate
├── RFP-X_최종보고서_Team1_v9.pdf   # ★ 최종 프로젝트 보고서
│
├── notebooks/                      # 단계별 작업 노트북
│   ├── final_data_01.ipynb         #   HWP/PDF 파싱, 청킹, ChromaDB 구축 (정설아·정봄)
│   ├── retrieval_V9.ipynb          #   Retrieval 파이프라인 (박찬영)
│   └── prom09.ipynb                #   프롬프트 엔지니어링 및 Generation 파이프라인 (박민성)
│
├── archive/                        # 이전 버전
│   └── com_02.ipynb                #   통합 파이프라인 이전 버전
│
├── docs/                           # 보고서·가이드
│   ├── generation_final_report.md  #   Generation 파트 실험 보고서
│   └── project_code_guide.md       #   서비스 구현 가이드
│
├── data/                           # 문서 메타데이터 · 평가셋 (pm_data.xlsx)
│
├── .gitignore
└── README.md
```

★ 표시가 프로젝트의 최종 결과물입니다. 나머지는 단계별 작업 기록입니다.

---

## 🔧 기술 스택

### 핵심 파이프라인

| 구성 요소 | 기술 |
|----------|------|
| **HWP 파싱** | olefile + zlib 직접 파싱 |
| **PDF 파싱** | PyMuPDF (fitz), pdfplumber |
| **텍스트 청킹** | 커스텀 청킹 (chunk_size=1024, overlap=200) |
| **임베딩 모델** | Qwen/Qwen3-Embedding-0.6B (로컬) |
| **벡터 DB** | ChromaDB (cosine similarity) |
| **검색 방식** | Hybrid Search (BM25 + Dense + RRF) + BGE-Reranker |
| **생성 모델** | gemma4:e4b (Ollama 로컬, 서비스용) / EXAONE-3.5 7.8B (프롬프트 실험용) |
| **서빙** | Streamlit (BidMate UI) |

### 평가 프레임워크

| 구성 요소 | 기술 |
|----------|------|
| **평가 프레임워크** | RAGAS + 커스텀 지표 |
| **평가 지표** | Faithfulness · Answer Relevancy · Context Precision · False Refusal Rate · Wrong Amount Rate |
| **QA 데이터셋** | GPT 기반 자동 생성 + 수동 검수 |
| **데이터 포맷** | `.xlsx` (Q001–Q500, 5개 질문 유형) |

### 질문 유형 분류 (Type A–E)

| 유형 | 설명 |
|------|------|
| **Type A** | 단순 사실 확인 (수치, 날짜, 기관명 등) |
| **Type B** | 다중 문서 비교 / 계산 |
| **Type C** | 후속 질문 (대화 히스토리 필요) |
| **Type D** | Hallucination 검증 (존재하지 않는 정보 질문) |
| **Type E** | 오타 / 구어체 포함 (fuzzy matching 대응) |

---

## 🚀 설치 및 실행

### 1. 환경 설정

```bash
git clone https://github.com/malit9999/AI-based-RFP-RAG-System.git
cd AI-based-RFP-RAG-System
pip install -r service/requirements.txt
```

### 2. Ollama 모델 설치

```bash
# Ollama 설치
curl -fsSL https://ollama.com/install.sh | sh

# 서비스용 LLM
ollama pull gemma4:e4b

# 프롬프트 실험용 LLM (선택)
ollama pull hf.co/lmstudio-community/EXAONE-3.5-7.8B-Instruct-GGUF
```

### 3. 실행 순서

```
1. notebooks/final_data_01.ipynb   → HWP/PDF 파싱, 청킹, ChromaDB 구축
2. notebooks/retrieval_V9.ipynb    → Hybrid Search 검색 파이프라인 실행
3. notebooks/prom09.ipynb          → 프롬프트 설계 및 Generation 파이프라인
4. final_code_com_03.ipynb         → 통합 평가 파이프라인 (RAGAS + 커스텀 지표)
5. service/                        → Streamlit BidMate UI 실행
```

---

## 📊 RAG 파이프라인 구조

```
[HWP / PDF 문서]
      │
      ▼
[문서 파싱 & 청킹]  ←  olefile / pdfplumber, chunk_size=1024
      │
      ▼
[임베딩 생성]  ←  Qwen3-Embedding-0.6B (로컬)
      │
      ▼
[ChromaDB 벡터 DB 저장]  ←  51,465개 청크
      │
      ▼
[사용자 질의]
      │
      ▼
[Hybrid Search]  ←  BM25 + Dense + RRF Fusion
      │
      ▼
[gemma4:e4b 답변 생성]  ←  Ollama 로컬, Tool Calling 지원
      │
      ▼
[RAGAS + 커스텀 지표 평가]  ←  False Refusal / Wrong Amount 등
```

---

## 📁 평가 데이터셋

- **파일명**: `pm_data.xlsx`
- **규모**: 500개 QA 쌍 (Q001–Q500)
- **생성 방식**: GPT 기반 자동 생성 → 팀 수동 검수
- **범위**: eval_batch 01–25 (강사 가이드라인 기준)
- **구성**: question / answer / context / source_doc / question_type

---

## 👥 팀 구성 및 역할

| 이름 | 역할 |
|------|------|
| 유연정 | PM · 평가 설계 총괄 · eval_batch 통합 검수 |
| 정설아 | 데이터 전처리 · HWP 파싱 · ChromaDB 구축 |
| 정봄 | 데이터 전처리 · PDF 파싱 · ChromaDB 구축 |
| 박찬영 | Retrieval · Hybrid Search 구현 · 검색 파이프라인 V1~V12 |
| 박민성 | Generation · 프롬프트 엔지니어링 · 통합 파이프라인 · Streamlit 서빙 |

---

## 📎 참고 자료

- [📋 협업 노트 / 회의록 (Notion)](https://www.notion.so/AI8-_-_1-RFP-X-35e1fffab02c80d9ae52daa9a6b5fa5a)
- [RAGAS 공식 문서](https://docs.ragas.io)
- [ChromaDB 공식 문서](https://docs.trychroma.com)
- [중급프로젝트 가이드라인](https://codeit.notion.site/AI-1ee6fd228e8d80d4834bee9cef8f44c1)

---

<p align="center">
  <sub>AI8기 중급 프로젝트 | 2026 · Team 1</sub>
</p>
