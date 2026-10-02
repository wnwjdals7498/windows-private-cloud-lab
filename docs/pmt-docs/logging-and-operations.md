# 운영 로깅·저장·관리

[모듈](modules.md) · [동작 레퍼런스](reference-flows.md) · 아래 보존 기간·용량·재시도 수치는 측정 전 추천 기본값이다. G01/G09/G10과 실측으로 조정한다.

## 1. 기록을 분리한다

| 기록 | 목적 | 저장 위치·소유 | 로그 삭제와의 관계 |
| --- | --- | --- | --- |
| 진단 이벤트 | 어떤 단계·오류가 발생했는지 분석 | JSONL, API/Worker/운영 스크립트별 파일 | 기간·용량에 따라 회수 가능 |
| 작업 원장 | 접수·실행 의도·실제 결과·재시작 복구 | 앱 DB, M03/M04/M07 | 일반 로그 정리로 삭제 금지 |
| 감사 원장 | 누가 무엇을 요청·거부·변경했는지 | 앱 DB `audit_events` | 업무 변경과 함께 영속화, 별도 보존 |
| 완료 통지함 | 유실·중복을 견디는 완료 전달 | 앱 DB outbox, M08 | 미전달/충돌 event 자동 삭제 금지 |
| 제품 로그 | IIS·Windows·Hyper-V·VMM·SQL 원인 조사 | 제품 고유 로그·Windows Event Log | 원본 보존 설정을 별도로 확인 |
| 증거 | 요구사항·실제 자원·복원 결과의 재현 | runtime evidence, 공개본만 문서 | 실행 ID·manifest·hash로 묶음 |
| 메트릭·추적 | 자원·지연·오류율·모듈별 흐름 | Activity/Meter·선택 OTel exporter | 고유 ID를 metric label로 사용하지 않음 |

진단 로그가 유실됐다고 완료한 VM을 다시 만들지 않는다. 반대로 로그에 성공 문장이 있어도 원장·실제 결과 검증이 없으면 작업 성공으로 처리하지 않는다.

## 2. 공통 로그 형식

앱 코드는 `ILogger`에 구조화 이벤트를 보내고 Serilog가 JSONL로 저장한다. File sink는 시간·크기에 따른 파일 순환을 지원한다. 원장과 감사 기록은 별도 DB transaction이다. [Serilog 공식 File sink](https://github.com/serilog/serilog-sinks-file)

| 필드 | 의미·규칙 |
| --- | --- |
| timestamp_utc, level, event | UTC 시각, severity, 안정적인 event 이름 |
| service, instance_id, module, release | API/Worker/ops 구분과 배포·실행원 |
| environment_id, mode | 실습 환경·real/mock 구분. 고객 식별 정보는 불필요하게 넣지 않음 |
| request_id, job_id, external_job_id, attempt | Cloud 상관·업무 시도·Windows 작업 연결 |
| invocation_id, vmm_job_id | 실행 프로세스와 제품 작업, 필요한 경우만 |
| external_vm_id, trace_id, span_id | 논리 VM·진단 흐름 연결, 해당 event에 필요한 값만 |
| stage, outcome, duration_ms, error_code | 단계·판정·측정 시간·안정 오류 코드 |
| observation_version, observed_at | 상태 관측·최신성 대조 |

주요 event는 `command.accepted`, `command.rejected`, `job.claimed`, `execution.submitting`, `execution.submitted`, `execution.unknown`, `vm.observed`, `job.completed`, `cleanup.required`, `completion.retry_scheduled`, `completion.delivered`, `security.denied`, `maintenance.started`, `retention.completed`를 기본으로 한다. event 이름은 메시지 문구와 별개로 고정한다.

성공 접수/완료는 Information, 일시 오류·stale·재시도는 Warning, 실패·복구 불가·감사 쓰기 장애는 Error로 기록한다. Debug/Trace는 기본 비활성화하고 문제 조사 중 제한된 범위·시간에만 켠다.

## 3. 저장·순환·보존 기본값

| 대상 | 추천 상한/기간 | 회수 규칙 |
| --- | --- | --- |
| API/Worker JSONL | 각각 최대 30일·1GiB, 파일 32MiB 또는 UTC 날짜 전환 시 순환 | 세 조건 중 먼저 도달한 기준 적용. 30일 최소 보존을 보장하는 값은 아님 |
| ops JSONL | 최대 30일·512MiB | 종료된 실행 로그부터 회수 |
| 한시적 Debug | 최대 7일·256MiB, 별도 예산 | 자동 종료 시각과 회수 기록 |
| IIS 접근 로그 | 최대 14일·512MiB | 선택한 필드만, 내부 주소 등 접근 제한·공개 제외 |
| 작업 상세·전달 완료 outbox | terminal 이후 180일 온라인 보관 제안 | 미완료·재확인·미전달·분쟁 기록은 회수 제외. 삭제 전 archive/재조회 정책 검증 |
| 감사 | 180일 온라인 보관 제안 | 별도 보존 계정이 archive/회수. 서비스 계정의 임의 삭제 금지 |
| 멱등/VM 삭제 tombstone | 자동 만료 없음 | 작은 식별·hash·최종 결과 참조 유지. 늦은 재전송으로 자원 재생성 방지 |
| 비공개 실행 증거 | 일반 시험 90일, 최종 인수/복원 증거는 release 수명 동안 | 개인정보·비밀 제거 후 보존. 실제 저장 여유와 원천 정책에 맞춰 조정 |
| 임시 staging | 완료 후 24시간 내 회수 | 실행 중·recovering 참조 폴더 제외, 루트·소유 표식 확인 |

API/Worker의 파일은 instance별로 분리한다. 여러 프로세스가 같은 파일에 동시에 쓰지 않는다. 날짜/크기 순환·파일 수 제한·전체 예산을 모두 설정한다. **파일 개수 제한을 보존 일수로 해석하지 않는다.** 세부 상한의 총합과 export/archive 임시 공간까지 P01 디스크 예산에 포함한다.

회수기는 하루 1회 실행하는 유지보수 작업을 기본으로 한다. 저장량·나이·보존 예외를 먼저 조회하고 삭제 계획을 기록한다. 활성 파일을 압축/삭제하지 않는다. 압축본·내보내기 파일도 용량에 합산하며 삭제는 자체 관리 루트·승인 대상만 LiteralPath로 처리한다.

로그 예산 80% 경고, 90% 긴급 경고를 제안한다. 회수 대상이 없는데 원장/실행용 공간이 부족하면 신규 변경 접수를 503으로 막고 운영자에게 원인·필요 공간을 표시한다. VM·백업·미전달 원장을 자동 삭제해서 공간을 만들지 않는다.

## 4. 쓰기 장애와 내구성

- API는 접수 원장·필수 감사 commit에 실패하면 202를 반환하지 않는다.
- Worker는 외부 변경 전에 실행 의도를 저장한다. 저장 실패 시 새 부작용을 실행하지 않는다.
- 외부 작업 제출 후 DB가 끊기면 새 제출을 멈추고 recovering으로 복구한다. 실행 중 작업의 존재를 무시하거나 자동 보상 삭제하지 않는다.
- 진단 JSONL 실패는 손실 계수·health 경고와 Windows Event Log fallback으로 알린다. Event Log source는 배포자가 미리 준비하며 앱이 관리자 권한으로 만들지 않는다.
- 일반 진단용 async buffer를 쓰면 크기·drop 정책을 명시하고 drop을 계측한다. 필수 감사·작업·통지를 이 buffer에만 넣지 않는다.
- 자기 진단 sink가 실패한 sink에 다시 로그를 쓰며 무한 반복하지 않게 한다. 종료 시 flush는 제한 시간을 두고 원장 상태를 보호한다.

## 5. 민감정보와 접근권한

비밀번호·토큰·쿠키·개인키·전체 connection string·SSH 비밀키·전체 HTTP 본문·전체 PowerShell 인수·profile에 포함된 비밀을 기록하지 않는다. 서비스 진단에는 실제 IP/UNC 대신 관리 대상 별칭·불변 ID를 기본으로 쓴다.

IIS·제품 로그에는 내부 주소가 포함될 수 있으므로 접근 제한된 원본으로 분류한다. 공개 증거에는 별도 마스킹을 수행한다. 마스킹 규칙은 allowlist 기반이며 정규식 치환만으로 비밀 제거를 보장한다고 보지 않는다. export 후 키·인증서·사용자 정보 검사를 수행한다.

| 주체 | 권한 |
| --- | --- |
| API 계정 | 자체 로그 쓰기, 인증서 키 최소 읽기, 접수/필요 조회 권한 |
| Worker 계정 | 자체 로그·작업 상태·outbox, 필요한 제품 관리 권한 |
| 운영자 | 허용된 진단·감사 조회, 승인된 장애 조치 |
| 배포 계정 | migration·서비스 설정·파일 ACL. 평상시 앱 신원과 분리 |
| 보존/백업 계정 | archive·검증된 회수·복원. 앱의 임의 감사 삭제와 분리 |
| Linux API 읽기 계정 | G06에서 정한 SQL 객체만 SELECT, 다른 데이터 접근 거부 |

로컬 로그·DB가 관리자 변경에 절대 불변이라고 주장하지 않는다. 변조 탐지·독립 보존이 필요해지면 별도 보관 대상과 권한 경계·무결성 manifest를 추가한다.

## 6. 통지·복구 운영 기본값

| 항목 | 초기 제안 | 조정 근거 |
| --- | --- | --- |
| VM 변경 동시성 | 환경 전체 1, VM별 1 | 메모리·IO·제품 작업 부하 실측 |
| 작업/통지 탐색 | 2초 간격, 빈 큐에서 backoff | DB 부하·접수 지연 |
| Worker heartbeat | 10초, lease 60초 후보 | 중단 탐지 시험. 만료 후 실제 실행 종료 검증은 별도 필수 |
| 완료 HTTP timeout | 한 번 10초 | 연결·TLS·수신 commit 실측 |
| 자동 통지 재시도 | 5초부터 지수 지연+jitter, 최대 5분 간격·10회 | transient만. 소진 시 보류·경보, outbox 보존 |
| VM 동작 deadline | 동작별 필수 설정, 임의 공통값 없음 | 이미지 복사·기동·종료·이관 각각 측정 |

HTTP client의 자동 재시도와 outbox 재시도를 겹쳐 요청 횟수가 곱해지지 않게 한다. `Retry-After`는 상한 안에서 존중한다. 보류 통지 재처리는 원래 event ID·본문으로 수행하고 작업을 새로 만들지 않는다. [Microsoft HTTP client](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-requests)

경보 대상은 oldest queued age, recovering job, 미전달 outbox age, Worker heartbeat, SQL 불가, 관측 stale, 실제 자원 불일치, 여유 디스크, 로그 drop, 인증서 만료 30일 이내다. 인증서 1년/30일 갱신 원칙은 기존 결정이며 실제 신뢰 체인·폐기 배포는 G05로 검증한다.

## 7. 관측과 관리자 표시

M06이 CPU·RAM·디스크·SMB·VM 상태를 수집하고 M10이 작업 지연·오류·큐·재시도를 계측한다. 단위·관측 위치·수집 시각을 포함한다. L0 RAM에 L2 할당량을 다시 더하지 않는다. high-cardinality job/VM ID는 로그·trace에 두고 metric label에는 넣지 않는다.

초기에는 로컬 JSONL·Event Log·필요한 counter를 사용한다. G09 이후 OpenTelemetry exporter/collector를 연결하되 수집기는 업무 완료의 필수 동기 의존성이 아니다. exporter 연결·buffer·전송 실패를 제한하고 traces 보존·용량을 별도 예산으로 둔다. [Microsoft OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel)

Cloud 관리자 화면에는 정상/실패/실행 중 외에 미관측·오래된 관측·수동 조치·통지 대기를 표시한다. 진단 원문·내부 비밀 대신 안전한 오류 코드·요약·상관 ID·마지막 관측을 제공한다.

## 8. 배포·업데이트·복원

배포 manifest에 release ID, Git revision, SDK/runtime, NuGet/PS 모듈, 계약·DB·bridge version, scripts/image hash를 포함한다. 비밀은 포함하지 않는다. API와 Worker는 같은 계약을 지원하는 조합으로 배포한다.

배포 순서는 사전 상태/backup 확인→새 접수 정지→진행 작업 정리 또는 명시적 인계→호환 migration→Worker→API→readiness/인증/접수·조회 점검→접수 재개다. destructive migration은 rollback에 필요한 데이터 보존을 먼저 확인한다. 파일 rollback이 DB rollback까지 해결한다고 가정하지 않는다.

SQLite backup은 일관된 DB backup/정지 절차를 사용하고 WAL 파일이 있는 실행 중 `.db` 하나만 복사하지 않는다. SQL은 제품 backup·복원 검증을 따른다. job·멱등·VM binding·audit·outbox·catalog snapshot을 함께 보존한다.

복원 시 worker를 곧바로 실행하지 않는다. 실제 VM/VMM Job과 복원된 원장의 시차·완료 통지 여부를 먼저 대조한다. 오래된 DB backup 복원이 이미 생성한 VM을 다시 만들지 않게 recovering으로 검증한다. AD·SQL/VMM·이미지·고객 데이터·인증서 복원 순서는 P17에서 실제 시험한다.

물리 호스트의 고객 VHDX와 같은 디스크에 있는 복구본은 물리 디스크 장애 대비 복구본으로 표시하지 않는다. S2D 게스트 디스크의 호스트 snapshot은 백업 수단으로 사용하지 않는다. [Microsoft S2D 게스트 제한](https://learn.microsoft.com/en-us/windows-server/storage/storage-spaces/storage-spaces-direct-in-vm)

## 9. 운영 결과 기록

운영 조치에는 시각·행위자·대상·사유·관련 job/run ID·적용 전후·복구 가능성·검증 결과를 남긴다. 공개 인수 증거는 환경 범위·버전·실행 조건·기대/실제 결과·원본 hash·한계를 포함한다. 정기 자동화는 향후 사용자 요청이나 운영 요구에 따라 구현하며 이 문서 작성만으로 예약 작업을 등록하지 않는다.
