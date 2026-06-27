# Handoff: 디디에어컨(DD AIRCON) 차량용 자석시트 배너 편집기

## Overview
차량(포터)용 자석시트 광고 배너를 직접 편집·다운로드하는 단일 페이지 웹 편집기다.
사용자가 **측면 배너(2050×230mm)** 와 **후면 배너(1490×230mm)** 두 종을 한 화면에서
드래그·크기조절·문구수정 하고, "PNG 저장"으로 두 장을 한 번에 내보낸다.

대상 사용자: 광고 배너를 직접 만들어야 하는 비전문가(에어컨 설치 자영업자). 모바일/데스크톱 모두 지원.

## About the Design Files
이 번들의 `ddaircon-editor/index.html` 은 **HTML로 만든 동작하는 디자인 레퍼런스(프로토타입)** 다.
실제로 작동하지만, 그대로 프로덕션에 넣으라는 의미는 아니다. 목표는 **이 편집기의 UI와 동작을
대상 코드베이스의 기존 환경(React/Vue/Svelte 등)과 패턴으로 재구현**하는 것이다.
아직 환경이 없다면, 프로젝트에 가장 적합한 프레임워크를 골라 구현하면 된다.

단, 이 프로젝트는 의존성이 전혀 없는 순수 HTML+Canvas 단일 파일이라 **그대로 호스팅해도
바로 동작**한다. "리팩터링/확장"이 목적이 아니면 파일을 그대로 써도 무방하다.

## Fidelity
**High-fidelity (hifi).** 최종 색상·타이포·간격·아이콘·내보내기 동작까지 모두 확정된 상태다.
픽셀 단위 좌표는 `index.html` 의 `DEF_SIDE` / `DEF_REAR` 배열에 그대로 들어있으므로,
재구현 시 그 값을 그대로 옮기면 동일한 결과가 나온다.

---

## 핵심 개념 (좌표계)
- 각 배너는 자체 **디자인 좌표계** 를 가진다: 측면 `D={W:2050,H:230}`, 후면 `D={W:1490,H:230}` (단위 px = mm 1:1).
- 모든 요소(`layer`)의 `x,y` 는 이 좌표계 기준 절대 위치(왼쪽·위 기준, px).
- 화면 표시 시 stage 폭에 맞춰 비율(`sf = stage.clientWidth / D.W`)로 환산해 `%` 로 배치한다.
- PNG 내보내기는 `D.W × SC`, `D.H × SC` (`SC=2`) 캔버스에 같은 좌표를 그대로 곱해 그린다 → 측면 4100×460, 후면 2980×460px.
- 위·아래 **네이비 띠(band)** 는 높이 `BAND=18`(px, 디자인좌표) 의 사각형으로 top/bottom 에 그린다.

## 레이어(요소) 타입
| type | 의미 | 주요 속성 |
|------|------|-----------|
| `img` | 이미지 (로고, 네이버 검색박스, 아이콘) | `asset`, `x`, `y`, `w`(디자인폭), `scale` — 높이는 원본 비율 auto |
| `text` | 글자 (서비스 문구, 전화번호) | `text`, `x`, `y`, `fontPx`, `weight`, `color`, `scale` |
| `rule` | 세로 구분선 | `x`, `y`, `w`(두께), `h`(높이), `color`, `scale` |

- 텍스트는 `white-space: pre` 로 렌더 → **사용자가 입력한 공백(띄어쓰기)이 그대로 반영**된다. (중요 요구사항)
- 텍스트 letter-spacing: -1px, line-height: 1.
- 폰트: `'Noto Sans KR','Malgun Gothic',sans-serif` (Google Fonts Noto Sans KR, weight 400~900 로드).

## Screens / Views

### 단일 화면 (편집기)
세로 스택 레이아웃, 중앙 정렬, 배경 `#eef1f4`.

1. **헤더** — 제목 "DD에어컨 배너 편집기" (`#16324f`, 800), 부제 회색 안내문.
2. **측면 배너 카드** — 흰 카드(`#fff`, border `#d6dde4`, radius 10, 그림자), 안에 제목 라벨 + `#stageSide`(aspect-ratio 2050/230).
3. **후면 배너 카드** — 동일 구조, `#stageRear`(aspect-ratio 1490/230).
4. **컨트롤 패널** (max-width 560 흰 카드):
   - "선택한 요소" 라벨 (선택 시 `배너이름 · 요소라벨`, 파랑 `#0e74c9`)
   - "문구 수정" 텍스트 입력 (text 레이어 선택 시 활성)
   - "요소 크기" 슬라이더 (scale 0.4~2.2, step 0.02) + 퍼센트 표시
   - "테두리 색상" 토글: `네이비(#14246E)` / `원래 파랑(#2A6AC4)`
   - "로고" 토글: `지금 로고`(평면 DD AIRCON, `assets/dd_logo_editor.png`) / `기존 로고`(입체 돔 로고, base64 `ASSETS.ddLogo`)
   - 버튼: "전체 초기화"(회색) / "PNG 저장"(파랑, 측면·후면 2장 다운로드)
   - 상태 텍스트 (저장 진행/완료/오류)

## Interactions & Behavior
- **드래그 이동**: 요소에 `pointerdown` → 전역 `pointermove` 로 `x,y` 갱신(디자인좌표로 환산), `pointerup` 종료. 드래그 시 해당 요소 자동 선택.
- **선택**: 요소 클릭 시 파란 outline(2px `#0e74c9`, offset 2). 빈 stage 클릭 시 선택 해제.
- **문구 수정**: 입력값 즉시 해당 text 레이어 `text` 갱신·리렌더.
- **크기**: 슬라이더가 `scale` 갱신. img=폭배율, text=폰트배율, rule=두께·높이 동시 배율.
- **테두리 색상 토글**: 두 배너의 위·아래 띠 색 동시 변경.
- **로고 토글**: 두 배너의 `ddLogo` 이미지 src 동시 교체.
- **전체 초기화**: `layers = JSON.parse(JSON.stringify(DEF))` 로 양쪽 배너를 기본 배열로 되돌림.
- **PNG 저장**: 각 배너를 오프스크린 캔버스에 SC=2 로 렌더 → `toBlob` → `a[download]` 로 순차 다운로드. 파일명 `DD_배너_측면_2050x230.png`, `DD_배너_후면_1490x230.png`.
- **반응형**: stage 는 `aspect-ratio` + `width:100%`. `resize` 시 `relayoutAll()`. 폰트 로드 완료(`document.fonts.ready`) 후 재배치.

## State Management
- `banners[]`: 각 배너 객체 `{ key, name, D, stage, def, layers[], els{}, topBand, botBand }`.
- `layers`: 현재 편집 중 레이어 배열(배너별). `def`: 초기 기본값(불변, 초기화 기준).
- `selected`: `{ bn, id } | null` — 현재 선택된 (배너, 레이어id).
- `bandColor`: `'navy' | 'blue'` (기본 `'blue'`).
- `logoChoice`: `'current' | 'orig'` (기본 `'orig'`).
- 데이터 패칭 없음. 전부 클라이언트 상태. (저장 영속화는 미구현 — 확장 후보: localStorage)

## Design Tokens
- 네이비(브랜드/카테고리 글자): `#14246E`
- 띠 파랑(대체): `#2A6AC4`
- 강조 빨강(서비스 동작 문구): `#D11E2B`
- 검정(휴대폰 번호): `#141414`
- 네이버 그린(검색박스 이미지 내): `#03C75A`
- 구분선 회색: `#d5d9e0`(굵은) / `#cdd4de`(가는)
- UI: 배경 `#eef1f4`, 카드 테두리 `#d6dde4`, 본문 텍스트 `#16324f`, 보조 `#5b6b7a`, 액션 파랑 `#0e74c9`
- 띠 높이: 18px(디자인좌표) · 카드 radius 10 · 버튼 radius 7~8
- 폰트: Noto Sans KR (400/500/700/800/900)

## Assets
모든 이미지는 `index.html` 안에 base64(`ASSETS` 객체)로 내장되어 **외부 의존성이 없다**.
- `ASSETS.ddLogo` — 기존(입체 돔) DD AIRCON 로고
- `ASSETS.naver` — "디디에어컨" 네이버 검색박스 이미지
- `assets/dd_logo_editor.png` — 지금(평면) DD AIRCON 로고. **상대경로 참조** (`../assets/...`) 이므로 이 파일도 함께 가져가야 함. 같이 동봉됨.
- 수화기/핸드폰 아이콘 — 코드 내 인라인 SVG data URI(`ICONS.handset` 네이비, `ICONS.mobile` 검정). 별도 파일 없음.

## Files
- `ddaircon-editor/index.html` — 편집기 본체(단일 파일, 전부 포함). DEF_SIDE/DEF_REAR 가 레이아웃 기본값.
- `assets/dd_logo_editor.png` — 평면 로고(상대경로로 참조됨).

## 확장 아이디어 (선택)
- 배치 상태 localStorage 영속화(새로고침 유지)
- 서비스 문구 줄 추가/삭제 버튼
- 전화번호 입력 1곳 → 두 배너 동시 반영
- 요소 정렬/스냅 가이드, undo/redo
- 실제 인쇄용 PDF(벡터) 내보내기
