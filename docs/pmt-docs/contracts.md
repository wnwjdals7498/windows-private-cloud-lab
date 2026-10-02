# 모듈 통신과 데이터 계약

[모듈](modules.md) · [동작 레퍼런스](reference-flows.md) · 상태: 실제 Windows 계약 설계안

## 1. 통신 경계

| 경계 | 방식 | 인증·형식·수명 |
| --- | --- | --- |
| API 내부 모듈 | C# method + immutable DTO | DI 조립·명시적인 transaction, 내부 HTTP 없음 |
| API → Worker | 같은 영속 작업 DB | API가 접수, Worker가 원자적 점유. 메모리 signal은 깨우기 보조만 |
| Worker → PowerShell | 고정 프로세스 + 제한된 stdin JSON / stdout 결과 JSON | 신뢰된 스크립트·Worker 신원·protocol version·invocation ID |
| PowerShell → Hyper-V/VMM | 설치 버전 공식 cmdlet | 전용 계정·허용 대상. 원격 범위와 제품 연결은 adapter 내부 |
| Linux API → Windows API | HTTPS JSON | 상호 TLS·실제 등록 출발지·서비스별 권한 |
| Windows Worker → Linux API | HTTPS JSON 완료 이벤트 | 상호 TLS·등록 주소·outbox, 전송 재시도 가능 |
| Linux API → MSSQL | TLS·읽기 전용 객체 | 서버 인증서 검증·전용 login·G06 승인 객체 |
| Compute → Storage | SMB 3 | AD 인증·공유/NTFS 권한·암호화·출발지 제한 |
| 고객 단말 → Rocky | SSH | 공개키·허용 계정·출발지, 비밀키는 고객 소유 |

TLS가 종료되는 IIS와 애플리케이션의 서비스 신원 매핑을 함께 검증한다. 외부가 보낸 인증서 헤더나 X-Forwarded-For를 그대로 신뢰하지 않는다. 인증서는 체인·유효기간·용도·이름·폐기 정책과 등록 서비스 매핑을 검증한다. 오프라인 CA 환경에서도 폐기 정보 배포·갱신 실패 정책을 G05에서 정한다. [Microsoft 인증서 인증](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/certauth)

## 2. Cloud와 호환 기준

2026-10-02 확인한 Cloud `contracts/prototype-v1.json`은 **simulated** 모드의 create 전용 계약이다. 필드에 `additionalProperties: false`가 있으므로 새 필드를 몰래 추가하지 않는다. 실제 서비스에서는 아래 공통 식별 의미를 재사용하고 `real` mode·보안·실제 조회 계약·확장 동작을 양쪽 adapter와 함께 검증한다.

| 필드 | 소유자·의미 |
| --- | --- |
| `version` | 외부 wire 계약 버전. 이미지/배포/DB 버전과 별개 |
| `request_id` | Cloud의 상관 ID. Windows가 길이·형식 검사 후 전파 |
| `job_id` | Cloud 업무 작업 ID. Windows 내부 PK와 혼동 금지 |
| `vm_id` | Cloud 논리 VM ID. 고객 소유권은 Cloud가 검증 |
| `external_job_id` | Windows가 영속 접수 시 발급하는 작업 ID |
| `external_vm_id` | Windows가 생성 접수 때 예약하는 안정적인 논리 자원 ID. 실제 VM 존재를 뜻하지 않음 |
| `hyperv_vm_id` | 실제 Hyper-V가 생성한 VM GUID, 확인 전 null. VMM/이관 후 매핑 기록 |
| `vmm_job_id` | VMM의 작업 식별자. 외부 접수 ID와 별개 |
| `attempt` | Cloud의 명시적 업무 시도 번호. HTTP 재전송·Worker 재기동으로 증가시키지 않음 |
| `execution_attempt_id` | Windows 내부 실행 시도 ID. Cloud attempt를 덮어쓰지 않음 |
| `external_idempotency_key` | 같은 Cloud 시도의 외부 접수 중복 방지 키 |
| `event_id` | 불변 완료 이벤트 ID. 재전송 시 동일 ID·동일 본문 |
| `mode` | 실제 Windows endpoint는 real만 수락. simulated 입력 거부 |
| `observed_at` | 실제로 상태를 읽은 UTC 시각. 응답 시각으로 바꾸지 않음 |

현재 공통 create 명령의 필드는 `version`, `request_id`, `job_id`, `vm_id`, `operation`, `attempt`, `external_idempotency_key`, `profile`, `mode`다. `profile`은 `profile_id`, `profile_version`, `snapshot`을 포함한다. snapshot의 CPU·메모리·디스크는 Windows의 등록 catalog와 일치해야 하며 임의 자원 요청으로 해석하지 않는다.

첫 실연동은 승인한 profile에서 image/network를 서버가 결정한다. SSH 공개키·추가 프로필·새 동작 필드가 필요하면 **새 계약 버전에서 명시적으로 정의하고 양쪽을 갱신**한다. 현재 모의 v1이 실제 공개키 전달까지 지원한다고 표시하지 않는다. 버전·업무 동작·DB schema·배포 version은 서로 분리한다.

## 3. Windows API 표면 — 구현 예정

| 경로·방향 | 기능 | 응답·제한 |
| --- | --- | --- |
| `POST /vmm/v1/jobs` | 승인 동작 접수 | 최초 영속 접수 202. 동일 키/동일 본문은 기존 식별자, 다른 본문 409 |
| `GET /vmm/v1/jobs/{external_job_id}` | 관리·접수 복구용 상태 | 읽기 권한·신선도 포함. 페이지의 MSSQL 조회 결정을 대체하지 않음 |
| `GET /vmm/v1/vms/{external_vm_id}` | 관리용 마지막 관측 | 실제 생성 전 pending 구분. 소유하지 않은 대상 거부 |
| `GET /health/live`, `GET /health/ready` | 프로세스·접수 준비 | 원격은 관리 접근 제한. 비밀·상세 경로 미노출. ready는 DB·계약 버전 등 검사 |
| `POST /internal/v1/vmm-completions` (Cloud) | 완료 통지 | 등록한 Cloud endpoint로만 전송, 2xx는 영속 수신 확인을 의미하도록 공동 검증 |

처음 real-v1 경로는 create만 활성화한다. 전원·삭제·사양 변경은 P15에서 명령 schema·권한·상태 정책을 추가한 버전으로 활성화한다. 위 경로가 현재 실제로 열려 있다는 뜻은 아니다.

인증·인가 실패 401/403, 잘못된 필드·버전 400, 숨겨야 할 대상 404, 멱등/상태/자원 충돌 409, 크기 초과 413, 호출 제한 429, 영속 저장 불가 503을 기본 mapping으로 둔다. IIS TLS handshake에서 거부된 호출에는 앱 JSON 오류가 없을 수 있다. 오류 wire는 `{error:{code,message},request_id,mode}`를 기본으로 하고 현재 Cloud parser와 계약 시험한다.

외부 명령 본문에는 스크립트 경로·명령 문자열·UNC·호스트 접속 자격 증명·임의 callback URL을 허용하지 않는다. 만료·불신 인증서 우회 옵션을 운영 설정으로 제공하지 않는다.

## 4. 상태 전이

| Windows 내부 상태 | 외부 표시 | 기준 |
| --- | --- | --- |
| queued | queued | 영속 접수, 실행 전 |
| running | running | 점유 후 단계 수행 |
| verifying | running | 제품·실제 자원 결과 대조 중 |
| recovering / needs_attention | reconciliation_required | 실행 여부 또는 결과가 불명확하거나 수동 복구 필요 |
| succeeded | succeeded | 작업별 실제 결과 검증 완료 |
| failed | failed | 실패 근거·잔여 자원·복구 분류 확정 |

Cloud의 `dispatching`은 Cloud→Windows 전달 단계다. Windows 내부 `verifying` 등을 Cloud enum에 임의 추가하지 않는다. 완료 이벤트는 확정된 succeeded/failed만 보낸다. 불명확 상태를 failed로 만들어 자원을 재할당하지 않는다.

VM 전원, 삭제/존재, 게스트 초기화/SSH 준비, 작업 결과, 통지 전달, 마지막 관측 신선도는 별도 필드다. terminal 상태는 임의 되돌리지 않고 정정이 필요하면 새 이력·대조 결과를 남긴다.

현재 Cloud 규칙대로 **통지가 누락된 경우 SQL 관측값만으로 고객 작업을 자동 성공 확정하지 않는다.** Windows 최종 원장과 outbox를 재조회해 같은 이벤트를 재전송하거나 명시적인 조정 절차를 수행한다.

## 5. 영속 모델과 원자성

| 기록 | 주요 내용 | 업무 작성자 |
| --- | --- | --- |
| jobs | 외부/Cloud ID·operation·state·catalog version·mode·현재 시도·state version | M03 |
| idempotency_records | service principal+환경+키 고유성·정규화 본문 해시·접수 ID | M03 |
| execution_attempts / job_steps | 실행 의도·invocation·제출 전후·제품 Job ID·관측·실패 | M03 |
| resource_leases / reservations | VM별 점유·generation·심박·용량/주소 예약 | M03 |
| vm_bindings | Cloud VM↔Windows VM↔Hyper-V/VMM ID·환경·backend·삭제 tombstone | M04 |
| vm_observations | 실제 상태·사양·게스트 준비·source·observed_at | M06 |
| completion_outbox | 불변 payload·event ID·delivery state·횟수·다음 전송 시각 | 생성 M03, 전달 M08 |
| audit_events | 행위자·허용/거부·대상·변경 전후·reason·상관 ID | M03/M04/M09의 업무 흐름 |
| status_read_model | 승인된 필드·job/source version·관측 시각 | M13, 앱 투영 선택 시 |

접수 transaction은 멱등 확인·job·바인딩 예약·audit를 함께 저장한다. 실행 전에는 `submit_intent`를 기록하고 transaction을 종료한다. 제품 호출 후 참조를 저장한다. 완료 transaction은 job 결과·바인딩/관측·audit·outbox를 함께 저장한다. 같은 앱 DB에 있는 조회 투영은 함께 갱신하고, 다른 조회 원천은 source version과 지연을 명시한다.

두 Worker가 같은 자원을 변경하지 못하도록 DB 조건부 갱신·고유 제약·lease generation을 사용한다. **lease 만료는 기존 PowerShell/VMM 실행이 끝났다는 증거가 아니다.** 재점유 시 먼저 실행 프로세스·제출 의도·VMM Job·VM 표식을 대조하고 기존 실행 종료가 불명확하면 추가 변경을 막는다. generation은 오래된 DB 쓰기를 막지만 제품의 외부 부작용까지 취소해 주지는 않는다.

DB 자동 재시도 범위에 cmdlet 호출을 넣지 않는다. 같은 요청으로 중복 생성되지 않는 효과를 목표로 검증하지만, 분산 환경의 절대적인 exactly-once 실행을 약속하지 않는다. 제출 후 연결 단절은 recovering으로 두고 재조회한다.

## 6. PowerShell bridge

고정된 64비트 `powershell.exe`를 `UseShellExecute=false`로 실행하고 `-NoProfile -NonInteractive -File <trusted-entrypoint>`만 지정한다. 요청값은 command line에 붙이지 않는다. 스크립트 실행 정책은 배포 정책을 따르며 임의 Bypass를 기본으로 넣지 않는다.

stdin JSON에는 `protocol_version`, `invocation_id`, 내부 `external_job_id`, 승인 `operation`, 허용 대상 ID, 검증한 실행 계획, 상대 작업 제한을 둔다. 비밀은 요청에 넣지 않고 서비스 신원·제한된 비밀 참조로 해결한다. stdout은 단일 결과 envelope, stderr는 제한·마스킹한 진단이다. 제품 객체는 필요한 원시 필드로 변환하며 PSObject 전체를 직렬화하지 않는다.

bridge는 UTF-8 인코딩·JSON 깊이·입출력 크기·종료 코드·child 프로세스 수명·동시 stdout/stderr drain을 시험한다. .ps1/.psm1은 Windows PowerShell 5.1의 비ASCII 해석을 고려해 UTF-8 BOM을 사용하고 wire JSON은 BOM 없는 UTF-8로 통일한다.

프로세스 timeout/강제 종료가 이미 제출한 VMM Job을 취소했다고 가정하지 않는다. `submitted`·`not_submitted`·`unknown` 결과를 구분하고 M03 복구로 넘긴다. 작업 중단 기능은 제품별 안전한 중단이 검증된 동작에서만 제공한다.

## 7. MSSQL 조회와 통지

G06은 VMM 내부 DB의 공식 읽기 계약과 별도 앱 상태 DB를 비교해 결정한다. 별도 앱 DB를 택하면 버전 있는 view 또는 제한된 조회 객체만 Linux 읽기 계정에 노출한다. 계정은 서버 전체·VMM DB·다른 앱 테이블을 읽지 못하게 한다. **VMM 내부 DB를 선택해도 쓰기·커스텀 테이블·trigger를 추가하지 않는다.**

조회 결과는 외부 ID·실제 VM 매핑·operation·상태·관측 시각·원천 version·오류 요약을 포함한다. 완료 event는 Cloud의 기존 `event_id`, `job_id`, `external_job_id`, `external_vm_id`, `attempt`, `status`, `observed_at`, `error_code`, `mode`를 기반으로 공동 확정한다. version 필드를 추가해야 한다면 기존 strict parser와 함께 변경한다.

outbox는 최소 1회 전달을 전제로 같은 event를 재전송한다. 네트워크 오류·408·429·일시적 5xx는 제한된 지수 지연+jitter, 영구 4xx·인증서 오류·본문 충돌은 보류·경보로 처리한다. 401/403을 빠르게 반복하지 않는다. callback ACK가 유실돼도 VM 작업을 다시 실행하지 않는다. HTTP 클라이언트 구현과 보존·횟수 기본안은 [운영 규칙](logging-and-operations.md)을 따른다.
