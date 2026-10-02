# 기능별 모듈 책임

모듈은 코드 책임의 경계이며 각각 별도 서비스나 프로젝트를 만들라는 뜻은 아니다. ID는 구현 계획·참조 흐름·확장 문서에서 공통 사용한다. [구조](architecture.md)

## 책임·입출력·소유권

| ID | 모듈 / 주 위치 | 기능·주 입력 → 결과 | 소유 범위 / 하지 않는 일 |
| --- | --- | --- | --- |
| M01 | Configuration/Catalog / Application·Domain | 환경·profile/image/network ID → 검증된 불변 실행 계획·catalog version | 허용 목록·설정 검사. 요청값으로 호스트 설정을 덮어쓰지 않음 |
| M02 | Admission / API | 인증한 명령 DTO → 접수/중복/거부 응답 | HTTP 파싱·크기·버전·route·응답. VM 변경과 상태 원장 직접 갱신 없음 |
| M03 | Jobs / Application·Domain | 명령 → 접수·점유·단계·시도·최종 결과·복구 계획 | `jobs`, `job_steps`, `execution_attempts`, 멱등 기록·resource lease·예약. 작업 상태의 유일한 업무 작성자 |
| M04 | VmLifecycle / Application·Domain | 동작·현재 관측·profile → 실행 단계·허용 여부·검증 결과 | create/start/stop/restart/delete/resize/network 정책·`vm_bindings`. 고객 로그인·소유권 DB 없음 |
| M05 | Execution / Infrastructure·PowerShell | 검증된 계획·invocation → 제품 Job/VM 참조·실행 결과 | Hyper-V/VMM adapter·bridge. 성공 HTTP 응답·SQL 테이블 임의 수정 없음 |
| M06 | Inventory / Application | host/VM 대상 → 제품 원시값·정규화 상태·observed_at·freshness | `vm_observations`, 호스트 관측. 관측만으로 Cloud 작업 성공을 변경하지 않음 |
| M07 | Persistence / Infrastructure | 저장 port·transaction → 영속 기록·조건부 갱신 | DB 기술·migration·제약·점유 SQL. 업무 상태 정책은 M03/M04에 둠 |
| M08 | Completion / Application·Infrastructure | outbox event → 전달·재시도·보류 | `completion_outbox` 전달 필드. VM 동작을 재실행하거나 결과 본문을 변조하지 않음 |
| M09 | Security / API·Infrastructure | 서비스 인증서·대상·요청 → service principal·허용/거부 | 신원·권한·인증서·비밀 참조. 회원 인증·JWT 발급 없음 |
| M10 | Observability / Infrastructure·tools | 타입 이벤트·metric·evidence → 진단 로그·감사 기록·증거 | 수집·마스킹·회수·경보. 로그를 작업 상태 DB로 쓰지 않음 |
| M11 | LabOperations / PowerShell entrypoints | 운영자 명령·lab profile → L0/AD/SMB/Cluster 구성·점검·이관 | Windows8/Linux8/4+4 모드·부팅·보존 전환·복구. 고객 API 호출 경로에 노출하지 않음 |
| M12 | ImageProvisioning / PowerShell·Application | imageId·복제 계획·초기화 정책 → 독립 VM 디스크·게스트 초기화 상태 | 이미지 manifest·해시·계정/공개키 주입·신원 초기화. 고객 비밀키 보관 없음 |
| M13 | ReadModels / Application·Persistence | 검증된 작업/관측 → 승인된 MSSQL 읽기 모델 | 앱 투영을 선택한 경우 전용 view/table. VMM 내부 DB 수정·Cloud MySQL 수정 없음 |

감사는 각 업무 변경과 같은 트랜잭션에서 `audit_events`에 추가한다. M10은 event 형식·조회·보존을 담당하며 감사 목적으로 별도 느린 HTTP 호출을 트랜잭션 안에서 실행하지 않는다.

## 모듈 사이 호출

| 호출자 → 대상 | port/메서드 방향 | 핵심 제한 |
| --- | --- | --- |
| M02 → M09 | AuthenticateService / AuthorizeCommand | 인증서 유효성과 작업 권한을 분리 |
| M02 → M03 | AcceptCommandAsync | 영속 접수 결과만 반환 |
| M03 → M01 | ResolveExecutionPlan | 프로필 snapshot과 등록값 불일치 거부 |
| M03 → M04 | PlanOperation / VerifyOperation / PlanCleanup | 외부 HTTP DTO를 그대로 cmdlet 인수로 사용하지 않음 |
| M03 → M07 | IJobStore / IUnitOfWork / IResourceLeaseStore | 트랜잭션은 짧게, 네트워크·cmdlet 호출 제외 |
| M04 → M05 | IVmBackend.Submit / Observe / Reconcile | VMM 관리 중에는 VMM 변경 경로만 사용 |
| M04/M06 → M05 | IInventoryReader | 제품의 실제 상태를 재조회 |
| M04 → M12 | IImageProvisioner | backend에 맞는 이미지 준비·초기화 |
| M03 → M08 | 최종 결과와 outbox payload를 함께 저장 | 생성은 M03 트랜잭션, 전달 갱신은 M08 |
| M08 → Cloud | ICompletionTransport | HTTPS·mTLS·등록 endpoint. 요청 본문의 callback URL 사용 금지 |
| M03/M06 → M13 | ProjectCommittedResult / ProjectObservation | 각 source version·관측 시각 보존 |
| 전 모듈 → M10 | ILogger / Activity / Meter / 감사 DTO | 비밀·전체 요청·제품 객체 통째 기록 금지 |
| M11 → M05/M12의 PowerShell 공용 함수 | 명시된 운영 entrypoint | API/Worker와 유지보수 충돌을 차단한 뒤 실행 |

이 port 이름은 구현 기준안이다. 모든 인터페이스를 먼저 빈 구현으로 만들지 않는다. 첫 구현에서 실제 호출되는 경계만 추가하고 [의존 방향](architecture.md)을 유지한다.

## 동작별 분리 범위

| 동작 | M04 정책 | M05 실행 | M06/M12 완료 확인 |
| --- | --- | --- | --- |
| 생성 | 이미지·사양·주소·용량 검증, 실행 표식 예약 | 이미지 준비·VM 생성·사양/NIC·기동 | 실제 VM 존재·사양·전원, 별도 게스트 준비 |
| 조회 | 대상과 범위 검사 | 제품 읽기 | 원시 상태·정규화·시각·오래됨 |
| 시작 | 이미 실행 중·충돌 작업 처리 | 승인 backend의 기동 | 실제 전원 상태 |
| 중지 | 정상 종료·제한 시간·강제 종료 정책 | 정상 종료 요청 | 실제 Off, 시간 초과는 별도 오류 |
| 재시작 | 재시작 허용·진행 작업 충돌 | 게스트/제품 재시작 | 부팅 시각·접근 회복, Running 하나로 완료 금지 |
| 삭제 | 바인딩·파일 참조·보존 정책 | VM 등록 제거·정책상 파일 정리 | 실제 제거·잔여 자원·주소 해제 |
| 사양 변경 | 전원 상태별 CPU/RAM·디스크 확장·네트워크 정책 | 승인 가능한 제품 변경 | 실제 사양·게스트 용량·SSH 재접속 |
| 인프라 축소 | M11이 작업 접수 중단·목록·복구 지점 관리 | 운영 이관 명령 | 파일/게스트 데이터·바인딩·가동 대수 대조 |

## 구현 단계 매핑

| 단계 | 주 모듈 | 먼저 연결할 결과 |
| --- | --- | --- |
| P00 | M01, M03, M09, M10 | 설정·명령/오류 계약·증거 형식 |
| P01 | M01, M10, M11 | 읽기 전용 실호스트 점검 |
| P02 | M01, M09, M11 | 네트워크·신원·접근 설정 |
| P03 | M05, M11 | 단독 Compute의 Hyper-V |
| P04 | M01, M12 | Rocky 이미지·복제·고유 신원 |
| P05 | M03, M04, M05, M06, M12 | 직접 VM 자동화·재조회·부분 실패 |
| P06 | M01, M02, M03, M05, M07, M09, M10 | IIS 접수·독립 Worker·초기 영속성 |
| P07 | M09, M11 | AD/DNS·시간·서비스 권한 |
| P08 | M05, M06, M11, M12 | VMM 등록·공식 명령·이미지 |
| P09 | M02, M03, M05, M07, M08, M09, M13 | VMM API·작업 DB·SQL 읽기·통지 |
| P10 | M05, M06, M09, M11 | Storage·SMB 권한/암호화 |
| P11 | M05, M06, M10, M11 | Compute 이동·장애·복귀 |
| P12 | M03, M04, M06, M07, M11, M13 | 데이터 보존·바인딩·8→4 전환 |
| P13 | M01, M08, M09, M11, M13 | Cloud 인계·4+4 실제 통신 |
| P14 | M02, M03, M04, M05, M06, M08, M12, M13 | 회원 신청·VM 생성·SSH·정합성 |
| P15 | M03, M04, M05, M06, M08 | 전체 수명주기 |
| P16 | M03, M06, M08, M09, M10, M13 | 복구·관측·인증서 운영 |
| P17 | M01, M07, M10, M11 | 재현·backup/restore·인계 |

P00~P05에는 PowerShell과 계약 문서 중심으로 시작할 수 있다. 이 단계의 M03 정책을 PowerShell 호출 절차로 먼저 확인하더라도 P06 이후 영속 상태 전이의 소유자는 C# Jobs로 통일한다. PowerShell에 별도 큐·통지 재시도·DB 상태 머신을 중복 구현하지 않는다.
