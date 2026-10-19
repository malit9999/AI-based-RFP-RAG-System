# 입찰메이트 RFP 챗봇 서비스 — Claude Code 구현 지침서

> 이 문서는 Claude Code가 Streamlit 기반 RFP Q&A 챗봇 서비스를 구현할 때
> 참고하는 전체 명세서입니다. 반드시 처음부터 끝까지 읽고 시작하세요.

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 서비스명 | 입찰메이트 RFP 챗봇 |
| 목적 | B2G 입찰 담당자가 RFP 문서를 자연어로 질의하는 웹 챗봇 |
| 프레임워크 | Streamlit |
| 실행 방식 | `streamlit run app.py` → 브라우저 localhost:8501 접속 |
| 개발 환경 | Mac Mini M4 24GB, Python 3.12 |

---

## 2. 기술 스택

| 구성요소 | 상세 |
|----------|------|
| 웹 프레임워크 | Streamlit |
| LLM | gemma4:e4b (Ollama, localhost:11434) |
| 임베딩 | Qwen/Qwen3-Embedding-0.6B (HuggingFace, MPS) |
| 벡터DB | ChromaDB (PersistentClient, 코사인 유사도) |
| 하이브리드 검색 | Dense + BM25 + RRF (k=60) |
| 시각화 | Plotly |
| 데이터 | pandas, openpyxl |

---

## 3. 경로 설정

```python
BASE_PATH   = Path("/Users/who/Desktop/code_it/project01/final_files")
DATA_PATH   = BASE_PATH / "data"
CHROMA_PATH = str(BASE_PATH / "chroma_seol_qwen3")
FILES_PATH  = DATA_PATH / "files_advanced"       # 원본 HWP/PDF
CHUNKS_PATH = DATA_PATH / "chroma_export.json"   # 51,366개 청크
META_PATH   = DATA_PATH / "data_list_advanced.xlsx"

COLLECTION_NAME = "rfp_seol_chunks_qwen3"
LLM_MODEL       = "gemma4:e4b"
EMBEDDING_MODEL = "Qwen/Qwen3-Embedding-0.6B"
```

---

## 4. 핵심 함수 (수정 금지)

아래 함수들은 `com_04.ipynb`에서 추출하여 `core/functions.py`에 그대로 이식한다.
로직 변경 없이 import만 하여 사용할 것.

```
ask(query, history)                → dict   # 질문 → 답변 생성
retrieve(query, top_k)             → list   # 하이브리드 검색
get_embeddings(texts)              → list   # Qwen3 임베딩
get_agency_filter(query)           → tuple  # 발주기관 fuzzy 필터
ConversationManager                         # 히스토리 관리 V1
ConversationManagerV2                       # 히스토리 관리 V2 (rewrite)
parse_answer(answer)               → str    # **답변** 섹션 추출
get_section(text, name)            → str    # 섹션 추출
SYSTEM_PROMPT_V7E                           # 확정 시스템 프롬프트 (수정 금지)
TOOL_INSTRUCTION                            # Tool 호출 지시문 (수정 금지)
TOOL_SCHEMAS                                # Tool 스키마 6개 (수정 금지)
chat_ollama_with_tools(...)        → str    # Tool Calling LLM 호출
execute_tool(name, args)           → str    # Tool 실행
```

### ask() 반환 딕셔너리 구조

```python
{
    "query"            : str,
    "answer"           : str,          # **답변** / **근거** / **출처** 구조
    "contexts"         : list[str],    # 검색된 청크 텍스트
    "chunks"           : list[dict],   # 전체 검색 결과
    "answer_chunks"    : list[dict],   # LLM 입력 청크
    "evidence_card"    : str,          # 정답 후보 요약
    "used_deterministic": bool,
    "deterministic_type": str | None,
    "history"          : list[dict],
    "rewritten_query"  : str,          # V2에서 재작성된 쿼리
}
```

---

## 5. 앱 구조 (멀티페이지)

```
app.py                          ← 메인 챗봇
pages/
  1_대시보드.py                  ← 예산 분석 대시보드
  2_사업검색.py                  ← 유사 사업 추천
  3_북마크.py                    ← 즐겨찾기
core/
  __init__.py
  functions.py                  ← 핵심 함수 (ask, retrieve 등)
  utils.py                      ← 공통 유틸
assets/
  style.css                     ← 커스텀 CSS
```

---

## 6. 구현 기능 명세 (발표용 Phase 1 우선)

### [Phase 1 — 발표용 데모 필수]

#### 기능 1. 멀티턴 RFP 질의응답 (메인)

- `st.chat_message` + `st.chat_input` ChatGPT 스타일 UI
- `st.session_state.history`로 히스토리 관리
- ConversationManagerV2 사용
- 답변을 **답변** / **근거** / **출처** 3섹션으로 렌더링
- 스트리밍 출력 적용

#### 기능 2. 발주기관 필터

- 사이드바 `st.selectbox`로 기관 선택
- 현재 적용 필터 배지 표시
- "전체" 선택 시 필터 해제

#### 기능 8. 핵심 조건 요약 카드

- 사업명 감지 시 상단 `st.expander`로 자동 표시
- 항목: 사업명 / 발주기관 / 예산 / 계약기간 / 입찰방식

#### 기능 11. 오타/약어 자동 보정 피드백

- 보정 발생 시 `st.info("💡 '...' → '...'로 이해했습니다.")` 표시

#### 기능 13. 출처 하이라이팅

- **출처** 섹션 파일명에 색상 강조 + 다운로드 버튼 연결

#### 기능 14. 신뢰도 표시

- 규칙 기반: 출처 있음+수치 있음=🟢 / 출처만=🟡 / 없음=🔴
- 답변 우측 상단 배지

#### 기능 15. 대화 히스토리 내보내기

- 사이드바 "내보내기" 버튼
- Excel / JSON 형식 `st.download_button`

#### 기능 17. 원본 파일 다운로드

```python
# 출처 섹션에서 파일명 감지 → 다운로드 버튼
source_text = get_section(answer, "출처")
file_names  = re.findall(r"[\w\s\-\(\)\[\]]+\.(hwp|pdf)", source_text, re.IGNORECASE)
for fname in file_names:
    file_path = FILES_PATH / fname
    if file_path.exists():
        with open(file_path, "rb") as f:
            st.download_button(
                label     = f"📄 {fname}",
                data      = f.read(),
                file_name = fname,
                key       = f"dl_{fname}"
            )
```

### [Phase 2 — 여유 시 추가]

- **기능 3**: 사업 비교 분석 (retrieve() 2회 + 비교 프롬프트)
- **기능 4**: 입찰 체크리스트 자동 생성
- **기능 5**: 예산 분석 대시보드 (pages/1_대시보드.py)
- **기능 6**: 유사 사업 추천 (pages/2_사업검색.py)
- **기능 16**: 북마크 (pages/3_북마크.py)

---

## 7. 사이드바 구성

```
[🏢 입찰메이트]
──────────────────
📌 발주기관 필터
  [드롭다운: 전체 / 기관명 목록]

──────────────────
⭐ 즐겨찾기 (N개)

──────────────────
📥 내보내기
  [Excel] [JSON]

──────────────────
ℹ️ 시스템 상태
  Ollama: 🟢 연결됨
  ChromaDB: 51,366청크
  모델: gemma4:e4b
```

---

## 8. 세션 상태 관리

```python
if "messages"      not in st.session_state: st.session_state.messages      = []
if "history"       not in st.session_state: st.session_state.history       = []
if "bookmarks"     not in st.session_state: st.session_state.bookmarks     = []
if "agency_filter" not in st.session_state: st.session_state.agency_filter = "전체"
if "manager"       not in st.session_state:
    st.session_state.manager = ConversationManagerV2()
```

---

## 9. 주의사항 (절대 준수)

- `SYSTEM_PROMPT_V7E`, `TOOL_INSTRUCTION`, `TOOL_SCHEMAS` 수정 금지
- `ask()`, `retrieve()` 로직 수정 금지
- ChromaDB 재구축 금지 (기존 컬렉션 그대로 사용)
- `max_completion_tokens` 파라미터 사용 (max_tokens 아님)
- 발주기관 필터 적용 시 n_results 초과 방지 처리 필수:
  ```python
  available = sum(1 for c in all_chunks if c["metadata"].get("발주기관") == agency)
  safe_n = max(1, min(n_candidates, available))
  ```
- 모든 텍스트 비교는 `unicodedata.normalize("NFC", text)` 적용

---

## 10. 실행 방법

```bash
# 의존성 설치
pip install streamlit plotly openpyxl chromadb \
            sentence-transformers rank_bm25 rapidfuzz \
            FlagEmbedding openai ollama langchain-ollama

# Ollama 서버 (별도 터미널)
ollama serve

# 앱 실행
streamlit run app.py
# → http://localhost:8501
```

---

## 11. 구현 순서 (권장)

```
Step 1: core/functions.py 생성 (com_04.ipynb에서 핵심 함수 이식)
Step 2: app.py 기본 챗봇 (기능 1, 2, 11, 14, 17)
Step 3: 출처 하이라이팅 + 내보내기 (기능 13, 15)
Step 4: 요약 카드 (기능 8)
Step 5: pages/ 추가 (여유 시)
```
