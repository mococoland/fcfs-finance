# UI 디자인 가이드

데모용 단일 페이지(`static/index.html`)에만 적용. 백엔드/동시성이 본질이므로 UI는 **검증용 최소 도구**로 만든다. 마케팅 페이지가 아니다.

## 디자인 원칙
1. 도구처럼 보여야 한다 — 선착순 신청 흐름(로그인 → 상품목록 → 신청 → 상태폴링)을 한 화면에서 확인하는 대시보드.
2. 상태가 즉시 보인다 — 신청 결과(접수/마감/중복/기간외)와 잔여 수량을 명확한 텍스트·색으로.
3. 장식 0 — 애니메이션·그래픽 없이 데이터와 버튼만.

## AI 슬롭 안티패턴 — 하지 마라
| 금지 사항 | 이유 |
|-----------|------|
| backdrop-filter: blur() | glass morphism은 AI 템플릿의 가장 흔한 징후 |
| gradient-text (배경 그라데이션 텍스트) | AI가 만든 SaaS 랜딩의 1번 특징 |
| "Powered by AI" 배지 | 기능이 아니라 장식. 사용자에게 가치 없음 |
| box-shadow 글로우 애니메이션 | 네온 글로우 = AI 슬롭 |
| 보라/인디고 브랜드 색상 | "AI = 보라색" 클리셰 |
| 모든 카드에 동일한 rounded-2xl | 균일한 둥근 모서리는 템플릿 느낌 |
| 배경 gradient orb (blur-3xl 원형) | 모든 AI 랜딩 페이지에 있는 장식 |

## 색상
### 배경
| 용도 | 값 |
|------|------|
| 페이지 | #0a0a0a |
| 카드 | #161616 |

### 텍스트
| 용도 | 값 |
|------|------|
| 주 텍스트 | #fafafa |
| 본문 | #d4d4d4 |
| 보조 | #a3a3a3 |
| 비활성 | #737373 |

### 데이터/시맨틱 색상
| 용도 | 값 |
|------|------|
| 접수/성공 | #22c55e |
| 마감/에러 | #ef4444 |
| 처리중/대기 | #eab308 |
| 중립/기본 | #525252 |

## 컴포넌트
### 카드
```
border: 1px solid #262626; background: #161616; border-radius: 8px; padding: 20px;
```
### 버튼
```
Primary: background:#fafafa; color:#0a0a0a; border-radius:6px; padding:10px 16px;
Disabled: opacity:0.4; cursor:not-allowed;  (마감/처리중일 때)
```
### 입력 필드
```
background:#0f0f0f; border:1px solid #262626; border-radius:6px; padding:10px 12px; color:#fafafa;
```

## 레이아웃
- 전체 너비: max-width 720px, 좌측 정렬 기본.
- 섹션 간격: 24px. 요소 간격: 12px.

## 타이포그래피
| 용도 | 스타일 |
|------|--------|
| 페이지 제목 | 28px / 600 / #fafafa |
| 섹션 제목 | 14px / 500 / #a3a3a3 |
| 본문 | 14px / 400 / #d4d4d4 |
| 결과 메시지 | 14px / 500 / 시맨틱 색상 |

## 기술
- Vanilla JS(프레임워크 없음), `fetch`로 API 호출. 토큰은 메모리/`localStorage`.
- 신청 후 `GET .../apply/status`를 짧은 주기로 폴링해 PENDING→CONFIRMED 전이를 보여준다.
- 별도 빌드 없이 Spring 정적 리소스로 동일 출처 서빙(CORS 불필요).

## 애니메이션
- 없음. (상태 텍스트 즉시 갱신만)
