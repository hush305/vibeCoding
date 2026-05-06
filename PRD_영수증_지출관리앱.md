# PRD: 영수증 지출 관리 앱 (Receipt Expense Tracker)
### Product Requirements Document | Ver 1.3 | 2026-04-23

---

## 1. 문서 개요

| 항목 | 내용 |
|------|------|
| 문서 버전 | v1.3 |
| 작성일 | 2026-04-23 |
| 기반 문서 | 프로그램개요서_영수증_지출관리앱 v1.0 |
| 개발 기간 | 1일 단기 스프린트 |
| 배포 목표 | Vercel (프론트엔드 + 백엔드 서버리스) |

---

## 2. 제품 목적 및 배경

### 2.1 문제 정의

| 문제 | 설명 |
|------|------|
| 수동 입력 번거로움 | 사용자가 영수증을 보고 가계부에 직접 입력해야 하는 반복 작업 |
| 관리 불연속성 | 영수증을 분실하거나 입력을 미루면 지출 내역 파악이 어려움 |
| 진입 장벽 | 기존 가계부 앱은 기능이 많아 단순 지출 추적에 과도한 학습 비용 발생 |

### 2.2 솔루션

영수증(이미지/PDF)을 업로드하면 **Upstage Vision LLM**이 자동으로 내용을 파싱·구조화하여 지출 데이터를 생성하는 경량 웹 애플리케이션을 제공한다.

### 2.3 목표 지표 (Success Metrics)

| 지표 | 목표값 |
|------|--------|
| 영수증 파싱 성공률 | 한국어·영어 영수증 기준 80% 이상 |
| 업로드 → 파싱 완료 응답 시간 | 10초 이내 |
| E2E 흐름 동작 (업로드 → 목록 조회) | 1일 스프린트 내 완료 |

---

## 3. 타겟 사용자 (User Personas)

### Persona A — 직장인 지출 관리자
- **특성**: 영수증이 많지만 입력이 귀찮은 30대 직장인
- **목표**: 최소한의 노력으로 월별 소비 패턴 파악
- **Pain Point**: 복잡한 앱 설치 없이 빠르게 영수증 기록을 남기고 싶음

### Persona B — 사이드 프로젝트 개발자
- **특성**: AI/OCR 기술을 실제 서비스에 적용해 보고 싶은 개발자
- **목표**: LangChain + Upstage Vision LLM 연동 실습 및 포트폴리오 구성
- **Pain Point**: 빠르게 동작하는 MVP를 만들고 싶음

---

## 4. 범위 정의 (Scope)

### 4.1 In Scope (1일 스프린트 내 포함)

| ID | 기능 | 우선순위 |
|----|------|---------|
| M-01 | 영수증 이미지 업로드 (JPG, PNG, PDF) | Must Have |
| M-02 | Upstage Vision LLM 기반 OCR 자동 파싱 | Must Have |
| M-03 | 구조화 JSON 추출 (가게명, 날짜, 품목, 금액) | Must Have |
| M-04 | 지출 내역 카드형 목록 조회 | Must Have |
| M-05 | expenses.json 파일 누적 저장 | Must Have |
| S-01 | 총 지출 합계 대시보드 | Should Have |
| S-02 | 날짜 범위 필터링 | Should Have |
| S-03 | 지출 항목 삭제 | Should Have |
| S-04 | OCR 파싱 결과 수정 후 저장 | Should Have |

### 4.2 Out of Scope (1차 제외)

- 사용자 인증/로그인
- 다국어 영수증 지원 (한국어·영어 외)
- 데이터베이스 연동 (Supabase, PostgreSQL 등)
- 다중 사용자 동시 접속 지원
- 모바일 네이티브 앱
- 카테고리 자동 분류 학습 (ML 학습 루프)

---

## 5. 기능 요구사항 (Functional Requirements)

### FR-01: 영수증 파일 업로드

**설명**: 사용자가 영수증 이미지 또는 PDF를 웹 UI에서 업로드할 수 있다.

| 항목 | 내용 |
|------|------|
| 지원 형식 | JPG, PNG, PDF |
| 최대 파일 크기 | 10MB |
| 입력 방식 | 드래그 앤 드롭 또는 파일 선택 버튼 |
| 업로드 중 피드백 | 진행률 표시바(ProgressBar) 노출 |

**수락 기준 (Acceptance Criteria)**:
- [ ] JPG, PNG, PDF 파일이 정상 업로드된다.
- [ ] 10MB 초과 파일은 오류 메시지를 표시하고 업로드를 차단한다.
- [ ] 지원하지 않는 형식(예: .gif, .docx) 업로드 시 오류 메시지를 표시한다.
- [ ] 업로드 진행 중 로딩 상태가 UI에 표시된다.

---

### FR-02: OCR 자동 파싱

**설명**: 업로드된 파일을 Upstage Vision LLM으로 분석하여 영수증 내용을 구조화된 JSON으로 반환한다.

| 항목 | 내용 |
|------|------|
| LLM 모델 | Upstage document-digitization-vision |
| 입력 | Base64 인코딩된 이미지 |
| 출력 | 구조화 JSON (아래 스키마 참조) |
| 오케스트레이션 | LangChain Chain + OutputParser |

**추출 필드**:

| 필드 | 필수 여부 | 설명 |
|------|-----------|------|
| store_name | 필수 | 가게(상호) 이름 |
| receipt_date | 필수 | 영수증 날짜 (YYYY-MM-DD) |
| receipt_time | 선택 | 영수증 시각 (HH:MM) |
| category | 선택 | 지출 카테고리 (예: 식료품, 외식) |
| items[] | 필수 | 품목 목록 (name, quantity, unit_price, total_price) |
| total_amount | 필수 | 최종 결제 금액 |
| payment_method | 선택 | 결제 수단 (예: 신용카드, 현금) |

**수락 기준**:
- [ ] 한국어 영수증에서 가게명, 날짜, 합계 금액이 정확히 추출된다.
- [ ] 영어 영수증에서도 동일하게 동작한다.
- [ ] LLM 파싱 실패 시 사용자에게 오류 메시지를 표시하고 재시도를 안내한다.
- [ ] 응답은 10초 이내에 반환된다.

---

### FR-03: 파싱 결과 미리보기 및 저장

**설명**: OCR 파싱 결과를 사용자가 확인하고, 필요 시 수정 후 저장할 수 있다.

**수락 기준**:
- [ ] 파싱 결과가 업로드 페이지 하단에 즉시 표시된다.
- [ ] 사용자가 각 필드를 직접 수정할 수 있다.
- [ ] "저장" 버튼 클릭 시 expenses.json에 append 저장된다.
- [ ] 저장 완료 후 성공 Toast 알림이 표시된다.

---

### FR-04: 지출 내역 목록 조회

**설명**: 저장된 지출 내역을 카드 형태로 목록 조회한다.

**수락 기준**:
- [ ] 메인 대시보드에 저장된 모든 지출 내역이 카드 형태로 표시된다.
- [ ] 각 카드에는 가게명, 날짜, 총 금액, 카테고리 뱃지가 표시된다.
- [ ] 날짜 범위 필터 적용 시 해당 기간의 내역만 필터링된다.
- [ ] 내역이 없을 경우 빈 상태(empty state) 안내 문구가 표시된다.

---

### FR-05: 지출 합계 대시보드

**설명**: 조회된 지출 내역의 합산 통계를 요약하여 표시한다.

**수락 기준**:
- [ ] 총 지출 금액이 대시보드 상단에 표시된다.
- [ ] 이번 달 지출 금액이 별도로 표시된다.
- [ ] 카테고리별 합계가 요약 표시된다.

---

### FR-06: 지출 항목 삭제

**설명**: 잘못 등록된 지출 항목을 삭제할 수 있다.

**수락 기준**:
- [ ] 각 지출 카드 또는 상세 화면에서 삭제 버튼이 제공된다.
- [ ] 삭제 전 확인 다이얼로그(Modal)가 표시된다.
- [ ] 삭제 확인 시 expenses.json에서 해당 항목이 제거되고 목록이 즉시 갱신된다.
- [ ] 삭제 완료 후 Toast 알림이 표시된다.

---

## 6. 비기능 요구사항 (Non-Functional Requirements)

### 6.1 성능

| 항목 | 요구사항 |
|------|---------|
| OCR 파싱 응답 시간 | 10초 이내 (Upstage API 응답 포함) |
| 목록 조회 응답 시간 | 1초 이내 (JSON 파일 읽기) |
| 동시 사용자 | 1인 기준 (개인 프로젝트 MVP) |

### 6.2 보안

| 항목 | 요구사항 |
|------|---------|
| API Key 관리 | Vercel 환경변수에만 저장, 소스코드 노출 금지 |
| API 엔드포인트 | URL 비공개로 최소 보안 유지 |
| 파일 업로드 검증 | 파일 형식 및 크기 서버 측 재검증 필수 |
| 인증 | 1차 범위 제외 (API URL 비공개로 대체) |

### 6.3 가용성 및 데이터 영속성

| 방안 | 설명 | 난이도 | 권장 |
|------|------|--------|------|
| localStorage 병행 저장 | 클라이언트에서 데이터 영속성 유지 | ⭐ 쉬움 | MVP 1순위 |
| Railway / Render 배포 | 일반 서버에서 파일 시스템 유지 | ⭐⭐ 보통 | 안정성 필요 시 |
| Vercel KV (Redis) | Vercel 내장 키-값 저장소 | ⭐⭐ 보통 | Vercel 유지 시 |
| Supabase 무료 플랜 | PostgreSQL DB로 영구 전환 | ⭐⭐⭐ 어려움 | 장기 운영 시 |

> **MVP 기본 채택**: localStorage 병행 저장 (Vercel 서버리스 파일 시스템 비지속 문제 대응)

### 6.4 유지보수성

- 백엔드와 프론트엔드를 독립 디렉토리(`backend/`, `frontend/`)로 분리
- 환경변수는 `.env` 파일 또는 Vercel 환경변수로 중앙 관리
- LangChain Chain 로직은 `services/` 레이어로 분리하여 교체 용이하게 구성

---

## 7. 데이터 구조 (Data Schema)

### 7.1 지출 항목 JSON 스키마

```json
{
  "id": "uuid-v4-string",
  "created_at": "2025-07-15T14:30:00Z",
  "store_name": "이마트 강남점",
  "receipt_date": "2025-07-15",
  "receipt_time": "13:25",
  "category": "식료품",
  "items": [
    {
      "name": "신라면 멀티팩",
      "quantity": 2,
      "unit_price": 4500,
      "total_price": 9000
    },
    {
      "name": "바나나우유",
      "quantity": 1,
      "unit_price": 1800,
      "total_price": 1800
    }
  ],
  "subtotal": 10800,
  "discount": 500,
  "tax": 0,
  "total_amount": 10300,
  "payment_method": "신용카드",
  "raw_image_path": "uploads/receipt_20250715_001.jpg"
}
```

### 7.2 저장 구조

- **파일 위치**: `backend/data/expenses.json`
- **저장 방식**: JSON 배열에 append (누적 저장)
- **ID 생성**: UUID v4 (서버 측 생성)
- **타임스탬프**: ISO 8601 형식 (UTC 기준)

---

## 8. API 명세 (API Specification)

### POST /api/upload
| 항목 | 내용 |
|------|------|
| 설명 | 영수증 파일 업로드 및 OCR 파싱 |
| 요청 형식 | `multipart/form-data` |
| 요청 파라미터 | `file`: 업로드 파일 |
| 성공 응답 (200) | 파싱된 지출 JSON 객체 |
| 실패 응답 (400) | 파일 형식/크기 오류 메시지 |
| 실패 응답 (500) | OCR 파싱 실패 메시지 |

### GET /api/expenses
| 항목 | 내용 |
|------|------|
| 설명 | 전체 지출 내역 조회 |
| 쿼리 파라미터 | `from`: 시작일 (YYYY-MM-DD), `to`: 종료일 (YYYY-MM-DD) |
| 성공 응답 (200) | 지출 항목 배열 `[]` |

### DELETE /api/expenses/{id}
| 항목 | 내용 |
|------|------|
| 설명 | 특정 지출 항목 삭제 |
| Path 파라미터 | `id`: UUID |
| 성공 응답 (200) | 삭제 성공 메시지 |
| 실패 응답 (404) | 항목 없음 메시지 |

### PUT /api/expenses/{id}
| 항목 | 내용 |
|------|------|
| 설명 | 특정 지출 항목 수정 |
| Path 파라미터 | `id`: UUID |
| 요청 Body | 수정할 JSON 필드 |
| 성공 응답 (200) | 수정된 지출 객체 |

### GET /api/summary
| 항목 | 내용 |
|------|------|
| 설명 | 지출 합계 통계 조회 |
| 쿼리 파라미터 | `month`: YYYY-MM (선택) |
| 성공 응답 (200) | 총합계, 이번달 합계, 카테고리별 통계 |

---

## 9. 화면 설계 (UI Specification)

### 9.1 페이지 목록

| 페이지 | 경로 | 핵심 컴포넌트 |
|--------|------|--------------|
| 메인 대시보드 | `/` | SummaryCard, FilterBar, ExpenseList, ExpenseCard |
| 업로드 | `/upload` | DropZone, ProgressBar, ParsePreview |
| 지출 상세/수정 | `/expense/:id` | ReceiptImage, EditForm, 삭제 버튼 |

### 9.2 주요 컴포넌트 명세

#### DropZone
- 드래그 앤 드롭 및 클릭 업로드 지원
- 지원 형식 안내 텍스트 표시 (JPG, PNG, PDF / 최대 10MB)
- 파일 선택 후 즉시 업로드 API 호출

#### ParsePreview
- OCR 파싱 결과를 인라인 편집 가능한 폼으로 표시
- 각 필드: 가게명, 날짜, 카테고리, 품목 목록, 합계
- "저장" / "취소" 버튼 제공

#### ExpenseCard
- 표시 항목: 가게명, 날짜, 총 금액, 카테고리 뱃지
- 카드 클릭 시 상세 페이지(`/expense/:id`)로 이동

#### SummaryCard
- 총 지출 금액 (전체 기간)
- 이번 달 지출 금액
- 카테고리별 지출 요약

#### FilterBar
- 시작일 / 종료일 날짜 입력
- "조회" 버튼으로 필터 적용
- "초기화" 버튼으로 전체 목록 복원

### 9.3 공통 컴포넌트

| 컴포넌트 | 역할 |
|----------|------|
| Badge | 카테고리 뱃지 (색상 코딩) |
| Modal | 삭제 확인 다이얼로그 |
| Toast | 저장/삭제/오류 알림 (3초 자동 소멸) |
| ProgressBar | OCR 처리 진행 상태 표시 |

---

## 10. 화면 디자인 & 스타일 가이드 (Design & Style Guide)

### 10.1 디자인 원칙

| 원칙 | 설명 |
|------|------|
| **간결함 (Simplicity)** | 불필요한 요소 없이 핵심 정보만 표시. 영수증 등록과 조회가 즉시 직관적으로 가능해야 함 |
| **신뢰감 (Trust)** | 숫자·금액·날짜가 명확히 구분되어 데이터를 신뢰할 수 있는 느낌 |
| **빠른 피드백 (Responsiveness)** | 업로드·저장·삭제 모든 동작에 즉각적인 시각적 피드백 제공 |
| **모바일 우선 (Mobile-First)** | 스마트폰에서 영수증을 촬영 후 바로 업로드하는 시나리오 고려 |

---

### 10.2 컬러 팔레트

#### 주요 색상 (Primary)

| 토큰 | Tailwind 클래스 | HEX | 용도 |
|------|----------------|-----|------|
| Primary | `indigo-600` | `#4F46E5` | CTA 버튼, 링크, 포커스 링 |
| Primary Hover | `indigo-700` | `#4338CA` | 버튼 hover 상태 |
| Primary Light | `indigo-50` | `#EEF2FF` | 선택된 항목 배경, 뱃지 배경 |

#### 중립 색상 (Neutral)

| 토큰 | Tailwind 클래스 | HEX | 용도 |
|------|----------------|-----|------|
| Text Primary | `gray-900` | `#111827` | 본문 주요 텍스트 |
| Text Secondary | `gray-500` | `#6B7280` | 보조 텍스트, 날짜, 힌트 |
| Border | `gray-200` | `#E5E7EB` | 카드 테두리, 구분선 |
| Background | `gray-50` | `#F9FAFB` | 페이지 배경 |
| Surface | `white` | `#FFFFFF` | 카드, 모달 배경 |

#### 의미 색상 (Semantic)

| 상태 | Tailwind 클래스 | HEX | 용도 |
|------|----------------|-----|------|
| Success | `green-500` | `#22C55E` | 저장 완료 Toast, 성공 아이콘 |
| Warning | `amber-500` | `#F59E0B` | 파싱 불확실 필드 강조 |
| Error | `red-500` | `#EF4444` | 오류 메시지, 삭제 버튼 |
| Info | `blue-500` | `#3B82F6` | OCR 처리 중 상태 표시 |

#### 카테고리 뱃지 색상

| 카테고리 | 뱃지 배경 | 텍스트 색 |
|----------|-----------|----------|
| 식료품 | `green-100` | `green-700` |
| 외식 | `orange-100` | `orange-700` |
| 교통 | `blue-100` | `blue-700` |
| 쇼핑 | `purple-100` | `purple-700` |
| 의료 | `red-100` | `red-700` |
| 기타 | `gray-100` | `gray-700` |

---

### 10.3 타이포그래피 (Typography)

#### 폰트 패밀리

```css
/* 한국어 최적화 */
font-family: 'Pretendard', 'Noto Sans KR', -apple-system, BlinkMacSystemFont, sans-serif;
```

> Pretendard CDN 또는 로컬 폰트 적용. Fallback으로 Noto Sans KR 사용.

#### 텍스트 스케일

| 역할 | Tailwind 클래스 | 크기 | 굵기 | 용도 |
|------|----------------|------|------|------|
| Page Title | `text-2xl font-bold` | 24px / 700 | Bold | 페이지 제목 |
| Section Title | `text-lg font-semibold` | 18px / 600 | Semibold | 섹션 헤더 |
| Card Title | `text-base font-semibold` | 16px / 600 | Semibold | 가게명, 카드 주요 텍스트 |
| Body | `text-sm font-normal` | 14px / 400 | Regular | 일반 본문 |
| Caption | `text-xs font-normal` | 12px / 400 | Regular | 날짜, 보조 정보 |
| Amount | `text-xl font-bold` | 20px / 700 | Bold | 금액 강조 표시 |

---

### 10.4 간격 및 크기 시스템 (Spacing)

TailwindCSS의 4px 기반 간격 시스템을 따른다.

| 토큰 | Tailwind | px | 용도 |
|------|----------|----|------|
| xs | `p-1` | 4px | 아이콘 내부 여백 |
| sm | `p-2` | 8px | 뱃지, 작은 버튼 패딩 |
| md | `p-4` | 16px | 카드 내부 패딩 |
| lg | `p-6` | 24px | 페이지 섹션 여백 |
| xl | `p-8` | 32px | 페이지 최상위 컨테이너 |
| 2xl | `p-12` | 48px | 히어로 섹션, DropZone |

---

### 10.5 레이아웃 & 그리드

#### 전체 레이아웃 구조

```
┌─────────────────────────────────────────┐
│  Header (sticky, h-16, bg-white, shadow) │
├─────────────────────────────────────────┤
│                                         │
│   max-w-4xl mx-auto px-4 py-6          │
│   (콘텐츠 최대 너비 896px, 중앙 정렬)    │
│                                         │
│   [페이지 콘텐츠 영역]                    │
│                                         │
├─────────────────────────────────────────┤
│  Footer (bg-gray-50, py-4)              │
└─────────────────────────────────────────┘
```

#### 반응형 브레이크포인트

| 브레이크포인트 | Tailwind | 너비 | 레이아웃 변화 |
|--------------|---------|------|--------------|
| Mobile (기본) | - | < 640px | 단일 컬럼, 풀 너비 카드 |
| Tablet | `sm:` | ≥ 640px | 카드 2열 그리드 |
| Desktop | `lg:` | ≥ 1024px | 카드 3열 그리드, 사이드 필터 |

#### ExpenseList 그리드

```
Mobile:   grid-cols-1
Tablet:   sm:grid-cols-2
Desktop:  lg:grid-cols-3
Gap:      gap-4
```

---

### 10.6 컴포넌트 스타일 명세

#### 버튼 (Button)

| 종류 | Tailwind 클래스 | 용도 |
|------|----------------|------|
| Primary | `bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-2 px-4 rounded-lg transition-colors` | 저장, 조회 CTA |
| Secondary | `bg-white hover:bg-gray-50 text-gray-700 font-semibold py-2 px-4 rounded-lg border border-gray-300 transition-colors` | 취소, 초기화 |
| Danger | `bg-red-500 hover:bg-red-600 text-white font-semibold py-2 px-4 rounded-lg transition-colors` | 삭제 확인 |
| Ghost | `text-indigo-600 hover:text-indigo-700 font-medium underline` | 텍스트 링크형 버튼 |

#### 카드 (Card)

```
bg-white
rounded-xl
border border-gray-200
shadow-sm hover:shadow-md
transition-shadow duration-200
p-4
cursor-pointer
```

#### 입력 필드 (Input)

```
w-full
px-3 py-2
border border-gray-300
rounded-lg
text-sm text-gray-900
placeholder-gray-400
focus:outline-none
focus:ring-2 focus:ring-indigo-500
focus:border-transparent
transition
```

#### DropZone

```
border-2 border-dashed border-gray-300
hover:border-indigo-400
rounded-xl
p-12
text-center
bg-gray-50 hover:bg-indigo-50
transition-colors duration-200
cursor-pointer
```

- 드래그 오버 시: `border-indigo-500 bg-indigo-50`
- 파일 선택 완료 시: `border-green-400 bg-green-50`

#### 뱃지 (Badge)

```
inline-flex items-center
px-2.5 py-0.5
rounded-full
text-xs font-medium
[카테고리별 색상 클래스 적용]
```

#### Toast 알림

```
fixed bottom-4 right-4
flex items-center gap-2
px-4 py-3
rounded-lg shadow-lg
text-sm font-medium text-white
animate-slide-up                   /* 아래에서 위로 슬라이드 인 */
z-50
```
- Success: `bg-green-500`
- Error: `bg-red-500`
- Info: `bg-blue-500`

#### Modal (삭제 확인)

```
/* 오버레이 */
fixed inset-0 bg-black/50 z-40 flex items-center justify-center

/* 다이얼로그 패널 */
bg-white rounded-2xl shadow-xl p-6 w-full max-w-sm mx-4
animate-scale-in
```

#### ProgressBar

```
w-full h-2
bg-gray-200 rounded-full overflow-hidden

/* 진행 바 */
h-full bg-indigo-500 rounded-full
transition-[width] duration-300 ease-out
```

---

### 10.7 화면별 레이아웃 와이어프레임

#### 메인 대시보드 (`/`)

```
┌────────────────────────────────────────────┐
│  [로고] Receipt Tracker   [+ 영수증 추가]   │  ← Header
├────────────────────────────────────────────┤
│                                            │
│  ┌──────────────┐  ┌──────────────┐       │
│  │  총 지출      │  │  이번달 지출  │       │  ← SummaryCard (2열)
│  │  ₩ 128,500   │  │  ₩ 45,300   │       │
│  └──────────────┘  └──────────────┘       │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  [시작일 ____] ~ [종료일 ____] [조회] │  │  ← FilterBar
│  └──────────────────────────────────────┘  │
│                                            │
│  지출 내역 (12건)                           │
│                                            │
│  ┌──────────────┐ ┌──────────────┐        │
│  │ [식료품]      │ │ [외식]        │        │
│  │ 이마트 강남점 │ │ 맥도날드      │        │  ← ExpenseCard (그리드)
│  │ 2025-07-15   │ │ 2025-07-14   │        │
│  │    ₩ 10,300  │ │     ₩ 8,900  │        │
│  └──────────────┘ └──────────────┘        │
│         ... (더 보기)                       │
│                                            │
└────────────────────────────────────────────┘
```

---

#### 업로드 페이지 (`/upload`)

```
┌────────────────────────────────────────────┐
│  [← 뒤로]  영수증 업로드                    │  ← Header
├────────────────────────────────────────────┤
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │                                      │  │
│  │    [↑ 아이콘]                         │  │
│  │    영수증을 드래그하거나               │  │  ← DropZone
│  │    클릭하여 업로드하세요               │  │
│  │    JPG, PNG, PDF · 최대 10MB         │  │
│  │                                      │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━        │  ← ProgressBar (처리 중)
│  OCR 분석 중... (indigo)                   │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  파싱 결과 미리보기                   │  │
│  │  ─────────────────────────────────  │  │
│  │  가게명   [ 이마트 강남점          ]  │  │
│  │  날짜     [ 2025-07-15            ]  │  │  ← ParsePreview
│  │  카테고리 [ 식료품               ▼]  │  │
│  │  결제     [ 신용카드              ]  │  │
│  │                                      │  │
│  │  품목 목록                           │  │
│  │  신라면 멀티팩  x2  ₩9,000          │  │
│  │  바나나우유    x1  ₩1,800           │  │
│  │  ─────────────────────────────────  │  │
│  │  합계                    ₩10,300    │  │
│  │                                      │  │
│  │  [취소]                   [저장하기]  │  │
│  └──────────────────────────────────────┘  │
│                                            │
└────────────────────────────────────────────┘
```

---

#### 지출 상세 / 수정 (`/expense/:id`)

```
┌────────────────────────────────────────────┐
│  [← 뒤로]  지출 상세                        │  ← Header
├────────────────────────────────────────────┤
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  [영수증 원본 이미지 미리보기]         │  │  ← ReceiptImage
│  │  (비율 유지, 최대 높이 300px)         │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  [식료품]  이마트 강남점              │  │
│  │  ─────────────────────────────────  │  │
│  │  가게명   [ 이마트 강남점          ]  │  │
│  │  날짜     [ 2025-07-15            ]  │  │  ← EditForm
│  │  카테고리 [ 식료품               ▼]  │  │
│  │  결제     [ 신용카드              ]  │  │
│  │                                      │  │
│  │  품목 목록                           │  │
│  │  신라면 멀티팩  x2  ₩9,000          │  │
│  │  바나나우유    x1  ₩1,800           │  │
│  │  ─────────────────────────────────  │  │
│  │  합계                    ₩10,300    │  │
│  │                                      │  │
│  │  [삭제]                   [수정 저장]  │  │
│  └──────────────────────────────────────┘  │
│                                            │
└────────────────────────────────────────────┘
```

---

### 10.8 인터랙션 & 애니메이션

| 요소 | 동작 | Tailwind / CSS |
|------|------|---------------|
| 카드 hover | 그림자 강조 | `hover:shadow-md transition-shadow duration-200` |
| 버튼 hover | 색상 어둡게 | `transition-colors duration-150` |
| Toast 등장 | 아래에서 위로 슬라이드 | `animate-[slide-up_0.3s_ease-out]` |
| Modal 등장 | 중앙에서 스케일 인 | `animate-[scale-in_0.2s_ease-out]` |
| ProgressBar | 너비 부드럽게 증가 | `transition-[width] duration-300 ease-out` |
| DropZone drag | 테두리·배경색 변경 | `border-indigo-500 bg-indigo-50` |
| 페이지 전환 | 페이드 인 | `animate-[fade-in_0.2s_ease]` |

**커스텀 애니메이션 (tailwind.config.js)**:

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      keyframes: {
        'slide-up': {
          '0%': { transform: 'translateY(16px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
        'scale-in': {
          '0%': { transform: 'scale(0.95)', opacity: '0' },
          '100%': { transform: 'scale(1)', opacity: '1' },
        },
        'fade-in': {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
      },
    },
  },
}
```

---

### 10.9 상태별 UI 명세

#### 로딩 상태 (OCR 처리 중)

- DropZone: 반투명 오버레이 + 스피너
- 업로드 버튼: 비활성화 (`disabled`, `opacity-50 cursor-not-allowed`)
- ProgressBar: 인디고 진행 바 + "OCR 분석 중..." 텍스트

#### 빈 상태 (데이터 없음)

```
┌──────────────────────────────────────┐
│                                      │
│       [영수증 아이콘 (회색)]           │
│                                      │
│       아직 등록된 지출 내역이 없습니다  │
│       영수증을 업로드해 보세요         │
│                                      │
│          [+ 첫 영수증 등록하기]        │
│                                      │
└──────────────────────────────────────┘
```

#### 오류 상태 (OCR 실패)

- 빨간 테두리 배너: `bg-red-50 border border-red-200 text-red-700 rounded-lg p-4`
- 메시지: "영수증 파싱에 실패했습니다. 이미지를 확인하고 다시 시도해 주세요."
- [다시 시도] 버튼 제공

#### 파싱 불확실 필드 강조

- LLM이 추출에 불확실한 필드: `border-amber-400 bg-amber-50` 입력 박스 스타일
- 필드 하단: `text-amber-600 text-xs "OCR 결과를 확인해 주세요"` 힌트 표시

---

### 10.10 Header 및 네비게이션

```
┌──────────────────────────────────────────────────────┐
│  [영수증 아이콘] Receipt Tracker          [+ 업로드]  │
└──────────────────────────────────────────────────────┘
```

| 항목 | 스타일 |
|------|--------|
| 배경 | `bg-white border-b border-gray-200 sticky top-0 z-30` |
| 높이 | `h-16` |
| 로고 텍스트 | `text-lg font-bold text-indigo-600` |
| 업로드 버튼 | Primary 버튼 스타일 (`sm` 사이즈) |
| 그림자 | `shadow-sm` |

---

## 11. 기술 스택 및 의존성 (Tech Stack)

| 구분 | 기술 | 버전 |
|------|------|------|
| 프론트엔드 | ReactJS | v18+ |
| 빌드 도구 | Vite | v5+ |
| 스타일링 | TailwindCSS | v3+ |
| HTTP 클라이언트 | Axios | v1+ |
| 백엔드 | Python FastAPI | v0.111+ |
| LLM 오케스트레이션 | LangChain | v0.2+ |
| OCR LLM | Upstage document-digitization-vision | - |
| 이미지 처리 | Pillow / pdf2image | - |
| 데이터 저장 | JSON 파일 | DB 미사용 |
| 배포 | Vercel | - |
| 버전 관리 | GitHub | main 브랜치 |

---

## 11. 환경변수 명세

| 변수명 | 설명 | 적용 위치 |
|--------|------|----------|
| `UPSTAGE_API_KEY` | Upstage API 인증 키 | Vercel 환경변수 (백엔드) |
| `VITE_API_BASE_URL` | 백엔드 API 기본 URL | Vercel 환경변수 (프론트 빌드 시 주입) |
| `DATA_FILE_PATH` | expenses.json 저장 경로 | Vercel 환경변수 (백엔드) |

---

## 12. 프로젝트 디렉토리 구조

```
receipt-tracker/
├── frontend/
│     ├── src/
│     │     ├── pages/
│     │     │     ├── Dashboard.jsx
│     │     │     ├── UploadPage.jsx
│     │     │     └── ExpenseDetail.jsx
│     │     ├── components/
│     │     │     ├── DropZone.jsx
│     │     │     ├── ParsePreview.jsx
│     │     │     ├── ExpenseCard.jsx
│     │     │     ├── SummaryCard.jsx
│     │     │     ├── FilterBar.jsx
│     │     │     ├── Badge.jsx
│     │     │     ├── Modal.jsx
│     │     │     └── Toast.jsx
│     │     └── api/
│     │           └── axios.js        # Axios 인스턴스 및 API 함수
│     ├── package.json
│     └── vite.config.js
├── backend/
│     ├── main.py                     # FastAPI 앱 진입점
│     ├── routers/
│     │     ├── upload.py             # POST /api/upload
│     │     ├── expenses.py           # GET, DELETE, PUT /api/expenses
│     │     └── summary.py            # GET /api/summary
│     ├── services/
│     │     ├── ocr_service.py        # LangChain + Upstage 연동 로직
│     │     └── storage_service.py    # expenses.json 읽기/쓰기
│     ├── data/
│     │     └── expenses.json
│     └── requirements.txt
├── vercel.json
└── README.md
```

---


---

## 13. 바이브 코딩 3원칙 적용

> 바이브 코딩 시 아래 세 가지 원칙을 항상 염두에 두고 진행한다.

### 원칙 1 — "뭘 만들면 완료인지" 체크리스트를 미리 적어라

> AI도 사람도 "끝"의 기준이 명확해야 헤매지 않는다.

각 Phase 시작 전, Claude Code에게 아래 형식으로 완료 기준을 먼저 요청한다.

```
> 이번에 [기능명]을 구현할 거야.
> 코드 작성 전에 "이게 완성됐다"고 볼 수 있는 체크리스트를 3~5개만 먼저 알려줘.
```

#### 전체 스프린트 완료 체크리스트 (Master Checklist)

**Must Have — 이것만 되면 배포 가능**
- [ ] JPG/PNG/PDF 영수증 업로드 → OCR 파싱 → JSON 반환 정상 동작
- [ ] 파싱 결과가 ParsePreview에 표시되고 수정 후 저장 가능
- [ ] 대시보드에서 저장된 지출 내역 카드 목록 조회 가능
- [ ] 10MB 초과 / 미지원 형식 업로드 시 오류 메시지 표시
- [ ] Vercel 배포 URL에서 E2E 전체 흐름 동작 확인

**Should Have — 완성도를 높이는 기능**
- [ ] 날짜 범위 필터링 동작
- [ ] 지출 항목 삭제 (Modal 확인 포함)
- [ ] 총 지출 / 이번달 지출 SummaryCard 표시
- [ ] 상세 페이지에서 수정 저장 동작

---

### 원칙 2 — 새로운 기술을 붙일 땐 "조사 먼저, 구현 나중"

> 5분의 조사가 1시간의 디버깅을 아껴준다.

본 프로젝트에서 처음 사용하는 기술은 아래와 같다. 각 기술을 사용하기 **전에** Claude Code에 먼저 조사를 요청한다.

| 기술 | 조사 시점 | 조사 프롬프트 예시 |
|------|-----------|-----------------|
| `langchain-upstage` | Phase 2 시작 전 | `"langchain-upstage의 ChatUpstage로 이미지를 Vision LLM에 전달하는 최신 방법을 use context7로 찾아줘"` |
| `pdf2image` | Phase 2 시작 전 | `"pdf2image로 PDF를 이미지로 변환할 때 Vercel 서버리스 환경에서 주의할 점이 있어?"` |
| `Vercel Python 서버리스` | Phase 8 시작 전 | `"FastAPI를 Vercel 서버리스로 배포할 때 Mangum 없이 동작하는지, 현재 권장 방법을 찾아줘"` |
| `Vite 환경변수 주입` | Phase 4 시작 전 | `"Vite에서 VITE_ 접두사 환경변수를 프로덕션 빌드에 주입하는 방법을 확인해줘"` |

**조사 후 확인할 것:**
- [ ] 공식 문서 또는 최신 예제 코드에서 동작 확인
- [ ] 현재 사용 버전(`requirements.txt`, `package.json`)과 호환 여부 확인
- [ ] Vercel 서버리스 환경에서의 제약사항 확인

---

### 원칙 3 — 버그가 나면 "분석 먼저, 수정 나중"

> "로그 보고 고쳐"보다 "원인부터 알려줘"가 근본 해결로 이어진다.

버그 발생 시 Claude Code에 아래 순서로 요청한다.

```
# ❌ 나쁜 방식
> 이 에러 고쳐줘: [에러 메시지]

# ✅ 좋은 방식
> 아래 에러가 발생했어. 수정하기 전에 원인부터 분석해줘.
> 어떤 흐름에서 왜 발생하는지 설명하고, 수정 방향을 제안해줘.
> 동의하면 수정을 진행할게.
> [에러 메시지 + 관련 코드]
```

#### 본 프로젝트 예상 버그 유형별 분석 체크리스트

| 버그 유형 | 분석 시 확인할 것 |
|-----------|-----------------|
| OCR 파싱 실패 (500) | LLM 응답이 JSON 형식인지 / 프롬프트 지시가 명확한지 / API Key 유효한지 |
| CORS 오류 | FastAPI `allow_origins` 설정 / Vercel 라우팅 경로 일치 여부 |
| PDF 변환 실패 | Vercel 환경의 Poppler 설치 여부 / `/tmp` 경로 사용 여부 |
| 데이터 손실 (Vercel) | 서버리스 컨테이너 재시작 여부 / localStorage 병행 저장 동작 여부 |
| 환경변수 미적용 | `VITE_` 접두사 확인 / Vercel 환경변수 등록 여부 / 재배포 여부 |

---

## 14. 개발 일정 및 완료 기준

> **전체 개발 타임라인 (1일 스프린트 기준)**
>
> ```
> [Phase 1] 환경 설정          ── 0.5h
> [Phase 2] 백엔드 핵심 API    ── 2.5h
> [Phase 3] 백엔드 부가 API    ── 1.0h
> [Phase 4] 프론트 환경 설정   ── 0.5h
> [Phase 5] 업로드 화면        ── 1.5h
> [Phase 6] 대시보드 화면      ── 1.5h
> [Phase 7] 상세/수정 화면     ── 1.0h  (Should Have)
> [Phase 8] 배포 & E2E 검증   ── 1.0h
>                              ────────
>                              총 9.5h
> ```

---

### Phase 1 — 프로젝트 환경 설정 (예상 0.5h)

#### 작업 목록

| # | 작업 | 산출물 | 우선도 |
|---|------|--------|--------|
| 1-1 | GitHub 레포지토리 생성 및 `.gitignore` 설정 | `receipt-tracker` 레포 | Must |
| 1-2 | 프로젝트 디렉토리 구조 생성 | `backend/`, `frontend/` 폴더 | Must |
| 1-3 | `.env` 파일 생성 및 `UPSTAGE_API_KEY` 등록 | `.env`, `.env.example` | Must |
| 1-4 | Python 가상환경 생성 및 패키지 설치 | `venv/`, `requirements.txt` | Must |

#### requirements.txt

```txt
fastapi==0.111.0
uvicorn[standard]==0.29.0
python-multipart==0.0.9
langchain==0.2.0
langchain-upstage==0.1.0
pillow==10.3.0
pdf2image==1.17.0
python-dotenv==1.0.1
```

#### 완료 기준
- [ ] `uvicorn backend.main:app --reload` 실행 시 FastAPI 서버가 정상 기동된다
- [ ] `http://localhost:8000/docs` Swagger UI가 열린다
- [ ] `.env` 파일이 `.gitignore`에 포함되어 있다

---

### Phase 2 — 백엔드 핵심 API: OCR 업로드 (예상 2.5h)

#### 2-1. FastAPI 앱 뼈대 (`backend/main.py`)

| # | 작업 | 내용 |
|---|------|------|
| 2-1-1 | FastAPI 앱 초기화 | CORS 설정, 라우터 등록, 정적 파일 서빙 |
| 2-1-2 | `backend/data/expenses.json` 초기 파일 생성 | 빈 배열 `[]` 저장 |
| 2-1-3 | `uploads/` 디렉토리 자동 생성 로직 | 업로드 파일 임시 저장 경로 |

```python
# backend/main.py 핵심 구조
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from routers import upload, expenses, summary

app = FastAPI()
app.add_middleware(CORSMiddleware, allow_origins=["*"], ...)
app.include_router(upload.router, prefix="/api")
app.include_router(expenses.router, prefix="/api")
app.include_router(summary.router, prefix="/api")
```

#### 2-2. 스토리지 서비스 (`backend/services/storage_service.py`)

| # | 작업 | 내용 |
|---|------|------|
| 2-2-1 | `load_expenses()` 함수 | `expenses.json` 읽기 → 리스트 반환 |
| 2-2-2 | `save_expenses(data)` 함수 | 리스트 → `expenses.json` 쓰기 |
| 2-2-3 | `append_expense(item)` 함수 | UUID 생성 후 리스트에 추가 저장 |

#### 2-3. OCR 서비스 (`backend/services/ocr_service.py`)

| # | 작업 | 내용 |
|---|------|------|
| 2-3-1 | 이미지 전처리 함수 | JPG/PNG → Base64, PDF → 이미지 변환 후 Base64 |
| 2-3-2 | LangChain Chain 구성 | `ChatUpstage` + `PromptTemplate` + `JsonOutputParser` |
| 2-3-3 | 시스템 프롬프트 작성 | JSON 스키마 명시, 한국어/영어 영수증 파싱 지시 |
| 2-3-4 | `parse_receipt(file_bytes, content_type)` 함수 | 전처리 → LLM 호출 → JSON 반환 |

```python
# 시스템 프롬프트 핵심 구조
SYSTEM_PROMPT = """
당신은 영수증 OCR 전문가입니다.
이미지에서 아래 JSON 형식으로만 응답하세요. 다른 텍스트는 포함하지 마세요.

{
  "store_name": "string",
  "receipt_date": "YYYY-MM-DD",
  "receipt_time": "HH:MM or null",
  "category": "식료품|외식|교통|쇼핑|의료|기타",
  "items": [{"name": "string", "quantity": int, "unit_price": int, "total_price": int}],
  "subtotal": int,
  "discount": int,
  "tax": int,
  "total_amount": int,
  "payment_method": "string or null"
}
"""
```

#### 2-4. 업로드 라우터 (`backend/routers/upload.py`)

| # | 작업 | 내용 |
|---|------|------|
| 2-4-1 | `POST /api/upload` 엔드포인트 | `UploadFile` 수신, 형식/크기 검증 |
| 2-4-2 | 파일 검증 로직 | 허용 MIME 타입 체크, 10MB 초과 차단 |
| 2-4-3 | OCR 서비스 호출 및 응답 | `parse_receipt()` → UUID 부여 → JSON 반환 |
| 2-4-4 | 오류 처리 | 400 (파일 오류), 500 (OCR 실패) HTTPException |

#### 완료 기준
- [ ] `curl -X POST /api/upload -F "file=@receipt.jpg"` 실행 시 구조화 JSON이 반환된다
- [ ] 10MB 초과 파일 업로드 시 400 오류가 반환된다
- [ ] PDF 파일 업로드 시 정상적으로 파싱된다

---

### Phase 3 — 백엔드 부가 API (예상 1.0h)

#### 3-1. 지출 CRUD 라우터 (`backend/routers/expenses.py`)

| # | 작업 | 엔드포인트 | 내용 |
|---|------|-----------|------|
| 3-1-1 | 전체 조회 | `GET /api/expenses` | `from`, `to` 쿼리로 날짜 필터링 |
| 3-1-2 | 항목 삭제 | `DELETE /api/expenses/{id}` | UUID로 항목 찾아 제거, 없으면 404 |
| 3-1-3 | 항목 수정 | `PUT /api/expenses/{id}` | 요청 Body로 필드 부분 업데이트 |

#### 3-2. 통계 라우터 (`backend/routers/summary.py`)

| # | 작업 | 엔드포인트 | 내용 |
|---|------|-----------|------|
| 3-2-1 | 합계 조회 | `GET /api/summary` | `month` 쿼리로 월별 필터, 총합·카테고리별 합계 반환 |

```json
// GET /api/summary 응답 예시
{
  "total_amount": 128500,
  "this_month_amount": 45300,
  "category_summary": [
    { "category": "식료품", "amount": 32000 },
    { "category": "외식", "amount": 13300 }
  ]
}
```

#### 완료 기준
- [ ] Postman으로 5개 엔드포인트(`POST /upload`, `GET/DELETE/PUT /expenses`, `GET /summary`) 전체 정상 응답 확인
- [ ] `GET /api/expenses?from=2025-07-01&to=2025-07-31` 날짜 필터가 동작한다
- [ ] 존재하지 않는 ID로 DELETE 시 404가 반환된다

---

### Phase 4 — 프론트엔드 환경 설정 (예상 0.5h)

| # | 작업 | 산출물 |
|---|------|--------|
| 4-1 | Vite + React 프로젝트 초기화 | `frontend/` 디렉토리, `package.json` |
| 4-2 | TailwindCSS 설치 및 설정 | `tailwind.config.js`, `postcss.config.js` |
| 4-3 | 커스텀 애니메이션 등록 | `tailwind.config.js` keyframes 추가 |
| 4-4 | Axios 인스턴스 구성 | `frontend/src/api/axios.js` |
| 4-5 | React Router 설치 및 라우팅 설정 | `App.jsx` — 3개 경로 등록 |
| 4-6 | 폰트 설정 | Pretendard CDN → `index.html` `<link>` 추가 |

```js
// frontend/src/api/axios.js
import axios from 'axios'
const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000',
})
export default api
```

#### 완료 기준
- [ ] `npm run dev` 실행 시 `http://localhost:5173` 에서 React 앱이 열린다
- [ ] TailwindCSS 클래스가 정상 적용된다
- [ ] `/`, `/upload`, `/expense/:id` 3개 경로가 라우팅된다

---

### Phase 5 — 업로드 화면 구현 (예상 1.5h)

#### 5-1. 공통 컴포넌트 선 구현

| # | 컴포넌트 | 파일 | 내용 |
|---|----------|------|------|
| 5-1-1 | Header | `components/Header.jsx` | 로고 + "영수증 추가" 버튼 |
| 5-1-2 | Toast | `components/Toast.jsx` | success/error/info 3종, 3초 자동 소멸 |
| 5-1-3 | Badge | `components/Badge.jsx` | 카테고리별 색상 뱃지 |

#### 5-2. 업로드 전용 컴포넌트

| # | 컴포넌트 | 파일 | 구현 내용 |
|---|----------|------|----------|
| 5-2-1 | DropZone | `components/DropZone.jsx` | 드래그 앤 드롭, 클릭 업로드, drag over 스타일 변경 |
| 5-2-2 | ProgressBar | `components/ProgressBar.jsx` | 너비 애니메이션, OCR 처리 중 텍스트 |
| 5-2-3 | ParsePreview | `components/ParsePreview.jsx` | 파싱 결과 인라인 편집 폼, 저장/취소 버튼 |

#### 5-3. 업로드 페이지 조립

| # | 작업 | 내용 |
|---|------|------|
| 5-3-1 | `UploadPage.jsx` 구현 | DropZone → POST /api/upload → ProgressBar → ParsePreview 흐름 |
| 5-3-2 | 파일 선택 시 즉시 API 호출 | `onChange` 핸들러에서 FormData 전송 |
| 5-3-3 | 저장 버튼 핸들러 | 수정된 JSON → `localStorage` 병행 저장 |
| 5-3-4 | 오류 상태 처리 | OCR 실패 시 빨간 배너 + 재시도 버튼 표시 |

#### 완료 기준
- [ ] 이미지를 드래그 앤 드롭하면 OCR 파싱이 실행된다
- [ ] ProgressBar가 처리 중 표시되고 완료 후 숨겨진다
- [ ] ParsePreview에서 필드를 수정하고 저장 시 대시보드로 이동한다
- [ ] Toast 알림이 저장 성공 시 표시된다

---

### Phase 6 — 대시보드 화면 구현 (예상 1.5h)

#### 6-1. 대시보드 전용 컴포넌트

| # | 컴포넌트 | 파일 | 구현 내용 |
|---|----------|------|----------|
| 6-1-1 | SummaryCard | `components/SummaryCard.jsx` | 총 지출 / 이번달 지출 / 카테고리별 요약 카드 |
| 6-1-2 | FilterBar | `components/FilterBar.jsx` | 날짜 범위 입력, 조회/초기화 버튼 |
| 6-1-3 | ExpenseCard | `components/ExpenseCard.jsx` | 가게명, 날짜, 금액, 카테고리 뱃지, 클릭 이동 |

#### 6-2. 대시보드 페이지 조립

| # | 작업 | 내용 |
|---|------|------|
| 6-2-1 | `Dashboard.jsx` 구현 | 마운트 시 `GET /api/expenses` + `GET /api/summary` 호출 |
| 6-2-2 | 필터 적용 | FilterBar 날짜 범위 → 쿼리 파라미터로 API 재호출 |
| 6-2-3 | 빈 상태(Empty State) | 지출 내역 없을 때 안내 메시지 + 등록 버튼 |
| 6-2-4 | 반응형 그리드 | `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3` |

#### 완료 기준
- [ ] 대시보드 진입 시 저장된 지출 내역이 카드 목록으로 표시된다
- [ ] SummaryCard에 총 지출 / 이번달 지출 금액이 표시된다
- [ ] 날짜 필터 적용 시 해당 기간 내역만 표시된다
- [ ] 내역이 없을 때 Empty State가 표시된다

---

### Phase 7 — 지출 상세/수정 화면 구현 (예상 1.0h) `Should Have`

| # | 컴포넌트/작업 | 파일 | 구현 내용 |
|---|--------------|------|----------|
| 7-1 | Modal | `components/Modal.jsx` | 삭제 확인 다이얼로그, 오버레이 클릭으로 닫기 |
| 7-2 | ReceiptImage | `components/ReceiptImage.jsx` | 원본 이미지 미리보기, 비율 유지 |
| 7-3 | EditForm | `components/EditForm.jsx` | 전 필드 인라인 편집, 품목 목록 수정 지원 |
| 7-4 | ExpenseDetail 페이지 | `pages/ExpenseDetail.jsx` | ID로 `GET /api/expenses` 조회 → 수정/삭제 |
| 7-5 | 삭제 흐름 | - | Modal 확인 → `DELETE /api/expenses/{id}` → 대시보드 이동 |
| 7-6 | 수정 저장 흐름 | - | `PUT /api/expenses/{id}` → Toast 성공 → 현재 화면 갱신 |

#### 완료 기준
- [ ] ExpenseCard 클릭 시 상세 페이지로 이동한다
- [ ] 필드 수정 후 "수정 저장" 클릭 시 PUT API가 호출되고 Toast가 표시된다
- [ ] "삭제" 클릭 시 Modal이 열리고, 확인 시 항목이 삭제되어 대시보드로 이동한다

---

### Phase 8 — 배포 및 E2E 검증 (예상 1.0h)

#### 8-1. Vercel 배포 설정

| # | 작업 | 내용 | 상태 |
|---|------|------|------|
| 8-1-1 | `vercel.json` 작성 | 프론트 빌드(`@vercel/static-build`) + 백엔드 서버리스(`@vercel/python` + Mangum) 라우팅 | ✅ 완료 |
| 8-1-2 | GitHub 레포 연동 | Vercel 프로젝트 생성 → GitHub Import | ⬜ 수동 |
| 8-1-3 | 환경변수 등록 | `UPSTAGE_API_KEY` (Vercel 대시보드) | ⬜ 수동 |
| 8-1-4 | 첫 배포 실행 | `git push origin main` → 자동 빌드 트리거 | ⬜ 수동 |

##### Vercel 등록 필수 환경변수

| 키 | 값 | 비고 |
|----|-----|------|
| `UPSTAGE_API_KEY` | `up_xxx...` | Upstage 콘솔에서 복사 |
| `VERCEL` | `1` | 자동 주입 (별도 설정 불필요) |

> `VITE_API_BASE_URL`과 `DATA_FILE_PATH`는 설정하지 않아도 됩니다.
> `VITE_API_BASE_URL`은 `.env.production`에서 `""`(빈값)으로 설정되어 같은 도메인 상대 경로를 사용하며,
> `DATA_FILE_PATH`는 `VERCEL=1` 감지 시 자동으로 `/tmp/expenses.json`을 사용합니다.

#### 8-2. E2E 시나리오 검증 체크리스트

| # | 시나리오 | 기대 결과 | 우선도 |
|---|----------|----------|--------|
| E2E-01 | JPG 영수증 업로드 → OCR 파싱 | 구조화 JSON이 ParsePreview에 표시됨 | Must |
| E2E-02 | ParsePreview 필드 수정 → 저장 | 대시보드에 수정된 내용으로 카드 추가됨 | Must |
| E2E-03 | 대시보드 날짜 필터 적용 | 해당 기간 내역만 필터링됨 | Must |
| E2E-04 | ExpenseCard 클릭 → 상세 진입 | 상세 정보 및 원본 이미지 표시됨 | Should |
| E2E-05 | 항목 삭제 → 목록 갱신 확인 | 삭제된 카드가 목록에서 제거됨 | Should |
| E2E-06 | PDF 파일 업로드 | 정상 파싱 및 저장됨 | Should |
| E2E-07 | 10MB 초과 파일 업로드 | 오류 메시지 표시, 업로드 차단됨 | Must |
| E2E-08 | 지원하지 않는 파일 형식 업로드 | 오류 메시지 표시됨 | Must |

#### 완료 기준
- [ ] Vercel 배포 URL에서 E2E-01 ~ E2E-03 시나리오가 모두 통과한다
- [ ] 브라우저 콘솔에 CORS 오류가 없다
- [ ] 환경변수가 정상 주입되어 Upstage API 호출이 성공한다

---

## 15. 리스크 및 대응 방안

| 리스크 | 영향 | 대응 방안 |
|--------|------|----------|
| Vercel 서버리스 파일 시스템 비지속 | 데이터 손실 | localStorage 병행 저장 또는 Railway 배포로 전환 |
| Upstage API 응답 시간 초과 | OCR 파싱 실패 | 타임아웃 설정 (15초), 재시도 안내 메시지 표시 |
| OCR 파싱 정확도 낮음 | 잘못된 데이터 저장 | ParsePreview에서 사용자 직접 수정 후 저장 가능하도록 처리 |
| PDF 변환 실패 (pdf2image) | PDF 업로드 불가 | Poppler 설치 가이드 제공, 오류 시 이미지 변환 안내 |
| 1일 스프린트 일정 초과 | 기능 미완성 | Should Have 기능 후순위 처리, Must Have 우선 완료 |

---

## 16. 용어 정의

| 용어 | 설명 |
|------|------|
| OCR | 광학 문자 인식 (Optical Character Recognition) |
| LLM | 대형 언어 모델 (Large Language Model) |
| Vision LLM | 이미지 입력을 처리할 수 있는 멀티모달 LLM |
| LangChain | LLM 오케스트레이션 프레임워크 |
| Upstage | 한국 AI 기업, document-digitization-vision 모델 제공 |
| E2E | End-to-End, 전체 흐름 테스트 |
| MVP | Minimum Viable Product, 최소 기능 제품 |
| UUID | Universally Unique Identifier, 전역 고유 식별자 |

---

*Receipt Expense Tracker | PRD v1.3 | 본 문서는 프로그램 개요서 v1.0 기반으로 작성되었습니다.*
*v1.1: 화면 디자인 & 스타일 가이드(섹션 10) 추가 | v1.2: 개발 일정 Phase별 세분화(섹션 13) 추가 | v1.3: 바이브 코딩 3원칙 적용(섹션 13) 추가*
