# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

영수증 지출 관리 앱 — 사용자가 영수증 이미지/PDF를 업로드하면 Upstage Vision LLM이 자동으로 파싱하여 구조화된 지출 데이터로 변환하는 경량 웹 애플리케이션. 데이터베이스 미사용; 데이터는 `backend/data/expenses.json`에 저장되며, Vercel 배포 시 클라이언트 `localStorage`를 보조 저장소로 활용한다.

## 개발 명령어

### 백엔드

```bash
# 가상환경 생성 및 활성화
python -m venv venv
venv\Scripts\activate  # Windows

# 의존성 설치
pip install -r backend/requirements.txt

# 개발 서버 실행
uvicorn backend.main:app --reload

# 확인: http://localhost:8000/docs (Swagger UI)
```

### 프론트엔드

```bash
cd frontend
npm install
npm run dev       # Vite 개발 서버
npm run build     # 프로덕션 빌드
```

### 환경변수

- `UPSTAGE_API_KEY` — Upstage API 인증 키 (백엔드)
- `VITE_API_BASE_URL` — 백엔드 기본 URL (프론트엔드, `VITE_` 접두사로 빌드 시 주입)
- `DATA_FILE_PATH` — `expenses.json` 저장 경로 (백엔드, 기본값: `backend/data/expenses.json`)

로컬 개발 시 `.env.example`을 `.env`로 복사하여 값을 입력한다. Vercel 배포 시에는 Vercel 대시보드에서 환경변수를 등록한다.

## 아키텍처

### 요청 흐름

```
브라우저 (React + Vite + TailwindCSS)
    │  HTTP REST
    ▼
FastAPI 백엔드
    ├── POST /api/upload  →  LangChain Chain  →  ChatUpstage Vision LLM  →  expenses.json에 저장
    ├── GET  /api/expenses        →  expenses.json 읽기 (?from=&to= 날짜 필터 지원)
    ├── PUT  /api/expenses/{id}   →  expenses.json 레코드 수정
    ├── DELETE /api/expenses/{id} →  expenses.json 레코드 삭제
    └── GET  /api/summary         →  expenses.json 집계 통계 (?month=YYYY-MM)
```

### OCR 파이프라인 (backend/services/ocr_service.py)

1. 업로드된 파일 수신 (JPG/PNG/PDF, 최대 10MB)
2. PDF인 경우 이미지로 변환 (`pdf2image` → Pillow) 후 Base64 인코딩
3. JSON 형식만 응답하도록 시스템 프롬프트를 설정한 LangChain을 통해 `ChatUpstage` 호출
4. LangChain Output Parser로 응답 파싱 → 구조화된 지출 딕셔너리 생성
5. UUID v4 + ISO 8601 형식의 `created_at` 타임스탬프 부여
6. `storage_service.py`가 `backend/data/expenses.json`에 append 저장

### 백엔드 디렉토리 구조

```
backend/
├── main.py                  # FastAPI 앱 진입점, CORS 설정, 라우터 등록
├── routers/
│   ├── upload.py            # POST /api/upload
│   ├── expenses.py          # GET / PUT / DELETE /api/expenses
│   └── summary.py           # GET /api/summary
├── services/
│   ├── ocr_service.py       # LangChain + ChatUpstage 연동 로직
│   └── storage_service.py   # expenses.json 읽기/쓰기 헬퍼
├── data/
│   └── expenses.json        # append 방식의 JSON 배열
└── requirements.txt
```

### 프론트엔드 디렉토리 구조

```
frontend/src/
├── pages/
│   ├── Dashboard.jsx        # / — SummaryCard + FilterBar + ExpenseList
│   ├── UploadPage.jsx       # /upload — DropZone + ProgressBar + ParsePreview
│   └── ExpenseDetail.jsx    # /expense/:id — ReceiptImage + EditForm
├── components/
│   ├── DropZone.jsx         # 드래그 앤 드롭 / 클릭 업로드; 파일 선택 즉시 API 호출
│   ├── ParsePreview.jsx     # OCR 결과 인라인 편집 폼 (저장 / 취소 버튼 포함)
│   ├── ExpenseCard.jsx      # 가게명, 날짜, 금액, 카테고리 뱃지 카드
│   ├── SummaryCard.jsx      # 전체 지출 합계 + 이번달 지출
│   ├── FilterBar.jsx        # 날짜 범위 입력 + 조회 / 초기화 버튼
│   ├── Badge.jsx            # 카테고리 색상 뱃지
│   ├── Modal.jsx            # 삭제 확인 다이얼로그
│   └── Toast.jsx            # 슬라이드 업 알림 (성공 / 오류 / 정보)
└── api/
    └── axios.js             # VITE_API_BASE_URL로 설정된 Axios 인스턴스
```

## 데이터 스키마

`expenses.json`에 저장되는 지출 항목:

```json
{
  "id": "uuid-v4",
  "created_at": "2025-07-15T14:30:00Z",
  "store_name": "이마트 강남점",
  "receipt_date": "2025-07-15",
  "receipt_time": "13:25",
  "category": "식료품",
  "items": [
    { "name": "신라면 멀티팩", "quantity": 2, "unit_price": 4500, "total_price": 9000 }
  ],
  "subtotal": 10800,
  "discount": 500,
  "tax": 0,
  "total_amount": 10300,
  "payment_method": "신용카드",
  "raw_image_path": "uploads/receipt_20250715_001.jpg"
}
```

카테고리 값: `식료품`, `외식`, `교통`, `쇼핑`, `의료`, `기타`

## 배포 (Vercel)

- 프론트엔드: React/Vite 정적 빌드
- 백엔드: FastAPI를 Vercel Python 서버리스 함수로 배포 (`vercel.json` 설정 필요)
- **주요 제약**: Vercel 서버리스 컨테이너는 ephemeral(비지속) — 호출 간 `expenses.json`이 유지되지 않는다. MVP 해결책: 클라이언트 `localStorage`에 병행 저장. 장기적으로는 Vercel KV 또는 Supabase로 전환 권장.
- `pdf2image`는 Poppler가 필요하다. Vercel에서는 쓰기 가능한 경로가 `/tmp`뿐이므로 변환된 이미지를 `/tmp`에 저장해야 한다.

## 자주 발생하는 문제

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| CORS 오류 | `main.py`의 `allow_origins`에 Vercel 프론트엔드 URL 미포함 | FastAPI CORSMiddleware에 해당 URL 추가 |
| Vercel에서 PDF 변환 실패 | Poppler 기본 미설치 | Vercel 시스템 패키지 설정 활용 또는 클라이언트 측 변환 |
| 환경변수 미적용 | 프론트엔드는 `VITE_` 접두사 필요; Vercel 환경변수 추가 후 재빌드 필수 | `VITE_API_BASE_URL` 사용; 환경변수 등록 후 재배포 |
| Vercel에서 데이터 유실 | 서버리스 파일 시스템 비지속 | localStorage 병행 저장 (MVP) 또는 외부 스토리지 도입 |

## 바이브 코딩 3원칙

1. **코딩 전에 "완료 기준" 체크리스트 정의** — 각 Phase 시작 전 3~5개의 완료 기준을 먼저 작성한다
2. **새로운 기술은 조사 먼저, 구현 나중** — `langchain-upstage`, `pdf2image`, Vercel Python 서버리스 사용 전 `context7`으로 최신 API 사용법을 확인한다
3. **버그는 분석 먼저, 수정 나중** — 원인을 먼저 설명하고 수정 방향을 제안한 뒤 동의 후 수정한다; 에러 메시지만 보고 즉시 수정하지 않는다
