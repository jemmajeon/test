# 수신거부 · UTM 우회 + 토큰 매핑 패턴 (2026-10-15)

> 다른 메일·캠페인에도 같은 방식으로 쓰도록 남긴 하네스. 코칭 메일 A/B 에 처음 적용했다. 규칙 정본은 `COACH_MAIL_TEMPLATES.md` §5.

## 왜 이렇게 했나
- 메일에 `n8n-fgs-009-web.hanpda.com/webhook/...` 를 그대로 걸면 고객에게 사내 자동화 주소가 보인다(사용자: «n8n 링크라서 띄우면 안 돼»).
- 링크에 몰 ID·이메일을 싣고 싶지 않다 → **링크에는 불투명 토큰만**, 몰/이메일/발송일은 **내부 DB 에서만** 푼다.
- 웨비나 때 쓰던 «개인 GitHub Pages 우회(`jemmajeon.github.io/test/`) + GA4 UTM» 이 문제없이 수집됐다 → 같은 틀을 재사용(사용자 결정).

## 구성 (한 줄 흐름)
메일 링크(토큰 UTM) → GitHub Pages `unsub.html`(버튼) → n8n `pro-unsubscribe` 접수 → 사내 MySQL `jhjun01_welcome_unsub`(+ n8n 팀채널 @here) → `unsub_history.sync()` 가 로컬 `call_records/unsub_history.db` 로 가져와 토큰을 몰·이메일·단계·발송일로 복원 → 워크스페이스 VOC · 콜카드 띠 · ERP 카드 · 혜미님 CRM.

| 조각 | 파일 | 역할 |
|---|---|---|
| 링크 생성 | `coach_stage_plan.py render()` | `UNSUB_BASE?utm_source=email&utm_medium=coach_stage&utm_campaign=pro_coach_unsub&utm_content=<토큰>` · `issue_token()` |
| 우회 페이지 | `unsub_page/unsub.html` | **버튼을 눌러야** 처리(스캐너 오작동 방지) · GA4 `G-BSZWXLY5DE`(웨비나와 같은 속성 529725254) · n8n 에는 `mall=tk_<토큰>&uid=<토큰>` 만 |
| 이력·매핑 | `unsub_history.py` | `issue_token`, `sync`(토큰→몰 복원), `unsubbed_malls`(발송 제외), `workspace_rows`(우리 액션 VOC), `push_crm`(CRM 1회) |
| 조회 | 8095 `GET /unsub/history?mall=` | 콜카드·ERP 가 읽음 (보기 전용) |
| 표시 | 콜카드 `UNSUB_BADGE_V1`, ERP `unsub-card`(ERP_UNSUB_V1), 워크스페이스 «우리 액션» | 소통채널 메일 · VOC 수신거부 요청 |

## 다른 캠페인에 쓰는 법
1. `unsub_history.issue_token(mall_id, email, stage, sent_date)` 로 토큰 발급 → `utm_content=<토큰>`, `utm_campaign` 만 캠페인 이름으로 바꾼다.
2. 페이지는 그대로 재사용(`unsub.html` 은 `utm_content` 만 읽는다). 새 캠페인이 GA4 에서 따로 보이는 건 `utm_campaign` 값 덕분.
3. `_lane()`(unsub_history.py)에 새 `step` 접두어를 넣으면 이력에서 갈래가 구분된다.

## 함정 (실측)
- **n8n 은 `step` 을 저장하지 않는다**(코칭 단계는 토큰 매핑으로 복원). 옛 n8n 직접 링크(옛 메일)를 누르면 이력엔 `unknown` 으로 쌓인다.
- **페이지를 GitHub 에 올리는 건 사용자 몫** — 개인 계정 `jemmajeon`/`test` 저장소 루트. 올린 뒤 `HEAD 200` 확인 전 실발송 금지. `gh` 는 회사 계정만 로그인돼 있다.
- 회사 조직(`cafe24-corp-automation`)은 Enterprise 라 **공개 Pages 생성 불가**(`members_can_create_public_pages=false`) → 고객이 열 수 있는 공개 호스팅은 개인 계정뿐(회사 지침 예외 — 사용자 판단).
- LIKE 에서 `tk_` 의 `_` 는 와일드카드 → `ESCAPE '!'` 로 막는다(백슬래시를 문자열에 넣으면 따옴표를 깨뜨린다).
- 테스트몰(`qwerty89`)은 이력엔 남되 `is_test=1` 이라 CRM·VOC 합치기·발송 제외에서 빠진다. 시험 뒤 n8n 접수 행(`tk_…`)은 사용자 확인 후 지운다.
- CRM 전송은 새 **실제** 수신거부에만(1회) · 전송 전 «우리가 메일을 보낸 적이 있는 몰인가» 를 확인(토큰 접수는 자체가 증거, 아니면 발송 로그) — 없으면 `보류` 로 두고 슬랙으로 확인 요청(jkr870 사고) · 담당 `전지혜`. ⚠️ `전지혜_자동` 은 **상위몰 웰컴메일 발송에만** 쓴다(사용자 2026-10-15) — 그 외 발송·기록엔 «자동화» 표현을 안 쓴다.
- 위조: 토큰 검증은 아직 없다(몰 ID 를 아는 사람이 옛 방식 URL 로 접수 가능) — 사용자 결정으로 버튼만 두고 감수.

## 접수 후 자동 처리 (8095 `unsub_worker` · 2분)
`sync()`(토큰 복원 → CRM 전송) → `notify_new()`(팀 채널에 몰 이름·메일 종류·발송일·CRM 결과). 테스트 행(`is_test`)은 CRM·슬랙·VOC 에서 빠진다. 이력 삭제가 필요하면 n8n 접수 행(`jhjun01_welcome_unsub`)과 로컬 행을 같이 지우고 내용을 `call_records/_deleted_*.json` 에 남긴다.

## 검증 한 번에
```
python unsub_history.py sync && python unsub_history.py list
curl -s localhost:8095/unsub/history | head -c 400
```

## v2 (2026-10-15 오후) — 외부에서 열리는 구조로 교체 (미적용: GitHub 업로드·구글 폼 연결 전)
**발견한 결함**: n8n(`n8n-fgs-009-web.hanpda.com`)은 **사내 전용**이라 고객 브라우저가 못 닿는다. v1 `unsub.html` 은 n8n 호출이 실패해도 4초 뒤 «처리되었습니다» 를 띄웠다 → 고객은 완료로 보지만 접수는 안 됨. `pro-unsubscribe` 웹훅 실행 기록 5건 전부 사내 IP(118.130.18.43)였다(외부 접수 0건). 아직 실고객에게 나간 수신거부 링크는 테스트몰뿐이었다(발송 기록 기준).
**원칙**: 고객 브라우저가 닿는 곳 = GitHub Pages + 구글(캘린더·폼·GA4). 사내(PC/n8n)는 고객이 못 닿으니 **공개 수신처에 쌓인 걸 사내가 가져간다(pull)**. 사내를 외부에 여는 방식(터널·포트)은 보안 지침상 쓰지 않는다.
| 파일 | 역할 |
|---|---|
| `unsub.html`(v2) | n8n 호출 제거. 구글 폼(공개 응답)에 토큰을 `fetch(no-cors)` 로 전송 + GA4 `unsubscribe_click`. **전송이 실패하면 완료라고 하지 않고 재시도/회신 안내.** `FORM_ACTION`·`ENTRY_TOKEN` 이 비면 버튼을 막고 회신 안내(가짜 완료 금지). 폼 주소는 `docs.google.com/forms/d/e/…/formResponse` 만 허용 |
| `meeting.html`(신규) | 미팅 일정 선택 우회. 버튼 → GA4 `meeting_click` + 구글 캘린더 예약 페이지로 이동. 이동 대상은 `CAL` 화이트리스트 키(`o=jh`/`sales`)로만(임의 주소·`__proto__` 차단). `sales`(영업파트 캘린더) 링크는 **미확정 — 비어 있어 버튼이 막힌다** |
| `unsub.bak_20261015_n8n.html` | v1 백업(롤백용) |
**검증**: `scratchpad/test_pages.js`(가짜 브라우저 12항목 통과: 정상·잘못된 토큰·임의 주소·폼 성공/네트워크 실패·미설정). GA4 조회(`run_report`, 속성 529725254)는 이 PC 에서 되지만 반영이 하루 단위로 늦어 «수신거부 이벤트» 는 아직 못 봤다(내일 재확인).
**남은 일(사용자·후속)**: ① 두 파일을 개인 계정 `jemmajeon/test` 에 업로드 → HEAD 200 확인 전 실발송 금지 ② 구글 폼(공개 응답·«토큰» 단답형 1문항·응답 시트 연결)을 만들어 `FORM_ACTION`/`ENTRY_TOKEN` 채우기 — 제가 폼 생성 권한이 없다 ③ 응답 시트를 읽어 `unsub_history` 에 넣는 수집기(미작성) ④ 일정 선택 메일이 `meeting.html` 링크(토큰)를 쓰도록 `coach_stage_plan.py` 수정(미작성·실발송 코드라 별도 승인) ⑤ 영업파트 캘린더 예약 링크 확인.
