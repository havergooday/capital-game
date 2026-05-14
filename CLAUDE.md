# Capital Game — 개발 문서

## 게임 개요

**오늘의 지구 퀴즈** — Wordle 형식의 일일 지리 퀴즈 게임.  
매일 같은 정답이 출제되며, 추측한 위치와 정답 위치의 거리(km)를 단서로 정답을 좁혀가는 방식.

- **수도 모드**: 세계 195개 수도 중 오늘의 정답 추측
- **국가 모드**: 세계 195개 국가 중 오늘의 정답 추측

거리만 제공하고 방향 정보는 없음 → Worldle보다 더 도전적.

---

## 기술 스택

- **순수 HTML/CSS/Vanilla JS** — 단일 파일(`index.html`)에 모든 코드 포함, 빌드 도구 없음
- **외부 API**: Wikipedia REST API (정답 공개 후 썸네일/설명 표시)
- **배포**: 정적 파일, GitHub Pages 호스팅

---

## 파일 구조

```
capital-game/
└── index.html   ← 전체 게임 코드 (HTML + CSS + JS 통합)
```

단일 파일 구조이므로 모든 수정은 `index.html`에서 이루어짐.

---

## 핵심 게임 로직

### 일일 정답 생성 (`getDailyAnswer`)

```js
function getDailyAnswer(data, salt, avoidName = null) {
  const d = new Date();
  const seed = d.getFullYear() * 10000 + (d.getMonth() + 1) * 100 + d.getDate() + salt;
  let h = seed ^ (seed >>> 16);
  h = Math.imul(h, 0x45d9f3b) | 0;
  h = h ^ (h >>> 16);
  let idx = Math.abs(h) % data.length;
  if (avoidName !== null && data[idx].n === avoidName) {
    idx = (idx + 1) % data.length;
  }
  return data[idx];
}
```

- salt = `0` (수도), salt = `2000` (국가)
- 날짜가 바뀌면 자동으로 새 정답
- `avoidName`: 나라 모드 호출 시 수도 정답의 국가명을 전달해 같은 나라가 겹치는 날 자동 회피 (인덱스 +1)

### 거리 계산 (Haversine 공식)

두 좌표 사이의 대원거리(km) 계산. 방향 정보는 제공하지 않고 거리만 사용자에게 표시.

### 거리별 색상

| 거리 | 색상 |
|------|------|
| 0km (정답) | 초록 `#4ade80` |
| < 500km | 진초록 `#22c55e` |
| < 1500km | 라임 `#84cc16` |
| < 3000km | 노랑 `#eab308` |
| < 6000km | 주황 `#f97316` |
| 이상 | 빨강 `#ef4444` |

---

## 데이터 구조

### 수도 데이터 (`CAPITALS`)

```js
{ n: "서울", c: "대한민국", lat: 37.5665, lng: 126.9780 }
// n: 수도명, c: 국가명(자동완성 보조), lat/lng: 좌표
```

### 국가 데이터 (`COUNTRIES`)

```js
{ n: "대한민국", r: "동아시아", lat: 36.50, lng: 127.80 }
// n: 국가명, r: 지역(15개), lat/lng: 국가 중심 좌표
```

---

## 상태 관리

전역 변수 기반 (프레임워크 없음):

```js
let currentMode = null;  // 'capital' | 'country'
let ANSWER = null;       // 오늘의 정답 객체
let guesses = [];        // 사용자 시도 배열 (거리 오름차순 정렬)
let won = false;         // 승리 여부
let acIndex = -1;        // 자동완성 선택 인덱스
```

---

## 주요 함수

| 함수 | 역할 |
|------|------|
| `startGame(mode)` | 게임 초기화, 정답 세팅, 화면 전환 |
| `goHome()` | 홈 화면 복귀 |
| `submitGuess()` | 추측 제출, 검증, 거리 계산 |
| `render()` | 추측 결과 카드 렌더링 |
| `showWin()` | 승리 패널 표시 + 위키 로드 |
| `fetchWiki()` | Wikipedia API 호출 |
| `shareResult()` | 결과 클립보드 복사 |
| `selectAC() / closeAC()` | 자동완성 관리 |
| `distColor() / barWidth()` | 거리 → 색상/너비 변환 |

---

## UI 구조

- **홈 화면**: 두 모드 선택 카드 (`#home`)
- **게임 화면**: 입력 + 자동완성 + 결과 목록 (`#game`)
- **승리 패널**: 정답 정보 + 위키 연동 + 공유 버튼

### 디자인

- 수도 모드: 파란 강조색 `#3b82f6`
- 국가 모드: 보라 강조색 `#8b5cf6`
- 배경: 다크모드 전용
- 반응형 (모바일 최적화, max-width: 480px)

---

## URL 라우팅

해시 기반:
- `#capital` → 수도 모드 직접 진입
- `#country` → 국가 모드 직접 진입

---

## 주요 개발 이력

| 버전 | 변경 내용 |
|------|-----------|
| 현재 | 수도/나라 같은 나라 겹침 버그 수정 (avoidName 회피 로직) |
| - | 공유 텍스트 개선 |
| - | 게임 스트릭/통계 기능 삭제 |
| - | 자동완성, 공유 URL, 파비콘, 스트릭 추가 |
| - | 정답 위키백과 상세정보 연동 |
| - | 국가 모드 추가 (도시 모드 제거) |

---

## 개발 시 주의사항

- 단일 파일 구조라 CSS/JS 모두 `index.html` 내 `<style>`, `<script>` 태그 안에 있음
- 데이터 배열(CAPITALS, COUNTRIES)도 스크립트 내에 인라인 정의됨
- 빌드/번들 과정 없음 — 파일 저장 후 브라우저에서 바로 확인 가능
- 통계/스트릭 기능은 의도적으로 제거된 상태 (재추가 요청 없으면 건드리지 않음)
