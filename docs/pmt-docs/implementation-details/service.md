# 서비스·작업 실행 세부 계획

[상세계획 안내](README.md) · [공유 계약](shared-contracts.md) · [병렬 실행](parallel-execution.md)

기준: `docs/implementation-plan.md` 및 contracts/modules/reference-flows. 구현 전 행동 계약이며 파일 생성·기술 선정·G04/G06 통과를 뜻하지 않는다. .NET/ASP.NET Core, SQLite, 앱 SQL 투영은 추천안이다. P06-01의 IIS/접수 저장 기술은 G04, P09-05의 조회 원천·객체는 G06 사용자 결정에 남긴다. 작업 DB는 API/Worker의 유일한 업무 접점이며 API가 작업 상태를 직접 갱신하지 않는다.

공통 시험 용어: 계약 시험은 독립 fixture로 DTO·상태·오류 경계를 검증한다. 모의 코드는 결정적 fake 저장소/bridge/backend로 순서와 장애를 주입한다. 실호스트 검증은 선택된 Windows/VMM/SQL/Cloud 실체에서 별도 증거를 남긴다. 아래 각 카드의 준비→자극→기대→증거는 이 세 수준을 혼합하지 않는다. Cloud simulated v1은 strict create 전용이다. 실제 endpoint는 real만 받고, 실제 확장은 버전·양쪽 adapter·계약 시험을 함께 바꾼다.

## S01 — 수신부 기술·실행 경계

| 항목 | 계약 |
|---|---|
| 목적 | P06 초기 수신 경로의 기술·운영 경계를 결정 가능한 비교안으로 구체화한다. |
| 추가 범위 | 인증된 command 파싱·크기/버전 검사·접수 서비스 호출·응답 mapping, API/Worker 분리. |
| 수정 범위 | 기존 P06 설계에서 수신부가 하는 일을 검증·위임·응답으로 한정한다. |
| 삭제 범위 | 현재 구현 코드가 없어 실제 삭제 대상은 없다. 향후 구현에서 기존 사용자 선택·미정 게이트를 덮어쓰지 않는다. |
| goal | P06-01에서 사용자 선택이 가능한 시험 기준과 책임 경계. |
| non-goal | IIS 구현 선택의 선승인, 실제 배포, HTTP에서 VM 변경. |
| 추적 작업 | P06-01 |
| 책임 모듈 | M02 Admission, M09 Security, M03 Jobs 호출 경계, M10 진단. |
| 선행/게이트 | P00 계약·P01 점검; G04에서 IIS 기술/저장 방식 결정 전 실환경 확정 금지. |
| 코드 동작 | 수신→TLS 종단 신원 매핑→허용 경로/버전/필드/본문 크기 검증→인가→M03 접수 요청→영속 성공 시 202 또는 중복 응답→안전한 오류 변환. |
| 목표-input | version: 외부 wire 버전 / Cloud 계약 / 필수. request_id: 상관 ID / Cloud / 필수. job_id, vm_id, operation, attempt, external_idempotency_key, mode: Cloud 업무 식별·동작 / Cloud simulated strict DTO / 필수, 형식·허용 enum 준수. profile: profile_id/version/snapshot / 승인 catalog / 필수, snapshot 일치. 인증 신원: 등록 클라이언트 인증서 및 출발지 / TLS·등록부 / 필수. |
| 목표-output | 접수 결과: external_job_id·external_vm_id·queued·접수시각 / M03→호출 Cloud / 불명확 상태를 성공으로 표현 금지. 오류: 안정 code·safe message·request_id / M02→호출자 / TLS handshake 거부는 JSON 보장 불가. 준비 상태: DB·계약 호환성 / API health→운영 / 의존성 불가면 not-ready. |
| 코드동작확인 Test방식 | 준비: 유효/무효 인증서와 고정 DTO fixture, M03 fake. 자극: 정상·필드/버전 거부·중복 전달·저장 장애·TLS 거부. 기대: 유효 요청은 접수 위임, 추가 필드/ simulated real endpoint 거부, 저장 전 202 금지, handshake 거부는 앱 응답과 구분. 증거: 독립 계약 결과, 모의 HTTP 기록, 실 IIS handshake/응답 증거를 각각 분리. |
| 코드동작확인 로깅방식 | event 의미: command.accepted/rejected 및 service readiness. 상관필드: request_id, service principal 식별자, mode, outcome, error_code. 분류: 진단 JSONL·감사 원장은 별도, 접수 업무 원장은 M03. 마스킹: 본문·인증서 키·주소·헤더 비밀 제외. 검증: allowlist 검사와 거부/정상 로그 대조. |
| 실패·복구 | 접수 DB/감사 commit 실패는 503 및 재접수 가능하도록 확정 결과 없음. API 재시작 후 DB 기준 조회. 수신 취소는 이미 커밋한 업무 취소가 아니다. |
| 완료 조건 | 계약: strict DTO/오류 mapping 합격. 모의: API가 접수만 위임함. 실호스트: G04 선택 후 TLS·IIS 실행·ready 증거 별도. |
| 인계 정보 | 기술 비교·G04 미결정 항목·선택 근거·배포/계약 버전을 다음 P06/P09 작업에 전달. |

## S02 — 서비스 인증·신뢰

| 항목 | 계약 |
|---|---|
| 목적 | API↔Cloud의 상호 TLS, 등록 신원, 서비스별 권한을 분리 검증한다. |
| 추가 범위 | 인증서 체인/유효기간/용도/이름/폐기 정책 검증, 서비스 principal 매핑, 허용 동작 검사, 교체 진단. |
| 수정 범위 | 기존 호출 인증이 인증서 유효성만으로 무제한 권한을 주지 않도록 한다. |
| 삭제 범위 | 현재 구현 코드가 없어 실제 삭제 대상은 없다. 향후 JWT/HMAC 과거 인증 경로를 새로 복구하지 않는다. |
| goal | 등록 호출자와 허용 동작을 식별. |
| non-goal | 회원 로그인·JWT 발급·임의 출발지/인증서 헤더 신뢰. |
| 추적 작업 | P06-02, P06-03 |
| 책임 모듈 | M09 Security, M02 Admission, M10 진단·감사. |
| 선행/게이트 | S01·P02 네트워크, G05 실제 CA/인증서/폐기·서비스 신원 승인. |
| 코드 동작 | TLS 검증→인증서 등록 identity 매핑→고정 출발지 대조→요청 operation/대상 권한 확인→허용/거부 감사. TLS 종단과 앱 identity 경계를 함께 확인. |
| 목표-input | peer certificate: 서비스 identity 증명 / TLS handshake / 필수. 등록 신원·허용 작업·대상범위: 운영 설정/등록부 / 필수. 폐기 신선도: 승인된 배포·정책 / G05에 따른 제약. 전달 헤더·X-Forwarded-For는 신뢰 원천이 아님. |
| 목표-output | principal: 등록된 서비스 주체 / M02 인가 사용. decision: allow/deny와 안정 사유 / M02·감사 / 알 수 없는 신원 deny. readiness: 신뢰 구성 상태 / 운영 / 만료·폐기정보 불가 시 정책에 따라 not-ready 또는 호출 거부. |
| 코드동작확인 Test방식 | 준비: 인증서 체인·이름·용도·폐기·등록 mapping fixture. 자극: 정상, 불신/만료/이름 오류/폐기/미등록/허용 밖 동작, 재전송. 기대: 등록+허용만 진행; 재전송은 S03 멱등 처리; 인증 실패가 새 작업을 만들지 않음. 증거: 인증 fixture 보고서, 거부 감사 event, G05 실인증서 허용·거부 증거. |
| 코드동작확인 로깅방식 | event 의미: security.denied/identity.mapped/certificate.health. 상관필드: request_id, principal alias, outcome, reason code. 분류: 진단+감사, 비밀·원본 키는 별도 보안 보관. 마스킹: 인증서 전체·키·실제 IP 제외. 검증: 내보내기 검사와 이전 identity 거부. |
| 실패·복구 | 신뢰 불가 호출은 거부·경보. 인증서 교체는 신·구 유효 기간을 G05 승인대로 운용하고 이전 신뢰 철회를 시험; 401/403을 빠르게 무한 재시도하지 않는다. |
| 완료 조건 | 계약: identity/권한 분리. 모의: 허용·거부 조합과 감사. 실호스트: CA·폐기 전파·IIS 실제 binding·키 ACL 증거 후. |
| 인계 정보 | 주체 등록, 작업 권한 표, 인증서 lifecycle·G05 미결정 사항을 Cloud·운영 인계에 전달. |

## S03 — 영속 접수·멱등

| 항목 | 계약 |
|---|---|
| 목적 | 응답 유실·동시 재전송에도 같은 업무 시도의 논리 접수를 하나로 유지한다. |
| 추가 범위 | 정규화 본문 해시, principal+환경+키 고유성, 원자적 job/binding/reservation/audit 접수. F01은 공통 계약 정의만 소유하며 실제 멱등·저장·접수 동작의 단독 책임자는 M03/S03이다. |
| 수정 범위 | API의 성공 응답 조건을 접수 transaction commit에 연결한다. |
| 삭제 범위 | 현재 구현 코드가 없어 실제 삭제 대상은 없다. 향후 메모리 큐 단독 접수 경로를 추가하지 않는다. |
| goal | durable acceptance 및 중복 판정. |
| non-goal | 실행 완료 보장, 영구 exactly-once 외부 부작용. |
| 추적 작업 | P06-04 |
| 책임 모듈 | M03 Jobs 업무 상태 작성자, M07 Persistence, M02 응답 mapping. |
| 선행/게이트 | S01/S02, P00 계약; 저장 기술은 G04 결정 이후 구현 선택. |
| 코드 동작 | 인가 DTO 정규화(필드 의미 변경 없이)→키 조회/해시 비교→동일 해시면 기존 식별자 반환→다른 해시면 409·감사→신규면 한 transaction에 job·예약·멱등·감사 commit→그 후 202. |
| 목표-input | principal/environment: 인증 주체·실행영역 / M09·설정 / 필수. external_idempotency_key: Cloud 시도 중복키 / Cloud / 필수. canonical request: operation/profile/attempt 및 의미 필드 / 외부 DTO / 필수·버전 schema 준수. attempt는 Cloud 업무 시도이며 Worker 재시작으로 증가하지 않음. |
| 목표-output | external_job_id: Windows 접수 ID / M03→Cloud. external_vm_id: 생성 예약용 논리 ID / M03→후속 실행·Cloud. state=queued·accepted_at: commit 결과 / M03→응답. conflict: 같은 키 다른 의미 본문 / M02→409. DB 장애는 접수 불명확 시 조회 재조정. |
| 코드동작확인 Test방식 | 준비: 저장소 fixture와 독립 해시 기대값. 자극: 정상·동일 재전송/동시 경쟁·다른 본문·commit 실패·응답 유실. 기대: 하나의 job/binding, 기존 ID 재사용, 충돌 409, 실패 commit은 202 없음, 응답 유실 후 재조회 동일. 증거: DB 제약/transaction 결과, 중복 수·응답 기록, crash-restart fixture. |
| 코드동작확인 로깅방식 | event 의미: command.accepted/rejected/idempotent.replayed. 상관필드: principal alias, key hash 식별자, request/job IDs, outcome. 분류: 업무 원장+감사 필수, 진단은 별도. 마스킹: raw key/body 없음. 검증: 원장 row와 event 상관 및 로그 손실에도 접수 유지. |
| 실패·복구 | DB commit 여부 불명확하면 동일 키로 원장 조회; 신규 키로 재생성 금지. tombstone/idempotency 정보는 자동 만료하지 않는다. |
| 완료 조건 | 계약: 중복/충돌 규칙. 모의: 경합·장애 원자성. 실호스트: 선택 DB migration을 배포 단계로 적용하고 복구 시험. |
| 인계 정보 | 키 scope, canonicalization 버전, 보존 규칙, 상태 schema/migration 호환성을 Worker·Cloud에 전달. |

## S04 — Worker·bridge·불명확 실행 복구

| 항목 | 계약 |
|---|---|
| 목적 | 작업 점유, 고정 PowerShell bridge, 외부 실행 제출과 불명확 결과의 안전한 복구. |
| 추가 범위 | lease generation·heartbeat, submit intent, protocol/invocation 식별, child process 수명·stdout/stderr 처리, reconcile. |
| 수정 범위 | timeout·Worker 중단·lease 만료 뒤 즉시 재실행하지 않도록 복구 우선 상태 전이. |
| 삭제 범위 | 현재 구현 코드가 없어 실제 삭제 대상은 없다. 향후 임의 명령 조립/Invoke-Expression/요청 지정 경로·callback 실행 경로를 추가하지 않는다. |
| goal | 추적 가능한 제출·관측·인계. |
| non-goal | 프로세스 kill이 VMM Job 취소라는 보장, exactly-once 보장. |
| 추적 작업 | P06-05, P06-06, P06-07 |
| 책임 모듈 | M03 Jobs, M05 Execution/bridge, M07 저장, M04 정책, M06 재관측. |
| 선행/게이트 | S03·P05 단독 VM 흐름; Windows PowerShell 5.1 고정 entrypoint와 실제 권한은 실행 단계에서 검증. |
| 코드 동작 | 조건부 claim→running/attempt/submit_intent commit 후 transaction 종료→고정 PowerShell에 제한 stdin JSON→submitted/not_submitted/unknown 결과 저장→unknown이면 recovering→기존 process·invocation·VMM Job·VM 재조회→입증된 경우에만 진행, 입증 불가 needs_attention 및 예약 유지. |
| 목표-input | job ID/attempt: 저장 작업 / M03 / 필수. invocation_id/protocol_version: bridge 호출 구분 / M03 / 필수. operation·검증된 execution plan·허용 대상 ID: M01/M04 승인값 / 필수·catalog 일치. deadline: 동작별 정책 / 설정 / 필수·단위 명시. 사용자 script/command는 거부. |
| 목표-output | bridge outcome: submitted/not_submitted/unknown / M05→M03. vmm_job_id/hyperv_vm_id: 각 제품 식별자 / adapter 관측 후 값 또는 미확정. stage observations/errors: M03 원장. recovery disposition: resume/reconcile/needs_attention / M03. |
| 코드동작확인 Test방식 | 준비: 결정적 bridge fake, 제출 전/후 crash 지점·가짜 제품 Job/VM 상태. 자극: 정상, 거부 입력, 같은 invocation 재호출, stdout 손상/timeout/Worker 종료/lease 만료. 기대: 외부 실행 전 의도 영속화; 같은 시도 식별; unknown은 재조회만 하고 자동 생성 반복 없음; proof 없는 케이스 예약 유지. 증거: 단계 원장·bridge 입출력 hash·fake 제품 상태, 실 VMM 제출/재시작 시험 별도. |
| 코드동작확인 로깅방식 | event 의미: job.claimed, execution.submitting/submitted/unknown, recovery.started/needs_attention. 상관필드: job/request/execution_attempt/invocation/VMM IDs, stage, generation, outcome. 분류: 진단·업무 원장·감사·제품 로그 구분. 마스킹: stdin 원문/전체 명령행/경로·자격증명 제외. 검증: 로그 유실 시 원장 복원 및 원본 증거 hash 확인. |
| 실패·복구 | stale generation의 DB 쓰기는 거부. process 종료·lease 만료는 부작용 취소 증거가 아니다. 이미 제출 가능성이 있으면 추가 submit과 자동 cleanup 금지. |
| 완료 조건 | 계약: bridge envelope와 결과 구분. 모의: 모든 중단점 reconciliation. 실호스트: PS5.1 인코딩/프로세스/VMM Job 재조회·권한 검증을 별도 기록. |
| 인계 정보 | 배포 release와 호환되는 bridge protocol, 실행 권한, 복구 증거 필드, needs_attention 운영 절차를 전달. |

## S05 — 접수 제한·초기 진단·시험

| 항목 | 계약 |
|---|---|
| 목적 | 과부하·잘못된 호출·필수 저장 장애에서 안전하게 접수를 거부하고 첫 운영 진단을 제공한다. |
| 추가 범위 | 본문 크기·호출 제한·ready/live 구분·최소 진단/계약 시험·저장공간 경보. |
| 수정 범위 | readiness는 DB/계약 준비를 반영하고 liveness와 분리한다. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상은 없다. 향후 로그 정리 경로는 진단 로그만 회수하고 업무 원장·감사·outbox를 건드리지 않는다. |
| goal | 안전한 admission gate와 초기 진단. |
| non-goal | 관측 도구가 업무 완료의 동기 의존성 되는 것. |
| 추적 작업 | P06-08, P06-09 |
| 책임 모듈 | M02, M03/M07 저장 상태, M10 Observability. |
| 선행/게이트 | S01·S03, P00 진단 계약. 최소 구조화 로그·마스킹은 F03/G10 및 P06 계약으로 진행한다. 성능 수집 위치·범위만 G09 결정 이후 확장한다. |
| 코드 동작 | 제한 검사→필수 접수/감사 저장 가능성 확인→저장 불가·포화 시 503/429→liveness와 readiness 별도 응답→허용된 요약 지표·초기 계약 시험. |
| 목표-input | body size/content type/version: HTTP / M02 / 상한·allowlist. capacity/readiness: 저장소/Worker/계약 호환 / 상태 probe / 관측값. rate identity: 등록 principal·정책 / M09 / key 본문 아닌 신원 기준. |
| 목표-output | HTTP 허용/거부·Retry-After 정책 / Cloud 호출자. ready/live 상태 / 운영 점검. diagnostic event/error code / 운영자. 내부 경로·비밀은 출력하지 않고 세부 unavailable을 구분. |
| 코드동작확인 Test방식 | 준비: 제한·저장소 장애·로그 sink 장애 fixture. 자극: 정상, 초과크기/빈도, 중복, DB/full disk, JSONL sink 오류. 기대: 필수 영속 실패 시 접수 중단, 진단 sink 오류는 별도 계측/fallback, 중복은 S03 결과, health에 원인 코드. 증거: 응답·health·Event Log fallback 모의 결과, 용량 주입 실호스트 증거는 별도. |
| 코드동작확인 로깅방식 | event 의미: command.rejected, readiness.changed, diagnostic.sink_failed/drop. 상관필드: principal alias, request_id, reason, capacity category. 분류: 진단 로그·감사·작업 DB 분리. 마스킹: 주소/본문/헤더 제외. 검증: 마스킹 export와 drop counter·fallback. |
| 실패·복구 | 원장 공간 부족이면 새 변경 503; VM/미전달 outbox를 삭제해 공간 확보 금지. 로그 sink 회복 후 수동 확인하며 누락 작업을 재실행하지 않는다. |
| 완료 조건 | 계약: 제한·probe/error 규칙. 모의: 포화·sink 장애. 실호스트: 운영 ACL, Event Log source 사전 배포, 크기/회수 예산은 G01 실측. |
| 인계 정보 | 상한·용량 경보·health 원인·초기 시험 결과와 G09/G10 미확정을 운영 문서로 전달. |

## S06 — VMM 실행 adapter·수신부 이전

| 항목 | 계약 |
|---|---|
| 목적 | 단독 Hyper-V 흐름에서 검증한 실행 경계를 VMM backend로 이전하며 API 계약을 유지한다. |
| 추가 범위 | 공식 VMM 명령 adapter, VMM Job/VM 조회·ID binding, backend capability, P09 수신부 연동. |
| 수정 범위 | M05 backend 선택과 M04 정책을 분리; VMM 관리 대상은 VMM 경로로만 변경. |
| 삭제 범위 | 현재 구현 코드가 없어 제거 대상은 없다. 향후 VMM 소유 VM에 대한 직접 Hyper-V 변경 진입 경로는 비활성화·폐기하되, 단독 Hyper-V backend 자체는 해당 binding이 VMM으로 이전된 뒤에만 정리한다. |
| goal | 승인된 VMM 작업 제출·관측. |
| non-goal | VMM DB 직접 쓰기, P08 실증 전 운영 가정. |
| 추적 작업 | P09-01, P09-02, P09-03 |
| 책임 모듈 | M05 Execution, M04 정책, M06 Inventory, M02 API. |
| 선행/게이트 | P08 VMM 등록·cmdlet 실증, S04 bridge, G04 수신 기술 결정. 이전 중 G05 인증서·출발지·서비스 권한을 새 위치 기준으로 재검증한다. |
| 코드 동작 | 전환 시작을 원장에 기록→새 접수 잠금→진행 작업 drain 또는 실행상태를 명시 인계→기존 수신부·Worker 비활성화→작업/VM mapping·reservation·lease generation·미전달 outbox 보존 확인→새 위치에서 인증서·고정 출발지·서비스 권한 및 backend binding 검증→새 VMM 실행 adapter 경로에서 한 실행원만 활성→새 경로에서 제한 접수·조회·실행 확인→정상 전환 후 접수 재개. 어느 단계든 실패하면 새 실행원을 차단한 채 원인을 기록하고, 진행 작업을 대조한 뒤 단일 실행원 원칙을 유지하며 원복한다. |
| 목표-input | backend/environment binding·VMM 대상·profile plan / M03·M01/M04 / 필수, 혼합 실행 경로 금지. 전환 manifest: 기존 작업·VM mapping·reservation·lease generation·outbox·활성 실행원 상태 / M03/M07 / 전환 전 대조. 이전/새 endpoint identity·인증서·고정 출발지·서비스 권한 / G05·운영등록 / 필수 검증. operation/deadline/invocation은 원래 계약 의미 보존. |
| 목표-output | 전환 상태: 접수 잠금/drained/handed-off/verified/rolled_back 및 실행원 식별 / 운영·M03. vmm_job_id·actual VM mapping·reservation/generation / M05/M06→M03/M13. 접수 잠금 해제 증거 / 운영. 불명확 제출은 unknown으로 인계하며 자동 재제출하지 않는다. |
| 코드동작확인 Test방식 | 준비: 기존/신규 수신부·Worker 두 세트의 통제된 fake, 진행·미완료·미전달 작업, mapping/reservation manifest, 인증 설정. 자극: 잠금→drain/명시 인계→구 실행원 비활성→새 신원/출발지/권한 검증→mapping·reservation 유지→한 실행원 활성→새 경로 제한 시험; 각 경계에서 실패와 재시작 주입. 기대: 처리 중 신규 접수 없음, 진행 작업 유실/재생성 없음, mapping·lease generation 보존, 두 실행원 동시 활성 0, 실패 원복에서도 이중 실행 0. 증거: 단계별 manifest/hash·활성 실행원 기록·접수/실행 추적, 실제 IIS/Worker 이전 및 VMM VM 조회 결과를 모의 결과와 분리. |
| 코드동작확인 로깅방식 | event 의미: migration.admission_locked/drained/handed_off/old_disabled/new_verified/rolled_back, execution.submitted, vmm_job.observed. 상관필드: migration ID, 이전/신규 endpoint·instance alias, job IDs, invocation/VMM/actual VM IDs, lease generation, reservation, stage/outcome. 분류: 전환 원장·감사·진단·제품 로그·증거 manifest를 분리. 마스킹: 인증서/키·실제 IP·명령행 비밀 제외. 검증: 전환 전후 ID·예약·outbox hash 및 구/신 실행원 활성 시간 겹침 없음 확인. |
| 실패·복구 | adapter/제품 오류는 stable code로 변환. VMM Job 불명확이면 S04 복구. Hyper-V 직접 fallback 자동 수행 금지. |
| 완료 조건 | 계약: backend 결과/ID 계약. 모의: adapter 및 전체 전환 실패 지점. 실호스트: P08 승인 계정·공식 cmdlet·실제 작업 결과, 접수 잠금부터 새 경로 검증·원복 가능한 이전까지 별도 증거. 계약/fake 준비만으로 P09-02/03 완료나 기능 완료를 표시하지 않는다. |
| 인계 정보 | backend 종류·capability·VMM/실 VM ID mapping·이전 시 대조 목록을 P09/P12에 전달. |

## S07 — MSSQL 읽기 계약

| 항목 | 계약 |
|---|---|
| 목적 | Linux API가 읽을 상태 투영을 G06 선택 아래 읽기 전용으로 제공한다. |
| 추가 범위 | 승인 객체·전용 login·TLS 검증·source version/freshness 및 SQL 오류 mapping. |
| 수정 범위 | 상태 조회 원천을 앱 투영 또는 승인된 제품 계약으로 명시하고 권위 구분. |
| 삭제 범위 | 현재 구현 코드가 없어 실제 삭제 대상은 없다. 향후 VMM 내부 DB 쓰기·커스텀 테이블/trigger·임의 SQL 권한 경로를 추가하지 않는다. |
| goal | 승인 필드의 안전한 읽기. |
| non-goal | MSSQL 값만으로 누락 완료 통지를 성공 확정. |
| 추적 작업 | P09-04, P09-05, P09-06 |
| 책임 모듈 | M13 ReadModels, M07 persistence 투영 시, M09 보안, M06 원천 관측. |
| 선행/게이트 | S06, G06에서 DB·객체·projection 결정. 별도 앱 DB는 기본 비교안이지 자동 선택 아님. |
| 코드 동작 | 실제 상태/작업원장 값 검증→승인 투영을 versioned view/객체로 제공→전용 login SELECT만→TLS 서버 신원 검증→상태·source version·observed_at 노출. |
| 목표-input | 승인 객체/version: G06 결정 / 구성 / 필수. 외부/실행 ID mapping, operation/state, observation time, safe error: 원장·M06 / 승인 필드만. SQL login·TLS 신뢰: 배포 보안 / secret 참조, 읽기 전용. |
| 목표-output | row projection: ID·작업/VM 상태·원천 version·시각·오류 요약 / Linux API→화면. fresh/stale/unavailable: M13 / 실제 신선도. 미매핑·오래된 값은 명시적 unknown/stale. |
| 코드동작확인 Test방식 | 준비: schema/view fixture, 권한 계정, TLS 검증 fixture. 자극: 정상 조회, 권한 밖 객체 접근, 중복 조회, TLS/DB 오류, 오래된 원천. 기대: 승인 필드만 읽힘, 쓰기/다른 객체 거부, 신선도 보존, SQL 관측만으로 자동 완료 금지. 증거: 권한 matrix·계약 결과·실 DB query audit/인증서 시험. |
| 코드동작확인 로깅방식 | event 의미: readmodel.query/succeeded/failed/stale. 상관필드: source/version, query class, duration, outcome; 고객 식별자는 최소. 분류: 진단·감사 별도. 마스킹: SQL 본문/connection string/실주소 제외. 검증: 권한 시험·결과 필드 allowlist. |
| 실패·복구 | SQL 불가 시 unavailable, 마지막 관측을 최신인 것처럼 표시하지 않는다. 통지 누락은 outbox 재전송/명시 조정으로 복구. |
| 완료 조건 | 계약: projection과 freshness. 모의: 권한/오류. 실호스트: G06 사용자 결정 후 실제 TLS/login/object 권한 검증; VMM 내부 DB 선택도 읽기만. |
| 인계 정보 | G06 결정문, 객체/version, 권한·TLS 결과, Cloud parser 호환성을 양쪽 adapter에 전달. |

## S08 — outbox·통지·상태 정합

| 항목 | 계약 |
|---|---|
| 목적 | 최종 업무 결과와 완료 이벤트를 원자적으로 기록하고 중복·유실·역순 통신에서 상태 정합을 유지한다. |
| 추가 범위 | 불변 event, 같은 transaction의 outbox 삽입, M08 retry/보류, 상태 version/freshness 대조. |
| 수정 범위 | 통지 전달 상태와 VM 작업 상태를 분리한다. |
| 삭제 범위 | 현재 구현 코드가 없어 실제 삭제 대상은 없다. 향후 callback 응답으로 VM 작업을 재실행하거나 SQL 조회만으로 누락 통지를 성공 처리하는 경로를 추가하지 않는다. |
| goal | 최소 1회 전달·멱등 수신·정정 가능한 대조. |
| non-goal | 네트워크 exactly-once. |
| 추적 작업 | P09-07, P09-08, P09-09, P09-10 |
| 책임 모듈 | M03 결과/outbox 생성, M08 전달, M13 투영, Cloud inbox는 외부 소유. |
| 선행/게이트 | S03·S06·S07, Cloud X02 계약/ACK, 이벤트 schema 공동 동결. |
| 코드 동작 | 작업 결과 검증→최종 state/audit/outbox 한 transaction→M08가 등록 endpoint로 mTLS 전송→영속 수신 ACK만 delivered→transient 재시도, 영구/인증 오류 보류→재통지는 동일 event ID/body. |
| 목표-input | terminal result: M03의 실제 검증 결과 / succeeded·failed만, recovering 제외. event identity: event_id/job IDs/external IDs/attempt/mode / 결과 snapshot. endpoint·신뢰 / 등록 설정 / callback URL은 요청에서 받지 않음. ACK: Cloud 영속 수신 확인 / 응답 계약 / 공동 검증. |
| 목표-output | completion event / M08→Cloud / 불변·버전 호환. delivery state pending/delivered/needs_attention / M08 운영. stateVersion·observed_at/source / M13 조회. 뒤늦은 결과가 더 최신 상태를 덮지 않음. |
| 코드동작확인 Test방식 | 준비: Cloud inbox 독립 fake, event fixture, 네트워크/ACK 장애 주입. 자극: 정상, ACK 유실/중복, 동일 event 재전송, 본문 충돌, 역순/이전 attempt, TLS 오류/SQL 상태만 존재. 기대: 업무 반영 1회, 바이트상 동일 event 재전송, 충돌 격리, 이전 결과 무시, SQL만으로 자동 성공 없음. 증거: outbox row·Cloud inbox row·state version 비교, 실 Cloud X02 증거 분리. |
| 코드동작확인 로깅방식 | event 의미: completion.retry_scheduled/delivered/needs_attention, state.reconciled. 상관필드: event_id, job IDs, attempt, stateVersion, delivery count, error code. 분류: outbox 업무 기록+진단+감사 별도. 마스킹: payload 전체/고객 정보 비노출. 검증: 재전송 본문 hash와 ACK 저장 확인. |
| 실패·복구 | transient 네트워크/408/429/일시 5xx는 제한 backoff+jitter. 인증서·영구 4xx·본문 충돌은 보류 경보. 재처리는 원 event 그대로, 새 작업 없음. |
| 완료 조건 | 계약: Cloud와 strict event/ACK 합의. 모의: 중복·순서·단절. 실호스트: X02 endpoint, TLS, 영속 ACK, SQL/readmodel version 대조. |
| 인계 정보 | Cloud schema/version·ACK 정의·retry 운영·미전달 event 처리 인계를 남긴다. |

## S09 — 첫 회원 VM·SSH

| 항목 | 계약 |
|---|---|
| 목적 | Cloud의 승인된 회원 신청에서 새 VM을 한 번 생성하고 실제 SSH 준비와 결과를 분리 제공한다. |
| 추가 범위 | P14 전체의 신청 연결, profile/image/network 선택, VM 생성·관측, 게스트 초기화·공개키, 통지·상태 연결. |
| 수정 범위 | Cloud 소유권은 Cloud가 확인하고 Windows는 호출자·매핑·허용 profile을 재검증한다. |
| 삭제 범위 | 현재 구현 코드가 없어 실제 삭제 대상은 없다. 향후 비밀키 수신/저장 경로 및 simulated v1에 없는 SSH 필드를 몰래 추가하는 경로를 두지 않는다. |
| goal | 실제 회원용 VM·SSH 접근 검증. |
| non-goal | mock 결과를 실연동 완료로 표시, SSH 준비를 전원 On으로 추정. |
| 추적 작업 | P14-01, P14-02, P14-03, P14-04, P14-05, P14-06, P14-07, P14-08 |
| 책임 모듈 | M02/M03/M04/M05/M06/M08/M12/M13. Cloud는 회원·소유권·화면 소유. |
| 선행/게이트 | P09 실수신/통지 계약, P04/P05 이미지·VM 단독 시험, X01/X02/X04, G03/G08. SSH 공개키 입력 필요 시 새 외부 계약 버전 승인. |
| 코드 동작 | Cloud 신청 검증→Windows 영속 멱등 접수→승인 catalog 실행계획→Worker/M05 생성→실제 VM ID·사양·주소 관측→M12 게스트 init·공개키 주입 및 SSH 관측→최종 작업/outbox→Cloud 표시. |
| 목표-input | Cloud job/vm IDs, request/attempt/key, operation/mode/profile snapshot: Cloud strict versioned contract / 필수. image/network 결정: 서버 catalog / 사용자 임의 경로 불허. SSH 공개키: 새 계약 승인 시 public key만 / Cloud·고객 / 정책 검증. |
| 목표-output | Windows job·논리 VM·실제 Hyper-V/VMM IDs 각각 구분. actual spec/power/observed_at: M06. guest readiness/SSH endpoint·host key fingerprint: 승인 공개 정보만, key secret 없음. 오류·cleanup·event: M03/M08. |
| 코드동작확인 Test방식 | 준비: 이미지 fixture, 독립 Cloud contract mock, SSH 상태 fake. 자극: 정상, 권한/프로필 거부, 중복/응답 유실, 이미지/네트워크 실패, SSH 지연/실패. 기대: VM 한 개, 실제 관측 기반 완료, SSH 미준비는 별도 상태, 실패 자원 cleanup policy 준수, 통지와 화면 정합. 증거: 계약·모의 기록, 실 회원 흐름/고객 공개키·실제 SSH 접속은 별도 승인된 환경 증거. |
| 코드동작확인 로깅방식 | event 의미: request accepted, vm observed, guest initialization/ssh readiness, job.completed. 상관필드: Cloud/Windows IDs·attempt·invocation·actual IDs·observation time. 분류: 원장·감사·진단·SSH 증거 구분. 마스킹: 공개키 원문·실IP·사용자 정보·비밀은 최소화/제외. 검증: export 스캔과 실제 SSH 접근 결과. |
| 실패·복구 | 이미지/실행 불명확이면 S04 reconcile, 자동 두 번째 생성 금지. SSH 실패는 VM 생성 실패와 별도 분류; G08에 따라 보존/정리. |
| 완료 조건 | 계약: 실제 전용 version에 필요한 SSH 입력 포함 여부 명시. 모의: 전체 실패 분기. 실호스트: Cloud 소유권·실제 VM 사양·고객 단말 SSH·통지/조회 일치 검증. |
| 인계 정보 | 이미지 hash·profile version·주소/host-key 공개 정책·실제/모의 상태·Cloud 화면 필드를 전달. |

## S10 — 전원 정책·시작/중지/재시작

| 항목 | 계약 |
|---|---|
| 목적 | VM별 직렬 변경 아래 전원 상태 전이와 완료 판정을 명확히 한다. |
| 추가 범위 | 상태×동작 정책, 시작, 정상 중지/시간초과, 재시작·재접근 관측. |
| 수정 범위 | 기존 create 전용 실제 계약을 새 version의 수명주기 명령으로 확장한다. |
| 삭제 범위 | 현재 구현 코드가 없어 실제 삭제 대상은 없다. 향후 정책 없는 강제 전원 차단·Running만으로 재시작 완료 판정 경로를 추가하지 않는다. |
| goal | 승인 정책과 실제 상태 확인. |
| non-goal | 초 단위 서비스 보장·미승인 force 옵션. |
| 추적 작업 | P15-01, P15-02, P15-03, P15-04 |
| 책임 모듈 | M03 Jobs/lease, M04 정책, M05 backend, M06 관측, M08 통지. |
| 선행/게이트 | P14 생성/접근 baseline, G08 전원/정상종료/강제 동작 정책, 외부 lifecycle 계약 승인. |
| 코드 동작 | 요청 인가→VM lease→현재 상태 확인→허용성 판정→제품 명령→실제 전원/게스트 재관측→결과·통지. 재시작은 부팅 시각/접근 복구 확인. |
| 목표-input | operation start/stop/restart, external_vm_id, attempt/key, version/profile identity: 승인 Cloud 계약 / 필수. current observation: M06 / fresh 여부 포함. graceful deadline/force policy: G08 / 승인 없으면 강제 동작 거부. |
| 목표-output | job transition, raw+normalized power state, observed_at, guest reachability/boot evidence, safe error, state version. 호출자에게 pending/unknown 분리. |
| 코드동작확인 Test방식 | 준비: 상태×동작 truth table과 backend fake. 자극: 정상·금지 상태·중복·응답 유실·제품 timeout·늦은 관측. 기대: 이미 실행 중 start 중복 기동 금지, stop 실패에서 force 금지, restart는 재접근/새 부팅 근거, 동일 VM 동시 작업 거부. 증거: table 결과·원장·제품 관측, 실 VM 전후 상태 별도. |
| 코드동작확인 로깅방식 | event 의미: vm.operation.accepted/denied, vm.power.observed, job.completed. 상관필드: job/external VM/actual ID/operation/state version. 분류: 감사·원장·제품 로그·진단 분리. 마스킹: 실제 주소/게스트 자격증명 금지. 검증: 상태 전이와 관측값 대조. |
| 실패·복구 | timeout은 재관측/unknown; 제품 실행 여부 확인 전 명령 재실행 금지. lease는 VM별 충돌을 차단. |
| 완료 조건 | 계약: versioned lifecycle schema. 모의: 각 정책 분기. 실호스트: G08 승인 정책으로 실제 시작·중지·재시작 및 재접근을 각각 증명. |
| 인계 정보 | 상태표·deadline 측정값·force 승인 유무·backend별 capability를 Cloud와 운영자에 전달. |

## S11 — 삭제·보존·해제

| 항목 | 계약 |
|---|---|
| 목적 | 논리 삭제, VM 등록 제거, 디스크 보존/제거, 주소 해제와 tombstone을 안전하게 분리한다. |
| 추가 범위 | 소유/실행/참조 검증, 삭제 정책, 잔여 자원 기록, 멱등 반복 처리. |
| 수정 범위 | delete는 하나의 모호한 파괴 명령이 아니라 검증된 단계와 보존 결과로 정의. |
| 삭제 범위 | 현재 구현 코드가 없어 실제 삭제 대상은 없다. 향후 공유 이미지·참조 중 디스크를 무조건 삭제하는 경로를 추가하지 않는다. |
| goal | 승인 범위에서 복구 가능한 삭제·주소 해제. |
| non-goal | VM 제거가 곧 파일 영구 삭제라는 가정. |
| 추적 작업 | P15-05, P15-06 |
| 책임 모듈 | M03 작업 원장, M04 lifecycle 정책/binding, M05 제품 제거, M06 검증, M08 통지. |
| 선행/게이트 | S10, G08 보존·잔여 자원·삭제 정책, 주소 소유자/해제 계약. |
| 코드 동작 | 인가·대상 binding→진행 작업/디스크 공유/보존 검사→삭제 단계를 제출→실제 VM 등록/디스크 참조 재확인→정책상 주소 해제→tombstone·audit·outbox 원자 기록. |
| 목표-input | external_vm_id 및 매핑, delete operation/attempt/idempotency, ownership principal, preserve policy: Cloud 계약/Cloud identity/G08 / 필수. 주소 lease identity·디스크 참조는 내부 inventory. |
| 목표-output | tombstone/삭제 단계·실제 VM 존재 여부·디스크/주소 잔여 자원·복구 가능성·state version. 불명확 상태는 unknown; repeated delete는 기존 결과. |
| 코드동작확인 Test방식 | 준비: 공유/비공유 디스크·진행 job·주소 lease fixture. 자극: 정상 삭제, 권한 거부, 중복, 제품 오류/중단, 공유 참조/진행 충돌. 기대: 확인된 대상만 제거, 공유 자산 보호, 주소는 정책 시점에만 해제, unknown 보존·재생성 금지. 증거: binding/reference 전후·tombstone·실 자원 목록; 실호스트 보존/복구 시험 분리. |
| 코드동작확인 로깅방식 | event 의미: vm.delete.started/completed, cleanup.required, address.released. 상관필드: job/VM IDs, 단계, policy, resource reference IDs. 분류: 원장·감사 필수, 진단/제품 로그/증거 별도. 마스킹: UNC/실 주소·개인정보 제한. 검증: 잔여 자원 manifest와 실제 대조. |
| 실패·복구 | 삭제 중단 후 actual resource 재조회, 정책상 보존, 필요 시 manual_required. cleanup 실패는 원 오류와 별도로 보존. 자동 삭제 반복 금지. |
| 완료 조건 | 계약: 삭제/보존 결과. 모의: 공유참조·중단·중복. 실호스트: G08 서면 선택별 VM 제거·디스크 정책·주소 재사용·복구 가능 범위 검증. |
| 인계 정보 | tombstone 참조·보존/회수 권한·주소 lease 결과·잔여 자원 및 복구 절차를 Cloud/운영에 전달. |

## S12 — CPU/RAM/디스크/네트워크 변경·충돌

| 항목 | 계약 |
|---|---|
| 목적 | 각 변경 종류에 필요한 사전 상태·단계·검증을 별도로 정의하고 VM별 충돌을 막는다. |
| 추가 범위 | CPU/RAM 변경, 디스크 확장 및 게스트 확장, 네트워크 주소/연결 전환·복구, 전체 충돌 matrix. |
| 수정 범위 | 자원 변경은 단일 generic resize 완료가 아니라 operation별 검증 결과로 확장. |
| 삭제 범위 | 현재 구현 코드가 없어 실제 삭제 대상은 없다. 향후 별도 검증 전 디스크 축소·정책 없는 네트워크 단절 변경 경로를 추가하지 않는다. |
| goal | 승인 크기/네트워크로 안전하게 수렴. |
| non-goal | 임의 크기/네트워크 입력, OS 게스트 완료 추정. |
| 추적 작업 | P15-07, P15-08, P15-09, P15-10 |
| 책임 모듈 | M03 serialize/lease, M04 policy, M05 product, M06 actual/guest observation, M08 event. |
| 선행/게이트 | S10/S11, P02-06 네트워크/address 계약, G08 크기·정지·디스크·원복 정책, 승인 Cloud command schema. |
| 코드 동작 | 동작별 lease→현재 전원/참조/용량/주소 관측→catalog·quota 정책 확인→승인 단계 실행→제품/게스트/SSH 재관측→state/outbox 기록. CPU/RAM은 정지 상태 기본안; 디스크 확장과 guest filesystem 확장은 분리; 네트워크 변경은 이전/신규 주소·단절·재접속·원복 평가. |
| 목표-input | 공통: operation key/attempt·VM mapping·freshness가 있는 현재 관측·승인 catalog target. CPU/RAM: 정지 상태 여부·허용 vCPU/메모리 범위 / M06+catalog+G08. 디스크: 기존 가상 디스크 식별·확장 목표·게스트 볼륨 mapping / M06·승인 profile; 축소 요청은 별도 지원 결정 전 거부. 네트워크: network profile·기존/new address lease·게스트 설정 정책 / catalog·승인 네트워크 서비스·G03; 임의 주소 입력 금지. |
| 목표-output | CPU/RAM: 제품 재조회값·전원/재기동 상태. 디스크: VHDX 용량과 guest partition/filesystem 사용 가능 용량을 독립 결과로 출력. 네트워크: NIC/profile mapping·새/이전 lease 상태·SSH 재접속 관측. 공통으로 stage result/error/cleanup/state version; 미관측은 pending/unknown. |
| 코드동작확인 Test방식 | 준비: 각 하위 동작의 단위·상태표, guest/network fake, VM lease fixture. CPU/RAM 자극: 허용·범위초과·실행중 상태·중복·재기동 지연; 기대: 승인 범위 적용, 정지 상태 정책 준수, 실제 재조회값 일치. 디스크 자극: VHDX 확장 후 guest 확장 성공/실패·축소 요청; 기대: 두 단계 결과 분리, 축소 거부, guest 실패를 성공으로 뭉개지 않음. 네트워크 자극: 주소 충돌·설정 단절·재접속 실패·복구; 기대: 신규 예약, 재접속 증거 전 완료 금지, 정책상 원복 또는 manual_required. 공통 자극: 응답 유실/제품 timeout 및 start-delete·change-change·restart-stop 경합; 기대: VM별 한 작업·unknown 재조회·중복 실행 금지. 증거: 하위 동작별 독립 fixture와 lease 경쟁, 실호스트 CPU/RAM·VHDX+guest 용량·주소/SSH 증거를 각각 구분. |
| 코드동작확인 로깅방식 | CPU/RAM: before/target/observed spec와 power state. 디스크: VHDX 단계 및 guest 확장 단계별 outcome/용량. 네트워크: 이전/new lease ID·연결 단계·재접속/원복 상태. 공통 event는 vm.resize.stage, disk.guest_extension, network.change/recovered, operation.conflict; 상관필드는 job/VM IDs, resource kind, state version, stage/outcome. 분류: 감사·작업 원장·진단·제품/게스트 증거 분리. 마스킹: 실제 IP/경로/계정정보 제한, 공개본 제거. 검증: allowlist export 및 동작별 관측 대조. |
| 실패·복구 | 외부 부작용 후 timeout은 상태 재조회. 네트워크 단절은 사전 정의된 원복만 실행; 원복 불확실 시 needs_attention. lease가 제품 작업을 종료하지 않으므로 새 변경 차단. |
| 완료 조건 | 계약: operation-specific schema/results. 모의: 모든 충돌 및 부분 실패. 실호스트: G08/G03 정책 승인 후 CPU/RAM·VHDX/guest·주소/SSH 각 증거를 별도 확보. |
| 인계 정보 | 승인 자원 범위·각 단계 완료 기준·lease 충돌 결과·주소 회수/복구 정보를 Cloud UI와 운영자에게 전달. |

## 통합 경계·착수 순서

공유 기반·운영 카드는 root 계획의 F01–F10 및 O01–O14에 둔다. 이 문서는 서비스 카드만 상세화하며 cross dependency는 위 추적 P ID를 기준으로 한다. 구현 병렬화는 계약/fixture가 먼저 고정된 뒤 가능하다: S01–S03 기반 접수, S04 병행 bridge 계약 후 Worker, S05 진단; P08 이후 S06, S07은 G06 게이트, S08은 Cloud X02 계약, S09는 앞선 실연동·이미지, S10–S12는 P14 및 G08 이후로 연결한다. G04/G06 선택을 기다리는 동안 독립된 계약 시험·fake adapter·비교 증거만 준비하고 선택 의존 구현을 독자 확정하지 않는다. 각 카드의 전체 완료 판정은 선행 P 작업의 필수 실호스트·제품·Cloud 증거까지 확보된 뒤에만 가능하다. 호스트가 없는 단계에서는 계약/정적 검토/fake 모의의 제한된 준비 상태만 기록하며 VM 기능 완료나 실호스트 검증 완료로 표시하지 않는다.
