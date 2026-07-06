# 굿투어 ERP 백엔드 설계서 (2차 개발)

작성일: 2026-07-06
대상: 현재 `/erp` 단일파일 앱(localStorage)을 다중 사용자·서버 저장·실제 문자 발송 체계로 확장

## 1. 목적

1차 MVP는 브라우저 localStorage에 저장되어 **직원 간 데이터 공유가 안 되고**, 문자는 발송 "기록"만 남는다. 2차 백엔드의 목표:

- 모든 직원이 같은 데이터를 실시간으로 보고 수정
- 서버 저장 + 자동 백업 (브라우저 초기화 무관)
- 실제 SMS/LMS 발송 및 결과(성공/실패) 수신
- 고객 입력폼을 진짜 공개 URL로 (토큰 검증 서버 처리)
- 권한을 서버에서 강제 (프런트 우회 불가)

## 2. 아키텍처 권장안

**권장: Supabase + Vercel** (현재 배포 파이프라인 유지)

```text
[직원 PC/모바일 브라우저]
   │  supabase-js (CRUD·실시간 구독)
   ▼
[Supabase]  PostgreSQL + Auth + RLS + Realtime + Edge Functions
   │
   ├─ Edge Function: send-sms ──► 문자업체 API (알리고/솔라피)
   ├─ Edge Function: submit-form (공개 입력폼 제출)
   └─ 스토리지: 견적 PDF, 첨부파일(2차)

[Vercel]  기존 정적 호스팅 그대로 (mingoo 저장소 /erp)
```

### 대안 비교

| 항목 | A. Supabase (권장) | B. Firebase | C. 자체 Node+PostgreSQL |
|---|---|---|---|
| DB | PostgreSQL — 기존 DB 설계서와 1:1 | NoSQL — 스키마 재설계 필요 | PostgreSQL |
| 권한 | RLS로 행 단위 강제 | Security Rules | 직접 구현 |
| 실시간 | Realtime 내장 | 내장 | Socket 직접 구현 |
| 운영 부담 | 낮음 (관리형) | 낮음 | 높음 (서버 관리 필요) |
| 비용 | 무료~$25/월 | 무료~종량제 | 서버비 월 2~5만원+ |
| 판단 | 설계서의 관계형 스키마 그대로 사용 가능 | 관계·집계 쿼리 불리 | 소규모 여행사엔 과함 |

## 3. 데이터베이스

`goodtour-erp-database-spec.md`의 19개 테이블을 **그대로 PostgreSQL로 생성**한다. 프런트 필드명과 이미 일치하므로 변환 비용이 없다.

핵심 결정사항:

- `users`는 Supabase `auth.users`와 1:1 연결 (`id uuid references auth.users`)
- 견적/예약 번호 채번은 **DB 함수**로 (동시 생성 시 중복 방지)
- 삭제 원칙(설계서 21번)은 RLS + 트리거로 강제: `sms_logs`·`change_logs`는 UPDATE/DELETE 불가

```sql
-- 예: 예약번호 채번 (R-20260706-0001)
create sequence res_seq;
create or replace function next_res_number() returns text as $$
  select 'R-' || to_char(now(),'YYYYMMDD') || '-' || lpad(nextval('res_seq')::text, 4, '0');
$$ language sql;

-- 예: 문자 이력 삭제 금지
create policy "sms_logs_no_delete" on sms_logs for delete using (false);

-- 예: 입금 삭제는 관리자만 + 이력 트리거
create policy "payments_delete_admin" on payments for delete
  using (exists (select 1 from users u where u.id = auth.uid() and u.role = 'admin'));
```

인덱스는 설계서 20번 목록 그대로 생성.

## 4. 인증 · 권한

- 로그인: Supabase Auth **이메일+비밀번호** (1차 화면의 계정선택 → 실제 로그인 폼으로 교체)
- 역할: `users.role` (admin/manager/staff/accounting/readonly)
- RLS 정책 매트릭스 (기능명세 1번 기준):

| 테이블 | admin | manager | staff | accounting | readonly |
|---|---|---|---|---|---|
| customers/consultations/quotes | RW | RW | RW | R | R |
| reservations + 세부(배편·숙박·투어·여행자) | RW | RW | RW | R | R |
| payments/refunds | RW | RW | R | RW | R |
| sms_templates | RW | RW | R | R | R |
| users(직원관리) | RW | R | - | - | - |
| 매출분석 뷰 | R | R | - | R | R |

- 퇴사 처리: `is_active=false` + Auth 계정 비활성화 (기존 데이터의 담당자 표기는 유지)

## 5. API 설계

단순 CRUD는 **supabase-js 직접 호출**(RLS가 권한 보장). 아래만 Edge Function으로:

| 함수 | 메서드 | 역할 |
|---|---|---|
| `send-sms` | POST | 템플릿 치환 → 문자업체 API 호출 → `sms_logs` 기록(성공/실패·업체 메시지ID) |
| `submit-form` | POST (공개) | 토큰 검증·만료 확인 → `customer_forms` 저장 → 상담 자동생성 → 담당자 알림 |
| `convert-quote` | POST | 견적→예약 전환 트랜잭션 (번호 채번, 항목 분배, 상태 전이, 이력 기록) |
| `analytics-summary` | GET | 기간별 매출·미수금·손익 집계 (DB 뷰 조회) |

상태 전이 규칙(기능명세 3번)은 `change_logs` 기록과 함께 `convert-quote`/입금 등록 트리거에서 서버측 검증.

## 6. 문자 발송 연동

국내 문자 API 업체 비교 (2026-07 기준, 정확한 단가는 계약 시 재확인):

| 업체 | SMS | LMS | 특징 |
|---|---|---|---|
| 알리고 (aligo.in) | ~9원 | ~27원 | 저렴, REST 단순, 소상공인 많이 씀 |
| 솔라피 (solapi.com) | ~9원 | ~30원 | 카카오 알림톡 통합 좋음, SDK 깔끔 |
| NHN Cloud | ~9.9원 | ~33원 | 대기업 안정성, 콘솔 좋음 |

공통 필수 절차: **발신번호 사전등록** (통신사 규정, 사업자등록증 필요). 광고성 문자를 안 보내므로 080 수신거부 번호는 불필요하지만, 정보성 문자 가이드라인은 준수.

발송 흐름:

```text
직원이 [발송] 클릭
→ Edge Function send-sms (템플릿 치환은 서버에서 재수행 — 위변조 방지)
→ 업체 API 호출 → 응답의 메시지ID 저장
→ sms_logs status: pending → sent/failed (+실패 사유)
→ 실패 시 화면 배지 표시, 재발송 버튼
```

카카오 알림톡은 2차 이후 (템플릿 사전심사 필요, 솔라피 경유 시 추가 개발 적음).

## 7. 고객 입력폼 (공개 페이지)

- URL: `https://<도메인>/erp/#/form/<token>` 그대로 유지
- 제출 시 `submit-form` Edge Function 호출 (anon 키, insert-only)
- 서버에서 토큰 존재·만료·중복제출 검증, IP 기준 rate limit
- 제출 성공 → 상담(견적필요) 자동 생성 → 담당자 대시보드에 노출 (Realtime으로 즉시 반영)

## 8. 실시간 공유

Supabase Realtime 채널 구독:

- `reservations`, `payments`, `consultations` 변경 → 대시보드·목록 자동 갱신
- 동시 수정 충돌은 `updated_at` 비교 후 "다른 직원이 수정했습니다. 새로고침할까요?" 안내로 처리 (소규모 팀에 충분)

## 9. 마이그레이션 (localStorage → 서버)

1. 현재 앱 **설정 → 데이터 관리 → JSON 백업 다운로드** (이 기능이 마이그레이션 출구)
2. 시드 스크립트가 백업 JSON을 읽어 테이블별 INSERT (id 재매핑 포함)
3. 프런트는 저장소 어댑터만 교체:

```js
// 현재: loadDB()/saveDB() → localStorage
// 2차:  같은 인터페이스로 supabase-js 호출 (화면 코드는 거의 무변경)
const store = { list(t){...}, insert(t,row){...}, update(t,id,patch){...} };
```

4. 병행 운영 1주 (구버전 읽기전용) 후 전환 완료

## 10. 보안 · 개인정보

- 전 구간 HTTPS (Vercel/Supabase 기본)
- 개인정보 보유기간: 설정의 동의문대로 "여행 종료 후 1년" — 월 1회 배치(pg_cron)로 만료 고객 비식별화
- `change_logs`/`sms_logs`는 insert-only (감사 추적)
- 백업: Supabase 일일 자동 백업 + 주 1회 JSON 수동 백업 병행
- 환경변수(문자 API 키)는 Edge Function 시크릿으로만 보관 — 프런트 노출 금지

## 11. 운영 비용 추정 (월)

| 항목 | 시작 | 성장 시 |
|---|---|---|
| Supabase | Free (500MB DB) | Pro $25 |
| Vercel | 현행 무료 유지 | 무료로 충분 |
| 문자 | 실비 (월 500건 기준 SMS ~5천원, LMS ~1.5만원) | 건수 비례 |
| **합계** | **~2만원 미만** | **~6만원 수준** |

## 12. 단계별 일정 (4~6주)

| 주차 | 작업 | 산출물 |
|---|---|---|
| 1주 | Supabase 프로젝트, 스키마 19테이블 + RLS + 채번함수 | DB 완성, 시드 |
| 2주 | Auth 로그인 교체, 저장소 어댑터로 CRUD 전환 | 다중 사용자 동작 |
| 3주 | send-sms 함수 + 알리고/솔라피 계약·발신번호 등록 | 실제 문자 발송 |
| 4주 | submit-form 공개화, convert-quote 트랜잭션, Realtime | 전체 흐름 서버화 |
| 5주 | 마이그레이션 리허설, 병행 운영, 권한 시나리오 테스트 | 체크리스트 통과 |
| 6주 | 실데이터 이전, 오픈 | 운영 전환 |

## 13. 준비물 (굿투어 측)

- 사업자등록증 사본 (문자 발신번호 등록)
- 직원 이메일 목록 (로그인 계정)
- 문자업체 선택 및 충전 (알리고 or 솔라피 권장)
- 도메인 (선택 — 현재 vercel.app 주소도 무방)
