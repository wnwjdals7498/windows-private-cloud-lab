# 기반·호스트 기능 구현 상세

[상세계획 안내](README.md) · [공유 계약](shared-contracts.md) · [병렬 실행](parallel-execution.md)

작성: 2026-10-02 · 상태: 병렬 구현 준비 계약 · 코드/실호스트 수행 전

이 문서는 `docs/implementation-plan.md`의 P00~P08 가운데 기반·호스트 기능을 책임지는 F01~F10의 행동 계약이다. 계획의 선행 관계와 G01~G10을 대체하지 않는다. 각 카드는 구현 준비와 실제 호스트 실행 가능 판정을 분리한다. 모의 검증 성공은 제품 실동작 증거가 아니다.

공통 원칙: Linux/Cloud 소유권·회원 권한은 Windows가 대신하지 않는다. Cloud mock은 현재 허용 필드·모드를 엄격히 지키며 실제 계약 확장은 양쪽 버전·adapter·시험을 함께 바꾼다. 외부 식별자, Windows job ID, VMM job ID, Hyper-V VM GUID는 서로 다른 의미다. 중첩 Hyper-V Compute 클러스터 실습은 제품 운영 지원 주장으로 바꾸지 않는다. 두 8대 실습을 겹쳐 가동하지 않는다. 상호 TLS, G04 IIS 기술 선택, G06 SQL 조회 원천에 대한 사용자 결정권을 보존한다.

## F01 — 공통 식별자·명령·오류 계약

| 항목 | 계약 |
|---|---|
| 목적 | 모든 계층에서 요청·업무·실행·제품 자원의 식별과 거부/불명확 상태를 혼동 없이 연결한다. |
| 추가 범위 | 버전 있는 명령·응답·오류 의미, ID 경계, 시각·null·상태 의미, 멱등 충돌 판정. |
| 수정 범위 | 외부 mock 호환 정책과 내부 ID 매핑의 관찰 가능한 규칙을 명확히 한다. |
| 삭제 범위 | 현재 삭제 없음(저장소에 실행 코드가 없음). 향후 금지 경로는 PHP 직접 호출, JWT/HMAC 복원이다. 최신 승인 계약은 Linux API와 상호 TLS다. |
| goal | 공통 의미·strict schema·오류 mapping·독립 contract fixture를 정의해 각 실행 소유자가 같은 의미를 구현하도록 한다. |
| non-goal | DB 멱등/접수 구현(S03), Hyper-V 실행·제품 재조회(F08), API/Worker 영속 처리(S04), Cloud 회원 인증·소유권, Cloud mock 필드 추가, 네트워크 호출 구현. |
| 추적 작업 | P00-01, P00-02, P00-05 |
| 책임 모듈 | M02 Admission, M03 Jobs, M04 VmLifecycle, M05 Execution, M06 Inventory, M09 Security. 저장 원자성은 M07, 전송은 M08의 계약을 따른다. |
| 선행/게이트 | PMT 최신 요구사항 정리. 구현 가능성은 계약 fixture로 먼저 판단. 실호스트 호출은 P00-08 이후 관련 P 단계와 G 게이트 통과 후만 가능. |
| 코드 동작(단계) | (1) wire schema version·허용 필드·mode의 공통 의미 정의 (2) 요청/업무/VM/제품 ID 의미와 오류 코드·stage·retryable·safe message mapping 정의 (3) canonical 의미 정규화 규칙과 동일/충돌 본문 판정 fixture 정의 (4) 독립 fixture로 정상·거부·충돌·unknown 표현 검증 (5) DB 접수/멱등, 실제 제품 재조회, API 응답/상태 전이는 해당 실행 소유 카드가 구현하고 본 계약의 의미를 준수한다. timeout은 미관측/unknown이지 실패 확정이 아니다. |
| 목표-input | 외부 wire `version`: 계약 버전, Cloud 계약 원천, 필수. `request_id`: 요청 상관값, 필수. `job_id`: Cloud 업무 작업 식별, 필수. `vm_id`: Cloud가 예약한 논리 VM 식별, 현재 create 계약에서는 필수. 실제 Hyper-V VM 존재와 별개이며 필수성 변경은 새 계약 버전으로만 가능. `operation`: 허용 동작, 필수/enum 제한. `attempt`: 업무 시도, 필수/범위 제한. `external_idempotency_key`: 같은 시도 재전송 구분, 필수/신원·환경과 결합하는 의미. `mode`: 실행 종류, 필수/실 endpoint는 real만 허용. `profile` 식별·버전·snapshot: 승인 catalog 원천과 일치해야 함. 내부 `contract_version` 또는 `idempotency_key` 별칭을 두는 구현은 wire 필드가 아닌 내부 의미명이며 strict 외부 schema에 추가하지 않는다. `correlation_id`는 내부 추적 보조 의미다. |
| 목표-output | 아래는 공통 계약의 필드 의미와 런타임 소유자를 지정한다. 이 카드가 실제 값을 생산하거나 DB write를 구현하지 않는다. `external_job_id`: 영속 접수 ID, S03 생산/운영·Cloud 소비. `external_vm_id`: 안정 논리 자원 ID, S03/F08 생산 또는 mapping/Cloud 소비, 아직 실 VM 아님. `hyperv_vm_id`: 실제 GUID, F08 제품 조회 후 생산/내부 소비, 미확인은 null. `vmm_job_id`: VMM 작업 ID, VMM adapter 생산/복구기 소비, 미제출·불명확 구분. `state/state_version`: S03/S04 생산/Cloud 상태 소비, unknown은 terminal 아님. `observed_at/source/freshness`: F08/S04 관측 소유자가 생산/상태 조회 소비, 미관측은 unavailable/stale. `error.code/stage/retryable/safe_message/correlation_id`: 공통 오류 mapper 계약, 내부 상세는 노출하지 않음. |
| 코드동작확인 Test방식 | 정상: 기존 wire fixture와 유효 profile 준비→schema/정규화/오류 mapper 적용→독립 기대값과 계약 의미 일치→fixture 결과 확인. 거부: 잘못된 version/mode/불허 필드/잘못된 ID 형식 fixture→contract validator 적용→정의한 거부 코드·필드 분류→응답 fixture 확인. 중복: 동일/상이 본문 fixture 준비→canonical 의미 동등성·충돌 결과만 계산→같은 본문/다른 본문 판정 일치→정규화 fixture 확인(실제 접수·동시성 검증은 S03). 실패·복구: 불명확·관측 불가·복구 필요 결과 fixture 준비→오류/state mapping→unknown을 terminal 실패로 매핑하지 않음→독립 기대 fixture 확인(실제 제품 재조회는 F08/S04). |
| 코드동작확인 로깅방식 | event는 `command.accepted/rejected`, `execution.unknown`, `vm.observed` 의미로 고정. request/job/external_job/attempt/invocation 및 필요 시 제품 ID로 상관. JSONL 진단, DB 작업·감사 원장, 증거 bundle을 분리. 키 원문·인증서 개인키·공개키 본문·전체 payload는 마스킹/제외하고 해시만 제한 저장. 검증은 fixture, 원장과 실제 자원 매핑 대조, 공개본 비밀 스캔. |
| 실패·복구 | schema/인증/권한/키 충돌은 영속 VM 변경 없이 거부. 접수 저장 불능은 성공 접수 응답 금지. 외부 실행 결과가 불명확하면 recovering/needs_attention으로 보존하고 제품 조회 전 재실행 금지. |
| 완료 조건 | 계약: ID·오류·상태·엄격 필드 규칙 문서화. 모의 코드 검증: 독립 fixture에서 직렬화·중복·거부·오류 mapping 통과. 실호스트: 실제 Cloud/제품 ID mapping과 TLS는 관련 G05 및 P06/P08/P09 통과 전 미검증. |
| 인계 정보 | 다음 F02/F03 및 서비스/API 카드에 필드 의미·strict mock 제한·ID 대응표·미확인 정책을 넘긴다. P00 요구사항 연결 결과를 첨부한다. |

## F02 — 설정·catalog·모드 프로필

| 항목 | 계약 |
|---|---|
| 목적 | 환경값과 허용 catalog를 비밀정보에서 분리하고 모드별 가동·변경 범위를 고정한다. |
| 추가 범위 | schema 검증, 설정 출처 우선순위, 역할·제품·경로·프로필·네트워크·허용 작업 등록, 단독/인프라8/Linux8/서비스4+4 상호배제. |
| 수정 범위 | 요청 자유값을 서버가 승인한 catalog snapshot으로 해석하며 호스트 설정을 요청이 덮어쓰지 못하게 한다. |
| 삭제 범위 | 현재 삭제 없음(실행 코드가 없음). 향후 금지 경로는 비밀 기본값과 요청 본문 기반 backend/DB/callback 선택이다. 이유: 고정 구성과 신원 경계를 보존한다. |
| goal | 누락·모순·미지원 설정은 readiness/사전검증 실패로 분명히 드러나며 유효 profile은 불변 실행 계획으로 해석된다. |
| non-goal | 실호스트 자원 수치 확정, G01/G02/G03 게이트 대체, 비밀 저장소 선정, 고객별 임의 사양 허용. |
| 추적 작업 | P00-03, P00-04 |
| 책임 모듈 | M01 Configuration/Catalog, M03의 profile snapshot 사용, M11 운영모드 상태. |
| 선행/게이트 | F01의 ID 의미. P00-03은 문서·schema 모의 검증 가능. 실호스트 profile 활성화는 P01-07/G01/G02와 P02-03/G03의 판정에 종속. |
| 코드 동작(단계) | (1) 기본값→역할/환경 제한 파일→승인 override 순서로 읽기 (2) schema·지원 버전·참조 경로·신원·상호배제 검사 (3) secret은 승인된 참조만 해석 (4) catalog와 설정 버전 고정 (5) 요청 profile ID/version/snapshot을 등록값과 대조 (6) 검증된 immutable plan만 M03/M05에 전달 (7) 변경/점검 모드 권한 분리. |
| 목표-input | `environment_id`: 격리 환경 식별, 운영 등록값, 필수. `schema_version`: 설정 구조 버전, schema 원천, 필수/지원 버전만. `mode_profile_id`: 가동 조합 식별, 운영 catalog 원천, 필수/등록 enum. `role_catalog`: VM 역할·설치 조합 의미, 승인 환경 등록, 역할별 필수. `product_catalog_version`: 버전·에디션 호환 기준, 공식 근거와 G02 확인 원천, 미확정은 candidate로 표시. `path_roots`: 허용 저장 루트 의미, G01 환경 원천, 필수/정규화 절대 경로 및 루트 밖 거부. `network_profile_id`: 승인 네트워크 참조, G03 원천, 등록값만. `operation_allowlist`: 허용 작업 집합, policy 원천, 비어 있거나 알 수 없는 항목 거부. `secret_refs`: 비밀 참조 식별, 승인 비밀 저장 원천, 비밀 본문 금지. `profile_id/version/snapshot`: catalog 원천, 요청과 일치해야 함. |
| 목표-output | `validated_config_version`: 검증 설정 버전, M01 생산/readiness·운영 인계 소비. `catalog_snapshot`: 해석 시점의 불변 허용값, M01 생산/M03 소비. `execution_plan`: 승인된 자원·경로·backend·작업 정보, M01 생산/M04/M05 소비, 입력이 불명확하면 생성 불가. `mode_readiness`: 충돌·필수 의존성·가동 허용 판정, M11/M01 생산/운영 소비, unknown dependency는 blocked. `validation_errors`: 필드 경로·안정 코드·안전 메시지, M01 생산/관리 소비, 전체 secret 미포함. |
| 코드동작확인 Test방식 | 정상: 비밀 없는 등록 profile fixture→load/resolve→동일 snapshot과 허용 plan→schema 검증 결과. 거부: 필수값 누락, root 이탈, real/mock 혼합, 비지원 schema 준비→load→readiness 실패/변경 금지→오류 목록 확인. 중복: 같은 설정 소스 재로딩→동일 canonical version/plan→해시 비교. 실패·복구: 참조 비밀 접근 불가/부분 파일 손상 준비→readiness 실패 후 정상 설정 복원→비밀 fallback 없이 재개→진단과 버전 이력 확인. |
| 코드동작확인 로깅방식 | `configuration.validated/rejected`, `mode.transition.blocked/ready`는 의미 중심. environment/mode/schema/catalog version와 오류 코드만 기록. 설정 원본·secret ref 상세·주소/실제 경로는 접근제한 진단/비공개 증거로 분류하고 공개본에서 마스킹. 재검증은 독립 schema fixture와 설정 hash 비교. |
| 실패·복구 | 시작/ready 시 필수 설정 오류면 접수 차단. 설정 수정 후 새 버전으로 검증하며 이미 진행 중인 job은 기존 snapshot을 보존한다. 모드 전환 불명확 시 어느 모드도 추가 기동하지 않고 수동 대조를 요구한다. |
| 완료 조건 | 계약: schema·우선순위·catalog·모드 상호배제 명세 완료. 모의 코드 검증: 잘못된 설정 거부와 불변 plan 검증 완료. 실호스트: 값은 G01/G02/G03 승인·측정 전 미확정. |
| 인계 정보 | F04/F05에 profile·모드 제한을, F06~F10에 고정된 역할/제품/경로 allowlist를 전달한다. G01~G03 미통과 상태를 함께 표기한다. |

## F03 — 실행·증거 프로토콜

| 항목 | 계약 |
|---|---|
| 목적 | 읽기 점검, 승인 변경, 실측 결과 대조를 분리하고 모든 작업을 재현 가능한 증거와 연결한다. |
| 추가 범위 | 실행 전 조건, 단계 기록, 중단/복구 방식, 수동·모의·실호스트 수준 표기, 증거 분류·마스킹. |
| 수정 범위 | 실행 결과는 명령 성공 여부가 아니라 상태 재조회와 증거 수준을 포함한다. |
| 삭제 범위 | 현재 삭제 없음(실행 코드가 없음). 향후 금지할 판정 방식은 콘솔 출력만으로 완료 선언하는 것이다. 로그/명령 성공은 실제 VM 상태·데이터 일치를 입증하지 못한다. |
| goal | 점검은 무변경, 변경은 명시된 단계/승인 대상만 수행, 결과와 복구 가능성이 검증 가능하다. |
| non-goal | 현재 실호스트 실행, 비공개 증거의 공개, 자동화 테스트 확대 자체. |
| 추적 작업 | P00-06, P00-07, P00-08 |
| 책임 모듈 | M10 Observability/Evidence, M03 작업 단계, M05 실행 결과, M06 관측. |
| 선행/게이트 | P00-03과 F01 오류·식별 계약. G10 증거 공개 형식 결정 전에도 비공개 원본/마스킹 초안 계약은 만들 수 있으나 공개본 승인으로 간주하지 않는다. |
| 코드 동작(단계) | (1) 시나리오·대상·수준·사전조건 선언 (2) read-only inspect 또는 변경 실행 권한 확인 (3) 실행 전 상태/용량/소유권 증거 수집 (4) 단계마다 invocation·결과·오류·부작용 기록 (5) 변경 후 독립 조회 (6) 불일치/중단 시 복구 분류와 잔여 목록 기록 (7) 원본 evidence manifest와 마스킹 요약을 연결. |
| 목표-input | `run_id`: 검증 실행 식별, 생성기 원천, 필수. `scenario_id`: 검증할 계획 작업 식별, P plan 원천, 필수. `verification_level`: contract/mock/host 의미, 실행 명세 원천, 필수. `target_ref`: 승인 대상 식별자, inventory/config 원천, 변경 모드에서 필수. `preconditions`: 버전·상태·가용성 조건, 관련 P/G 원천, 필수. `action_mode`: inspect/change 구분, 운영자 명시, 필수. `evidence_policy`: 공개/비공개 분류와 마스킹 규칙, G10 원천, 필수. `expected_state`: 독립 기대값, 계약/제품 조회 원천, 필수. |
| 목표-output | `step_results`: 단계·시작/종료·outcome·오류, 실행기 생산/리뷰어 소비, 미실행 명시. `observed_state`: 실제 재조회 결과·시각·출처, M06/제품 생산/검증 소비, 미관측 명시. `evidence_manifest`: run/요구사항/해시/수준/마스킹 관계, M10 생산/인계 소비. `recovery_status`: none/pending/completed/manual_required 의미, M03/M10 생산/운영 소비. `verification_claim`: contract/mock/실호스트와 통과/실패/미검증, M10 생산/완료 판정 소비. |
| 코드동작확인 Test방식 | 정상: 격리 fixture의 read-only 조사→대상 hash 불변, 기대 상태 대조→manifest와 독립 결과 확인. 거부: inspect 경로에 변경 유도 또는 승인 외 target 준비→실행→변경 없음/명시 거부→전후 snapshot 확인. 중복: 같은 run ID 입력 또는 동일 evidence 재수집→중복이 별도 실행으로 오인되지 않음→manifest 고유성 확인. 실패·복구: 중간 단계 오류/로그 저장 장애 준비→안전 중단·잔여 확인·복구 구분→원장과 evidence의 단계·실패 위치 대조. |
| 코드동작확인 로깅방식 | event는 `inspection.completed`, `change.started`, `change.step.completed/failed`, `evidence.redacted` 의미. run/scenario/target ref/step/invocation/verification level 상관. 진단 JSONL은 순환 가능, 작업·감사 원장은 별도 영속, 원본 증거는 ACL 적용 runtime 영역, 마스킹 요약만 공유. 명령 전체/비밀/실제 IP는 제외 또는 가림. manifest hash·독립 재실행·공개본 스캔으로 검증. |
| 실패·복구 | 원장 쓰기 불가 시 신규 변경 금지. 진단 로그만 실패하면 진행 중 작업을 중복 실행하지 않고 원장/제품 상태 대조. 실제 상태 불일치는 성공 선언하지 않고 manual_required/needs_attention. |
| 완료 조건 | 계약: 증거 수준·분류·사전/사후/복구 형식 완성. 모의 코드 검증: 무변경 점검·실패 기록·마스킹 검증. 실호스트: 각 관련 단계에서 G/선행조건 통과 후 별도 증거가 있어야만 인정. |
| 인계 정보 | 전 카드와 이후 작업자는 입력 대상·사전조건·명령/버전·관측·잔여·증거수준을 제공한다. G10 결정이 없으면 공개 형식은 미확정으로 전달한다. |

## F04 — 호스트·제품·자원 판정

| 항목 | 계약 |
|---|---|
| 목적 | 별도 실습 호스트에서 OS/제품/자원 가능성을 읽고 모드별 시작·중단 기준을 근거 있게 판정한다. |
| 추가 범위 | Windows/CPU/가상화 상태, RAM/디스크/IO, NIC/기존 네트워크, 제품 지원·설치 조합, 순간부하와 8대/4+4/전환 예산. |
| 수정 범위 | 임의 64GB 배정을 실행값으로 취급하지 않고 측정값·공식 지원 정보·사용자 결정을 구분한다. |
| 삭제 범위 | 현재 삭제 없음(실행 코드가 없음). 향후 이 카드에서 제거할 행동은 기존 Hyper-V/NAT/DHCP/DNS/방화벽 변경이다. 이 단계는 판정 전용이며 변경 책임은 P02다. |
| goal | 모드별 가능/불가/조건부와 여유·시작/중단 기준을 증거로 제시한다. |
| non-goal | 실제 VM 구축, 제품 설치, 지원되지 않는 조합을 지원 구성으로 표현, G02를 문서만으로 확정. |
| 추적 작업 | P01-01, P01-02, P01-03, P01-04, P01-05, P01-06, P01-07 |
| 책임 모듈 | M01 환경·제품 catalog, M10 측정 증거, M11 host/mode controller의 읽기 전용 사전조건. |
| 선행/게이트 | P00-08, G01(별도 PC 접근·자원), G02(OS/VMM/SQL/ADK 조합). 미통과면 실호스트 판정은 차단. |
| 코드 동작(단계) | (1) 읽기 전용 host 조사 (2) OS·CPU 기능·Hyper-V와 장치/네트워크 상태 수집 (3) 실제 가용 RAM·디스크·IO baseline 측정 (4) 공식 문서와 설치물/권한 버전 대조 (5) 중첩 자원 중복합산 없이 모드별 예산 계산 (6) 이미지 복사·동시부팅·복구 순간 부하 계획 (7) 부족·충돌·미검증을 분리해 실행 프로필 판정. |
| 목표-input | `host_identity`: 대상 실습 PC 식별, 운영 등록 원천, 필수/개발 PC와 구분. `os_build/edition`: 설치 상태, read-only 수집, 필수. `cpu_virtualization_features`: 중첩 조건 근거, 호스트 수집, 필수/누락은 불가 판정. `memory_available/baseline`: 실측 용량과 상시 사용, 성능 수집, 필수/시각 포함. `disk_capacity/filesystem/io`: volume별 가용·형식·지연, 실측 원천, VM/이관 복사 여유 포함. `network_inventory`: NIC·switch·NAT·DHCP·DNS·firewall 현황, read-only 수집, 필수. `product_matrix`: OS/VMM/SQL/ADK/runtime 버전·에디션·권한, 공식 자료+G02, 미확정 표시. `mode_profiles`: 인프라8/Linux8/서비스4+4/전환 수요, P00-04, 필수. `load_scenarios`: 부팅/복사/복구 부하와 한계, 측정계획, 필수. |
| 목표-output | `host_findings`: 수집값·수집 시각·출처·누락, M10 생산/후속 계획 소비. `support_matrix`: candidate/verified/unsupported distinction, M01 생산/설치 작업 소비, 공식 근거 필요. `resource_budget`: mode/실제 예약·여유·순간부하 산정, M01/M10 생산/M11 gate 소비. `feasibility`: pass/conditional/blocked와 이유, M11 생산/계획 소비, unknown은 blocked. `change_boundary`: 후속 P02가 승인 전 적용할 수 없는 항목, 증거 산출. |
| 코드동작확인 Test방식 | 정상: 모의 호스트 snapshot과 계산 기준 준비→read-only collect/estimate→불변 snapshot·예산/공식 matrix 일치→수집 manifest 확인. 거부: 잘못된 target/권한 불충분/비공식 버전 근거 준비→판정 실행→blocked 또는 unknown, 어떤 설정도 쓰지 않음→전후 snapshot 확인. 중복: 같은 측정 창 재수집→같은 원천값은 같은 산정, 시각만 새로 기록→입력 hash·계산 재현성 확인. 실패·복구: sensor 일부 실패/측정 중 host 자원 변동→결측 표기, 승인된 재측정·임계치 재계산→원래 값 덮어쓰지 않음→원본 측정과 재측정 관계 확인. |
| 코드동작확인 로깅방식 | `host.inventory.collected`, `resource.sampled`, `profile.feasibility.decided`는 source/time/unit/outcome 의미. host/environment/mode/run ID 상관. JSONL 진단·비공개 측정 원본·마스킹 요약을 분리; MAC/IP·일련번호·경로는 제한/마스킹. 계산은 입력 hash·단위·공식/산식 version으로 재현 검증. |
| 실패·복구 | 센서 누락, 자원 변동, 지원표 불일치는 성공 추정 대신 conditional/blocked. 원격 접속 단절 위험이 있는 조사 절차는 중단하고 로컬 복구가 가능한 후속 P02 승인 전 변경하지 않는다. |
| 완료 조건 | 계약: 측정 항목·단위·산정법·지원 상태 모델 완성. 모의 코드 검증: 계산·누락 거부·무변경 확인. 실호스트: 별도 실습 PC에서 실제 측정과 G01/G02 결론이 있어야 함; 개발 PC 자료로 대체 불가. |
| 인계 정보 | F05는 네트워크 기존 상태·G03에 필요한 조건, F06/F10은 실행 가능한 자원/제품 profile, F09는 OS/도메인 전제를 받는다. 미달 모드는 명시적으로 차단한다. |

## F05 — 네트워크·접근 경로

| 항목 | 계약 |
|---|---|
| 목적 | 관리·스토리지·클러스터·서비스·고객 SSH의 통신 의도를 주소·전달 방식과 분리해 검증한다. |
| 추가 범위 | 스위치/서브넷/라우팅/DNS/시간 설정, 중첩 MAC/NAT 경로 판정, 포트·방향·출발지 허용표, 주소 예약/해제, VIP와 실제 송신 IP 분리. |
| 수정 범위 | 연결 성공은 단일 ping이 아니라 이름·포트·신원·방향·실제 경로의 증거로 판정한다. |
| 삭제 범위 | 현재 삭제 없음(실행 코드가 없음). 향후 금지할 행동은 임의 기본 주소나 자동 NAT 선택이다. G03의 사용자 주소·SSH·이동·VIP 조건을 대체할 수 없다. |
| goal | 각 송신자/수신자 흐름이 최소 권한으로 명시되고 직접 SSH 및 필요한 관리 경로가 모드별 검증 가능하다. |
| non-goal | 실제 스위치/방화벽 변경 승인, Cloud VIP를 Windows가 소유, G03 선택 자동화. |
| 추적 작업 | P02-01, P02-02, P02-03, P02-04, P02-05, P02-06, P02-07, P02-08 |
| 책임 모듈 | M01 network catalog/config, M09 신원·출발지 검증, M11 모드별 네트워크 의존성. |
| 선행/게이트 | P01-07 및 G03. P02-04 변경은 원격 차단 시 로컬 복구 경로·실행 전 snapshot 확인 전 금지. |
| 코드 동작(단계) | (1) 관리/SMB/클러스터/고객/Linux VIP 목적을 주소와 독립 정의 (2) 중첩 NIC 전달 후보를 공식 근거·실습 조건에 대조 (3) 사용자 지정값 schema·중복 주소·route/name 검사 (4) 필요한 포트의 source/destination/direction 최소표 작성 (5) 승인된 변경 전 접속 기준 저장 (6) 격리된 경로 준비·변경 (7) 관리 재접속·DNS/time·SSH·SMB/TLS 경로 각각 검증 (8) 서비스4+4의 VIP와 실제 egress를 별도 확인. |
| 목표-input | `network_profile_id`: 등록 구성 식별, M01 원천, 필수. `subnets/routes`: 사용자 확정 주소 계획, G03 원천, 필수/중복·겹침 거부. `switch_binding`: 외부/내부 경로 의미, 호스트 inventory와 G03, 필수/지원 방식 확인. `name_resolution/time_sources`: DNS·시간 의존, 운영 계획 원천, 필수. `flow_matrix`: source→destination 목적·프로토콜, 계획과 제품 공식 포트 원천, 관리/데이터 구분. `address_reservation_owner/lifecycle`: 주소 할당/해제 책임, 승인 사용자 계획 원천, 필수. `ssh_access_path`: 고객 단말에서 L2 게스트로 가는 승인 경로, G03 원천, 필수. `service_vip/egress_identity`: 서비스 주소와 실제 송신 주소, Linux 실측 원천, P02-08에는 미확정 가능. |
| 목표-output | `connectivity_plan`: 승인된 목적·방향·선행, M01 생산/설치·방화벽 작업 소비. `effective_route/name/time`: 실제 경로·해석·시간관측, 수집기 생산/M11·서비스 인계 소비, 미확인은 unavailable. `firewall_allowlist`: 포트·source·destination·제품 근거, M01 생산/운영 소비, 과도 범위 거부. `address_lease_state`: reserved/assigned/released 의미, 주소 관리자 생산/M03/M04 소비, 불명확 시 재할당 금지. `ssh_reachability`: 경로·게스트 IP 관측·검증수준, M06/증거 생산/Cloud 인계 소비, IP와 키 비밀은 보호. `egress_identity`: 실제 source와 VIP 관계, 관측기 생산/Cloud mTLS allowlist 소비, 미실측은 null/unknown. |
| 코드동작확인 Test방식 | 정상: 격리된 flow fixture 및 등록 이름 준비→양방향 경로 probe→예상 포트·신원만 성공→패킷/연결/이름 결과 증거. 거부: 불허 source·포트·잘못된 주소·신뢰하지 않는 forwarding header 준비→접속→거부와 기존 경로 보존→방화벽/감사·전후 상태 확인. 중복: 같은 주소 예약 요청 동시 입력→한 소유자만 예약, 중복 할당 거부→lease 고유성 확인. 실패·복구: DNS/time/링크 중단 또는 변경 후 원격 세션 상실 준비→변경 중단/로컬 되돌리기→기존 관리경로 복원→라우팅·DNS·이벤트 증거 대조. |
| 코드동작확인 로깅방식 | `network.flow.checked/denied`, `address.reserved/released`, `network.change.rolled_back`는 흐름/판정 의미. environment/mode/source role/destination role/protocol/run ID로 상관. 진단 JSONL·방화벽/제품 원본·비공개 주소 mapping 증거를 분리하고 공개 요약은 IP/MAC 마스킹. 독립 단말/실제 관측 및 변경 전후 비교로 검증. |
| 실패·복구 | 이름·시간·route 불명확, 주소 중복, TLS 출발지 불일치 시 신규 작업/주소 사용 차단. 연결 끊김이면 승인된 로컬 복구 절차로 원복하고 기존 접근 복구 전 다음 단계 금지. |
| 완료 조건 | 계약: 흐름/주소 수명주기·VIP/egress 구분 완성. 모의 코드 검증: 중복/불허 경로 거부. 실호스트: G03 사용자 선택 뒤 실제 client→guest SSH, 관리·SMB 경로와 service egress 측정이 있어야 통과. |
| 인계 정보 | F06/F07/F08에는 내부·고객 네트워크와 SSH 선행조건을, F09/F10에는 도메인/서비스 통신표를, 서비스 카드에는 VIP·실제 출발지와 주소 수명주기를 전달한다. |

## F06 — 단독 Compute

| 항목 | 계약 |
|---|---|
| 목적 | 승인된 단독 Compute VM에 중첩 Hyper-V를 구성하고 내부 게스트 실행 기반을 증명한다. |
| 추가 범위 | Compute 사양, 종료 중 가상화 확장 노출, Windows Server 설치/기본 설정, 내부 Hyper-V·관리도구, 허용 저장 루트·의존성. |
| 수정 범위 | 일반 L1 VM과 중첩 Hyper-V 호스트를 구별하며 가동 대수·시작 순서를 mode controller와 맞춘다. |
| 삭제 범위 | 현재 삭제 없음(실행 코드가 없음). 향후 금지할 행동은 소유 표식·실제 참조 확인 없는 기존 VM/경로 삭제다. 보존 가능한 데이터와 다른 사용 자산을 보호한다. |
| goal | 물리 Hyper-V 설정과 Compute 내부 가상화 기능이 일치하고 안전한 허용 범위에서 게스트 생성 기반이 준비된다. |
| non-goal | 클러스터 구성·VMM 편입·고객 VM 서비스 완성·G01 자원 제한 우회. |
| 추적 작업 | P03-01, P03-02, P03-03, P03-04, P03-05, P03-06 |
| 책임 모듈 | M05 Hyper-V/host execution adapter, M11 host lifecycle/mode control, M01 role profile. |
| 선행/게이트 | P02-08, F04 실행가능 판정, F05 승인 네트워크, P03 변경에 필요한 G01/G02. 실제 VM 생성 전 대상·복구 경로 확인. |
| 코드 동작(단계) | (1) 계획된 Compute profile/여유 확인 (2) 명시된 이름·세대·CPU·고정 RAM·OS 디스크·NIC로 L1 생성 (3) 종료 상태에서 nested virtualization 설정 (4) Windows Server 설치/업데이트/이름·시간·망 확인 (5) Hyper-V 역할·도구와 내부 switch 구성 (6) 허용 저장 root/관리 신원/실습 소유 표식 설정 (7) 외부 재부팅과 내부 VM 의존성·자동시작을 관측. |
| 목표-input | `compute_role_id`: 단독 Compute 역할, M01 catalog, 필수. `host_target`: 승인된 물리 Hyper-V 식별, F04 inventory, 필수. `vm_generation/cpu/memory/os_disk/nic`: 등록 사양·단위, G01/role profile, 필수/가용예산 내. `nested_virtualization_policy`: 노출 조건, 공식 제품 절차+실측, 필수/host 호환 확인. `os_media_identity`: OS 버전/edition/hash, G02, 필수/검증 설치물. `switch_profile/storage_root`: F05/M01 원천, 필수/allowlist. `management_principal_ref`: 제한된 관리 신원, P07 이후 서비스 신원 계획, 비밀 본문 금지. |
| 목표-output | `compute_vm_id`: 실제 Hyper-V GUID, M05 생산/M11·M08 소비, 조회 전 미확인. `host_config_observation`: CPU/RAM/디스크/NIC/nested flag, Hyper-V 생산/검증 소비. `guest_readiness`: 설치·재부팅·내부 Hyper-V 상태, M06 생산/후속 F07 소비, 단계별. `allowed_roots/ownership_marker`: 적용된 관리 범위, M05 생산/삭제 보호 소비. `startup_dependency`: 외부/내부 시작·정지 관계, M11 생산/복구 소비, 미검증은 수동 순서. |
| 코드동작확인 Test방식 | 정상: 격리 물리 Hyper-V와 승인 여유 준비→Compute를 만들고 내부 Hyper-V 기능 확인→실제 설정/게스트 상태 일치→host·guest 증거 확인. 거부: 비승인 root, 부족한 RAM/디스크 또는 미지원 CPU 준비→생성 단계 진입→preflight 차단·자원 미생성→전후 inventory 확인. 중복: 같은 역할 요청 재전송→기존 소유 marker/VM ID 발견, 두 번째 VM 없음→Hyper-V inventory와 요청 mapping 확인. 실패·복구: 설치/재부팅 중 인터럽트 준비→현재 단계 재관측, 손상/잔여를 분류하고 승인된 되돌림→다른 VM/디스크 영향 없음→호스트 상태·setup/evidence 기록 확인. |
| 코드동작확인 로깅방식 | `compute.provision.started`, `nested_virtualization.verified`, `compute.readiness.failed` 의미. run/job/invocation/compute VM ID 연결. ops JSONL, 작업 원장·감사, 설치 로그와 비공개 증거를 분류; 호스트 이름·주소·라이선스 값은 제한/마스킹. 설정 조회와 게스트 관측의 독립 대조로 검증. |
| 실패·복구 | 중첩 기능 불일치·OS 설치 실패·접근 손실 시 F07로 진행 금지. VM/디스크 삭제는 소유 표식·실제 참조·보존 정책이 입증된 경우만; 불명확하면 보존·manual_required. |
| 완료 조건 | 계약: 사양·루트·표식·시작 순서가 정의됨. 모의 코드 검증: allowlist/preflight/멱등 동작. 실호스트: P02 통신과 G01/G02를 통과한 물리 호스트에서 L1/L2 상태 및 재부팅 의존성 증거 필요. |
| 인계 정보 | F07에는 내부 Hyper-V 버전·스위치·허용 저장경로·nested 제약을, F08에는 직접 제어 권한과 실제 ID 조회 경계를 전달한다. |

## F07 — 이미지·고유 신원

| 항목 | 계약 |
|---|---|
| 목적 | 공식 Rocky 설치 기반을 검증하고 복제본마다 네트워크·OS·SSH 고유 신원을 갖게 한다. |
| 추가 범위 | 공식 ISO/checksum 추적, L2 CPU/Gen2/Secure Boot, 설치·네트워크·시간·SSH, 초기화, 이미지 freeze/catalog, 독립 복제 시험. |
| 수정 범위 | Hyper-V 부팅 가능성, Rocky 제품 지원, VMM guest agent/template 지원을 각각 분리 기록한다. |
| 삭제 범위 | 현재 삭제 없음(실행 코드가 없음). 향후 기본 이미지에서 제외할 방식은 공유 부모 디스크 의존이다. 복제/수명주기·보존 위험이 별도 검증 전 해소되지 않는다. |
| goal | 이미지 원본은 불변 등록되고 두 복제본의 machine-id·IP/MAC·SSH host key가 독립임을 확인한다. |
| non-goal | 임의 외부 이미지, 지원되지 않는 에이전트 호환 주장, 고객 비밀키 보관, Rocky 관리 인터페이스 제공. |
| 추적 작업 | P04-01, P04-02, P04-03, P04-04, P04-05, P04-06, P04-07, P04-08 |
| 책임 모듈 | M12 ImageProvisioning, M01 image catalog, M05 Hyper-V 생성, M06 게스트/VM 관측. |
| 선행/게이트 | P03-06, P02-06. 공식 자료는 Rocky 공식 배포만 사용. L2 CPU 미달이면 해당 이미지 profile 차단. |
| 코드 동작(단계) | (1) 공식 배포처에서 ISO와 checksum 출처/버전을 기록 (2) L2 CPU·Gen2·Secure Boot 조건 검증 (3) 수동 설치, DNS/time/network/SSH 및 정상 종료 확인 (4) 비밀키 없이 승인 공개키 전달·초기화 절차 구성 (5) machine-id·SSH host keys·hostname·네트워크 신원 재발급 순서 확인 (6) full-copy와 초기화 방식 시험 후 이미지 동결 (7) hash·버전·최소 disk·지원 profile·초기화 version 등록 (8) 독립 복제 2대를 재부팅/초기화하고 충돌 여부 확인. |
| 목표-input | `image_source_identity`: 공식 ISO 릴리스/아키텍처/checksum 참조, Rocky 공식 자료 원천, 필수. `guest_cpu_features`: x86-64-v3 등 요구 충족 관측, L2 source 원천, 필수. `generation/secure_boot_policy`: 부팅 설정 의미, Hyper-V profile와 공식 조건, 필수/검증 조합만. `network/time_profile`: F05 승인값 참조, 필수. `initialization_version`: 게스트 고유화 절차 버전, M12 원천, 필수. `ssh_public_key_ref/material`: 공개키 초기화 입력, 승인 계약/고객 이후 단계 원천, P04 수동시험은 비밀 없는 시험키만. `copy_strategy`: 독립 복제 전략, 시험 결과 원천, 선택 전 candidate. `image_id/version/min_disk/profile_allowlist`: catalog 등록값, 운영 catalog 원천, 필수/불변. |
| 목표-output | `source_hash/provenance`: 원본 검증값/공식 출처, M12 생산/감사 소비. `image_manifest`: 버전·최소 디스크·초기화·허용 profile·지원 수준, M12 생산/M01·M05 소비, 미확정 지원은 unsupported/unverified. `clone_identity_observations`: VM별 machine-id/MAC/IP/host-key fingerprint uniqueness 결과, M06/게스트 probe 생산/검증 소비, 원문 키 제외. `guest_readiness`: boot/network/time/SSH 구분 결과, M06 생산/M04·M13 및 S09 소비, unknown 허용. `image_state`: draft/frozen/retired, M12 생산/M03 소비, frozen 원본 변경 거부. |
| 코드동작확인 Test방식 | 정상: 고정 공식 ISO/hash와 격리 Compute 준비→설치 후 복제 2대 초기화→서로 다른 신원·각 SSH 성공→manifest/hash 및 fingerprint 비교. 거부: checksum mismatch, CPU 조건 미달, 비허용 이미지 버전 준비→검증/생성 요청→등록/실행 차단→출처·오류 증거 확인. 중복: 같은 initialization invocation 재호출→같은 결과의 안전 처리 또는 명시적 거부, 신원 재사용 금지→전후 machine-id/host-key 비교. 실패·복구: 초기화 일부 실패/원본 변경 시도 준비→clone 격리, 원본 frozen 유지, 알려진 복구 절차 수행→다른 clone 신원·원본 hash 불변→게스트 및 manifest 증거. |
| 코드동작확인 로깅방식 | `image.verified/frozen`, `guest.identity.initialized`, `clone.identity.collision` 의미. image ID/version/hash/run/invocation/VM ID로 연결. JSONL 진단·비공개 원본 hash/provenance·마스킹 fingerprint summary 분류; SSH 공개키 본문·private key·실 IP는 로그 제외. 독립 키 fingerprint 및 OS identity 조회로 검증. |
| 실패·복구 | 원본 hash/고유화 검증 실패는 이미지 사용 중지; 기존 복제본을 자동 재초기화하지 않고 격리. VMM 지원 여부 미검증은 실행 가능과 분리해 기록하고 지원 주장 금지. |
| 완료 조건 | 계약: image manifest·신원 처리·지원 수준 의미 완료. 모의 코드 검증: checksum/allowlist/frozen/초기화 재실행 정책. 실호스트: 실제 L2 두 VM 부팅·네트워크·SSH와 고유 machine-id/MAC/host key 증거 필요. |
| 인계 정보 | F08은 검증된 이미지 ID/version/hash/복사 전략을, 서비스 생성 카드는 지원 한계와 공개키/게스트 준비 구분을 받는다. |

## F08 — Hyper-V 직접 자동화

| 항목 | 계약 |
|---|---|
| 목적 | VMM 도입 전 단독 Hyper-V에서 승인 plan의 생성·조회·정리·전원 동작을 안전하게 수행한다. |
| 추가 범위 | allowlist 변환, 재조회, 사전 용량/파일/주소 검사, 이미지 복사와 초기화, 부분 실패 추적, 멱등, 보존 정책. |
| 수정 범위 | 사용자 요청을 고정 PowerShell 기능의 typed arguments로 변환하며 이름/path/prefix 단독으로 대상 삭제하지 않는다. |
| 삭제 범위 | 현재 삭제 없음(실행 코드가 없음). 향후 금지할 실행 경로는 `Invoke-Expression`, 사용자 명령 문자열, 임의 path/switch dispatch다. 고정 실행 경로와 검증된 대상 경계를 지킨다. |
| goal | 동일 요청 재전송·부분 실패에도 중복 VM이 생기지 않고 실제 사양/ID/전원 결과가 조회값과 일치한다. |
| non-goal | VMM-managed VM 변경, 무조건 exactly-once 보장, 미결정 G08 cleanup/강제정지 정책 추정. |
| 추적 작업 | P05-01, P05-02, P05-03, P05-04, P05-05, P05-06, P05-07, P05-08, P05-09 |
| 책임 모듈 | M03 작업 단계·멱등·lease, M04 수명주기 정책, M05 Hyper-V/PowerShell adapter, M06 실제 inventory, M12 이미지. |
| 선행/게이트 | F07-08 이미지와 P03-06. G08 생성 실패 잔여·삭제 보존·강제 동작 정책은 P05-05/P05-07/P05-08 전 해당 하위동작에서 통과해야 함. |
| 코드 동작(단계) | (1) profile/image/network ID를 등록된 immutable plan으로 해석 (2) 실제 VM/요청 marker·용량·파일·주소·동시 작업 확인 (3) `submit_intent`와 invocation 저장 (4) 서버가 승인 경로에 복사 후 VM/NIC/spec/init 설정 (5) 기동·제품 Job/실 VM 재조회 (6) 단계별 소유 자원·잔여물을 기록 (7) 동일 키 본문 해시 비교, 다른 본문 충돌 (8) cleanup은 root·실제 참조·소유 표식·G08 확인 (9) start/stop/restart 상태표 적용 및 실제 결과 재조회. VMM 관리 상태로 바뀐 대상은 Hyper-V 직접 변경을 금지한다. |
| 목표-input | `external_job_id/idempotency_key/body_hash`: 내부 접수/멱등 의미(외부 wire는 `external_idempotency_key`), S03 원천, F08 adapter 입력에서 요구. `operation`: allowlisted 동작, F01 계약 의미, 필수. `external_vm_id`: 논리 대상, mapping 원천, 생성 시 내부 식별. `profile/image/network ID+version`: M01 catalog, 필수/등록값 일치. `invocation_id/protocol_version`: bridge protocol 내부 필드, M03/M05 원천, 필수. `validated_target/root`: 내부 실행 plan에서 생성, M01/F06 원천, 외부 path 직접 입력 금지. `resource_preconditions`: 용량·주소·현재 VM/job 관측, M06/host inventory 원천, 필수. `cleanup_policy/preserve_data`: G08 승인 원천, 하위 동작에 따라 필수. |
| 목표-output | `execution_disposition`: submitted/not_submitted/unknown, bridge/M05 생산/M03 복구 소비. `hyperv_vm_id`: 실 조회 ID, M05/M06 생산/M03 binding 소비, 관측 전 미확정. `vmm_job_id`: 이 adapter에는 없음; VMM 전환 후 별도 값. `step_results/owned_resources`: 단계·파일/VM 소유 증거, M05 생산/M03 cleanup 소비, 일부 실패 시 부분 목록. `actual_spec/power_state`: 제품 조회, M06 생산/검증 소비. `cleanup_status/residuals`: M04/M05 생산/M03/운영 소비, 보존/미확정은 manual_required. `operation_result`: 검증된 결과, M03 생산/Cloud 후속 소비, timeout은 unknown. |
| 코드동작확인 Test방식 | 정상: 격리 Hyper-V·frozen image·빈 승인 root 준비→생성→재조회·전원/사양 확인→요청/VM ID와 단일 VM 확인. 거부: 임의 path/switch/profile, 외부 VM, 공간 부족, VMM-owned VM 준비→실행→무변경 거부→감사/전후 inventory 확인. 중복: same key/same body 및 동시 제출, 이어 same key/different body→첫 요청만 자원 생성, 동일 응답 또는 충돌→DB 고유 제약·VM marker 확인. 실패·복구: 이미지 복사 후/VM 생성 후/submit 응답 전 bridge 중단 준비→reconcile 제품 job·VM·파일, G08에 따라 보존/정리/수동조치→재실행으로 중복 금지→job_steps·잔여 목록·제품 실물 대조. |
| 코드동작확인 로깅방식 | `execution.submitting/submitted/unknown`, `vm.observed`, `cleanup.required`, `operation.rejected` 의미. job/attempt/invocation/external_vm/hyperv_vm/stage 상관. Worker/ops JSONL은 순환 가능, job/audit 원장·소유 자원 목록은 보존, VHDX 원본 증거는 ACL 제한. 명령행·credential·사용자 공개키는 기록 금지. 원장·Hyper-V 조회·filesystem 참조를 함께 확인. |
| 실패·복구 | timeout/worker lease 만료는 명령 재실행 근거가 아니다. 제품 job/VM/file 대조 불가 시 예약·대상 소유 상태 유지하고 needs_attention. cleanup 실패는 원 오류와 별도 저장하며 미확정 데이터는 보존한다. |
| 완료 조건 | 계약: 고정 operation·plan·중복·잔여 상태 정의. 모의 코드 검증: 허용/거부/동시·부분 실패 fixture 통과. 실호스트: P05-09까지 실제 Hyper-V 생성·조회·정리/전원 대조가 필요하며, Pester mock만으로 완료 불가. |
| 인계 정보 | F09/F10에는 요청·실 VM 조회·실행결과 envelope를 넘긴다. P06은 동일 정책을 영속 C# Jobs/Worker로 소유하고 PowerShell에 큐/상태 머신을 중복 구현하지 않는다. |

## F09 — AD·DNS·서비스 신원

| 항목 | 계약 |
|---|---|
| 목적 | 인프라 이름/시간/도메인 의존성을 제공하고 수신·실행·제품·SQL·통지 신원을 최소 권한으로 분리한다. |
| 추가 범위 | AD/DNS VM 준비·도메인 가입·시간 계층, IIS/Worker/VMM Run As/SQL read/notification identity, ACL·복구 절차. |
| 수정 범위 | 인증서 유효성, 출발지 제한, 서비스 identity, 명령 권한은 서로 별도로 검사한다. |
| 삭제 범위 | 현재 삭제 없음(실행 코드가 없음). 향후 금지할 저장 방식은 비밀/개인키를 일반 설정·로그·작업 payload에 기록하는 것이다. 권한·회전·감사 경계를 유지한다. |
| goal | 각 서비스가 필요 작업만 수행하며 AD/DNS/time 장애와 계정 교체를 진단·복구할 수 있다. |
| non-goal | 사용자 로그인/회원 권한, CA 실발급 확정(G05), Cloud JWT, VMM/SQL 구현 자체. |
| 추적 작업 | P07-01, P07-02, P07-03, P07-04, P07-05, P07-06 |
| 책임 모듈 | M09 Security identity/policy, M11 host/service readiness, M01 역할 설정. |
| 선행/게이트 | P02-08, G02. AD VM 설치/도메인/백업은 승인된 버전·주소·복구 경로 후. 서비스 계정/CA 실제 선택은 P06/G05 및 P08 설치 조건과 조정. |
| 코드 동작(단계) | (1) AD/DNS 역할 VM과 복구정보 준비 (2) 이름 해석·도메인 가입·시간 신뢰 검증 (3) API 수신, 작업 Worker, VMM 서비스/Run As, SQL read-only, callback 발신 신원을 분리 (4) 계정별 필요한 파일·개인키·작업 root ACL 적용 (5) 허용 identity 성공/비허용 identity 거부 시험 (6) AD/time 잠시 불가 시 readiness 차단·진단 (7) 계정 교체·AD 복원 절차와 검증 evidence 기록. |
| 목표-input | `directory_role_profile`: 도메인/DNS 역할, M01·G02 원천, 필수. `domain_name/dns/time_sources`: 운영 확정값, P02/G03 계획, 필수/이름·시간 충돌 금지. `service_principal_map`: 서비스별 신원과 목적, 보안 설계 원천, 필수/일대일 책임. `permission_matrix`: operation/대상/권한 근거, M09 policy와 제품 요구, 최소 권한. `credential_ref/certificate_key_ref`: 비밀 저장소 참조, 승인 운영 원천, 필수 시 존재/본문 금지. `acl_targets`: 실행 파일·로그·증거·작업경로, 설치 배포 원천, 정규 root 제한. `rotation/recovery_policy`: 회전·복원 시점과 확인, P07 계획/G05 관련 조건, 문서화 필수. |
| 목표-output | `identity_binding`: TLS/서비스 신원→service principal mapping, M09 생산/M02·M08 소비, 인증만으로 전체 권한 부여 금지. `authorization_decision`: 허용/거부와 reason code, M09 생산/audit 소비. `domain_dns_time_readiness`: 실제 확인 source/time, M11 생산/readiness 소비, 불가 시 blocked. `acl_verification`: 계정별 접근 결과, 검증기 생산/운영 소비. `rotation_record`: 이전/신규 신원·시간·거부 확인 evidence, 운영기록 생산/M09 소비, 비밀 제외. |
| 코드동작확인 Test방식 | 정상: 격리 도메인·등록 서비스 신원·정책 준비→필요 작업 수행→최소 권한으로 성공→AD/DNS/time·ACL·감사 확인. 거부: 잘못된 인증서 mapping 또는 과도한 계정/다른 identity 준비→작업 호출→403/인증 거부, 변경 없음→보안 감사·ACL 확인. 중복: 같은 서비스 계정 신원으로 반복 인증→같은 principal로 기록되고 추가 권한/계정 생성 없음→identity map 대조. 실패·복구: DNS/time/AD 단절 또는 credential 교체 도중 오류 준비→readiness 차단, 승인 복구/롤백 후 재검증→기존 정상 신원 외 호출 거부→이벤트·교체 evidence 확인. |
| 코드동작확인 로깅방식 | `security.authenticated/denied`, `identity.permission.checked`, `directory.readiness.failed`, `credential.rotated` 의미. principal ID/service/module/correlation/stage로 연결. Security audit는 업무 DB, 제품 이벤트는 별도 제품 로그, 개인키/비밀은 어디에도 기록하지 않음; 도메인·계정·주소 공개본 마스킹. 허용·거부 matrix와 실제 ACL/인증 결과 대조. |
| 실패·복구 | AD/DNS/time 또는 identity mapping 불확실 시 신규 접수/작업 거부. 키/자격증명 권한 오류는 fail closed. 복구 중 기존 identity를 제거하기 전 새 identity 시험을 통과시키고 이전 신원 폐기 결과를 확인한다. |
| 완료 조건 | 계약: 역할별 신원/권한/장애 의미 확정. 모의 코드 검증: 신원 mapping·인가/거부·readiness 정책. 실호스트: 도메인 가입·실 ACL·VMM/SQL/인증서 접근은 G02/G05 및 실제 P07 evidence 후에만 승인. |
| 인계 정보 | F10에 VMM·SQL에 필요한 신원/권한표를, API/Worker 카드에 TLS principal과 실행 신원 경계를 전달한다. G05 CA·인증서 선택은 미통과 시 미확정으로 둔다. |

## F10 — SQL/VMM 설치·Compute 편입

| 항목 | 계약 |
|---|---|
| 목적 | 승인 버전 조합으로 SQL/VMM 관리 기반을 세우고 공식 VMM 경로에서 단독 Compute/Rocky VM을 제어한다. |
| 추가 범위 | 도메인 가입된 VMM/SQL VM, SQL 요구·계정·메모리·백업, VMM install/update/console/PowerShell/library, Run As 기반 Compute 등록, image·Job·실 VM 대조. |
| 수정 범위 | 직접 Hyper-V와 VMM 경로를 구별하고 VMM 관리 대상은 이후 변경을 VMM adapter로 통일한다. |
| 삭제 범위 | 현재 삭제 없음(실행 코드가 없음). 향후 금지할 행동은 VMM 내부 DB에 사용자 쓰기, 임의 table/trigger를 추가하는 것이다. 제품 소유 DB 경계와 공식 관리 표면을 지킨다. |
| goal | 설치 버전·모듈·Compute 등록이 실측되고 VMM Job ID, Hyper-V VM ID, 외부 job/VM ID가 정확히 연결된다. |
| non-goal | G06 조회 DB 결정, VMM 내부 DB를 Cloud 읽기 대상으로 자동 선택, Storage/cluster 구성(P10/P11), VMM 기능의 공식 지원을 미검증 OS에 가정. |
| 추적 작업 | P08-01, P08-02, P08-03, P08-04, P08-05, P08-06, P08-07 |
| 책임 모듈 | M05 VMM execution adapter, M06 inventory, M11 host/service lifecycle, M12 image provisioning, M09 Run As/service identity. |
| 선행/게이트 | F06 단독 Compute, F07-08 이미지, F09 domain/service identity, P06-09 설치 준비, G02 버전·edition·업데이트·권한. 실제 SQL read object는 P09/G06 범위로 유보. |
| 코드 동작(단계) | (1) VMM/SQL VM 자원·도메인·사전조건 preflight (2) SQL 설치·instance/collation/service identity/memory ceiling/backup 검증 (3) 승인된 VMM 버전 설치/update, 콘솔/PowerShell/library 확인 (4) Run As 등 제품 요구 신원으로 Compute 등록 (5) 호스트·switch·저장 경로를 제품 inventory에서 조회 (6) Rocky image/library·disk copy·configuration 주입을 VMM 경로로 시험 (7) 작업 ID↔VMM Job ID↔Hyper-V GUID binding, 조회/전원/정리 대조 (8) VMM/SQL/권한/host 장애를 각각 주입, 자원예산 갱신. |
| 목표-input | `product_version_matrix`: Windows/VMM/SQL/ADK/runtime edition·update, P01/G02 공식+실측 원천, 필수/선택 조합만. `vmm_sql_role_profile`: 분리 VM의 CPU/RAM/disk/network, F02/F04, 필수/예산 내. `domain_identity_map`: F09 신원·권한, 필수. `sql_instance_policy`: instance/collation/service/memory/backup semantics, 제품 요구·운영 승인 원천, 필수. `vmm_management_target`: Compute ID·연결·switch/storage scope, F06 inventory, 필수. `run_as_ref`: 제품 등록 권한 identity 참조, F09, 필요 최소 권한. `image_manifest/library_policy`: F07 동결 이미지·copy/init 지원수준, M12 원천, 필수. `job_operation`: 승인된 create/query/start/stop/cleanup test action, F01/F08 contract, 허용 enum. |
| 목표-output | `sql_readiness`: instance/version/connection/backup status, 설치·probe 생산/F10·후속 계획 소비, G06 객체 승인 아님. `vmm_readiness`: version/module/connectivity/library 상태, adapter 생산/M05 소비, 미확인은 blocked. `registered_compute`: VMM host identity·capabilities·switch/path observation, VMM 생산/M06/M11 소비. `vmm_job_id`: VMM command 결과, M05 생산/M03 복구 소비, unknown 별도. `hyperv_vm_id`: 실제 VM inventory, M06 생산/M04 binding 소비. `operation_mapping`: 외부 job/VM↔VMM Job↔Hyper-V GUID와 version, M03/M04 생산/운영 소비. `resource_measurements`: SQL/VMM/Compute 사용량, M10/M06 생산/F04 예산 업데이트 소비. `support_status`: tested/officially supported/unverified 의미 분리, M01/M10 생산/인계 소비. |
| 코드동작확인 Test방식 | 정상: 승인 조합·도메인·identity·Compute/image 준비→VMM 공식 관리 경로로 별도 Rocky 생성/조회/전원/정리→모든 ID와 사양 일치→VMM Job/Hyper-V 실물/작업원장 비교. 거부: 미지원 버전, 권한 없는 Run As, 비허용 host/path 또는 VMM DB 직접 쓰기 시도 준비→호출→preflight/권한 거부, DB·자원 변경 없음→제품 audit/DB read-only와 inventory 확인. 중복: 동일 작업 재전달 또는 같은 VMM Job 확인→기존 Job/VM 연결·새 생성 금지→M03 idempotency·VMM/Hyper-V mapping 확인. 실패·복구: SQL/VMM/host 단절, Job ID 저장 전 worker 종료 준비→각 의존성 복구 후 Job/VM 재조회, unknown이면 수동대기→중복 submit 없음→제품 이벤트·job_steps·실 VM 및 자원 재측정 확인. |
| 코드동작확인 로깅방식 | `sql.readiness.checked`, `vmm.connected`, `compute.registered`, `vmm.job.submitted/unknown`, `resource.sampled`는 안정 의미. external_job/attempt/invocation/vmm_job/hyperv_vm/host/mode 상관. Worker/ops JSONL, app 원장·감사, SQL/VMM 제품 로그, 비공개 설치/실측 증거를 분류. connection string·자격증명·전체 제품 객체·VMM 내부 DB 내용은 기록 금지; 서버/주소 공개본 마스킹. 제품 inventory·DB read-only 확인·실 VM 대조로 검증. |
| 실패·복구 | VMM/SQL 장애는 VMM 작업 재제출 근거가 아니다. 기존 job/VM을 제품에서 재조회하며 미확정이면 실행/용량 reservation 보존. SQL 상태 확인은 G06 결정과 분리; 내부 DB 수정 금지. Rocky 지원/에이전트 미검증이면 실행 가능 여부와 지원상태를 나눠 인계. |
| 완료 조건 | 계약: 설치조합·신원·ID mapping·오류 복구 명세. 모의 코드 검증: VMM adapter 계약 fixture, unknown/restart/중복 및 금지 DB write 거부. 실호스트: P08-07까지 설치된 제품 버전에서 공식 명령이 Compute/Rocky 실제 VM을 제어하고 자원 사용량·장애 복구 증거가 있어야 함. |
| 인계 정보 | P09에는 VMM Job·inventory·실제 ID·SQL 접속의 검증 사실만 넘기고 G06 조회 원천 선택은 열어둔다. P10은 Compute 등록/저장 경로와 갱신된 자원 여유를 받는다. |

## 카드 경계와 전체 게이트

| 카드 | 추적 P 작업 수 | 선행 흐름 |
|---|---:|---|
| F01 | 3 | P00-01 → P00-02/P00-05 |
| F02 | 2 | F01/P00-02 → P00-03 → P00-04 |
| F03 | 3 | P00-03 + F01 → P00-06 → P00-07 → P00-08 |
| F04 | 7 | F03 → P01, G01/G02 |
| F05 | 8 | F04 → P02, G03 |
| F06 | 6 | F05 → P03 |
| F07 | 8 | F06/P02-06 → P04 |
| F08 | 9 | F07 → P05; G08는 관련 하위동작별 차단 |
| F09 | 6 | F05/G02 → P07; G05는 인증서 구성에서 보존 |
| F10 | 7 | F06/F07/F09/P06-09 → P08, G02 |

합계 59개 P 작업이다. 이 카드 세트는 P00~P08 전체나 145개 작업을 대체하지 않는다. 특히 P06은 F08 다음의 영속 접수·인증·Worker 기반 선행계획으로 남고, P08은 G06 SQL 조회 선택을 확정하지 않는다. 실호스트 변경 전에 해당 P의 선행작업과 게이트를 개별 확인한다.
