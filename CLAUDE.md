# accouting-work — Claude 작업 가이드

## 앱 구조
단일 파일 모바일 웹앱 (`index.html`), 바닐라 JS, 다크테마. Netlify 배포 (main 브랜치 push → 자동 배포).

**탭 3개**:
1. **전표입력** — SAP 전표 양식 (DEFAULT_FORMS 기반)
2. **전표요약** — 매월 반복 전표 체크리스트 (SUMMARY 기반)
3. **숙소현황** — 직원 숙소 정보

---

## 핵심 데이터 구조

### SUMMARY (line ~308)
전표요약 탭. 86개 항목, 매월 반복 처리 전표 목록.
```js
{ord:1, date:"처리시기", name:"항목명", img:"파일명.png", 적요:"전표적요", pjt:"BBW1002", acct:"계정설명", chk:"정산|비정산|-"}
```
- `img`: `/images/` 폴더 기준 파일명 (SUMMARY_BASE = GitHub Pages URL)
- `chk`: "정산" = 초록, "비정산" = 빨강, 그 외 = 회색

### DEFAULT_FORMS
전표양식 탭 기본 양식 58개. 2026년 8~9월 SAP 전표 캡처 기준으로 값 채움.
```js
{type:'tax|normal|jiro|silmool|chaekwon|kita', kita:'arap|jeondo', title:'표시명', _default:true,
 fields:{'f-...':값, _rows:[행...], _ar:[AR/AP 반제행...], _jd:[전도금 미결행...]}}
```
- `f-dept`: 현장코드 (BBW1002=구미, BBW1003=구미증설, BBW1004=김천)
- `f-bikmok`: CC_BIKMOK[cc] 목록 값과 글자 그대로 일치해야 함 (BBW1003은 `계약외-비정산`, 나머지는 `계약 외-비정산`)
- 세금코드는 SAP 표기 그대로 (`VB 매입-세금계산서-일반-전자증빙 (10%)` 등). select에 없는 값도 `_setVal`이 옵션을 추가해 표시
- 행별 `f-taxnm`이 헤더와 다르면(예: 전기료 2행 `세금코드 없음`) 행 값 유지
- 불러올 때 토큰 치환 (`_tplResolve`): `{M}`/`{M-1}`/`{M+1}` → 처리월 기준 월, `@prevEnd @thisFirst @this15 @this20 @thisEnd @next15 @next20 @nextEnd @next3End` → 날짜
- 개인 계좌번호는 넣지 않음 (공개 저장소)
- 수정 후 검증: Playwright로 모든 양식 `fLoadDefault(i)` → 필드값 비교 (잠금화면은 `sessionStorage._auth_ok='1'`로 통과)

### CC_BIKMOK (line ~730)
계약비목 드롭다운 옵션. CC코드별로 다름.

---

## 이미지 관리

### 파일 위치
- 로컬: `/home/user/accouting-work/images/`
- 배포: `https://dudwnddl79-bot.github.io/accouting-work/images/`

### 네이밍 규칙
`{업무종류}_{현장}_{월}.{ext}`
- 현장: gumi(구미), up(증설/BBW1003), gimcheon(김천)
- 월: jul(7월), aug(8월), sep(9월) ...
- 예: `sikdae_gumi_aug.png`, `utility_pesu_jul.png`

### ZIP 파일 처리
사용자가 제공하는 zip에 #U 인코딩 한글 파일명이 들어있음.
```python
# 디코딩: #U4F4D → chr(0x4F4D) = '位'
re.sub(r'#U([0-9a-fA-F]{4})', lambda m: chr(int(m.group(1),16)), encoded_name)
```

**→ `scripts/update_images.py` 사용** (아래 참고)

---

## 반복 작업 절차

### 1. 새 ZIP 이미지 반영
```bash
# zip을 scratchpad에 압축 해제 후:
python3 scripts/update_images.py --zipdir /tmp/.../zipcontents --mapping scripts/image_mapping.json
```

### 2. SUMMARY 이미지 업데이트
SUMMARY의 `img` 필드를 새 파일명으로 교체 (Python으로 직접 치환).

### 3. DEFAULT_FORMS 추가/수정
전표 이미지를 보고 필드값 추출. f-bikmok은 fOnCC 호출 이후에 재설정 필요 (버그 수정 이미 반영됨).

### 4. 커밋/푸시/머지
```bash
git add index.html images/
git commit -m "메시지\n\nCo-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
git push -u origin claude/remove-names-images-e3h4yn
# PR 생성 → merge (MCP github 툴 사용)
# 충돌 시: git fetch origin main && git rebase origin/main && git push --force-with-lease
```

---

## 현장 코드
| 코드 | 현장 |
|------|------|
| BBW1002 | 구미 코오롱 폐수 |
| BBW1003 | 구미 증설 |
| BBW1004 | 김천 코오롱 폐수 |

## 주요 공급업체
- 코오롱인더스트리(주) 구미공장 — 위탁운영비(구미)
- 코오롱인더스트리(주) 김천공장 — 위탁운영비(김천 1공장)
- 코오롱인더스트리(주) 김천3공장 — 위탁운영비(김천 3공장)
- (주)서브원 — 안전장비
- 코오롱글로벌(주)FS김천1점 — 카페 식대

## 중요 규칙 (사용자 지시)
- **반드시 존댓말 사용**
- **매 수정 후 동작 검증 필수**
- **결재서류 이미지 속 개인 이름 제거 (HTML 텍스트는 수정 금지)**
- 개발 브랜치: `claude/remove-names-images-e3h4yn`
