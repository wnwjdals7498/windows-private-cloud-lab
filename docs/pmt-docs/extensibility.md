# 기능 확장 방식

[모듈](modules.md) · [계약](contracts.md) · 방향: 실제로 달라질 경계만 교체 가능하게 둔다.

## 확장 지점

| 늘어나는 것 | 변경 위치 | 유지할 계약 | 도입 조건·검증 |
| --- | --- | --- | --- |
| VM 동작 | M04 operation handler, M05 backend, wire schema | 접수·멱등·상태·실제 확인·통지 | 권한·전원 상태·충돌·실패/복구 시험 |
| 사양/이미지 | M01 catalog, M12 image manifest | 불변 profile/image version·서버 검증 | 신규 생성에 적용, 기존 VM 의미를 소급 변경하지 않음 |
| Hyper-V→VMM | M05 IVmBackend·IInventoryReader 구현 | Windows 논리 ID·실행 결과·오류 | 바인딩 backend를 명시. 설정 오류 시 다른 backend로 자동 우회 금지 |
| Windows 실행 대상 증가 | M01 대상 registry·M05 대상 선택 | 허용 대상·서비스 신원·제품 ID | 자원·동시성·타임아웃을 대상별 검증 |
| 저장소 provider | M07 port 구현·migration·점유 SQL | job ID·멱등 키·state version·outbox | 실제 DB별 동시성·복원·이관 시험 |
| MSSQL 조회 객체 | M13 versioned projection | 외부 ID·시각·원천 version·권한 | G06 변경 절차와 Cloud StatusReader 계약 시험 |
| Cloud 소비자 | M08 transport·서비스 registry | 불변 event·ACK·중복 방지 | 새 소비자별 권한·출발지·목적지·보관 정책 |
| 로그/추적 backend | M10 sink/exporter | 공통 event·correlation·민감정보 제외 | 수집기 장애가 VM 작업을 중복 실행하지 않음 |
| Lab 구성 자동화 | M11 역할 모듈 | 점검→변경→검증→복구 결과 | 새 역할의 precondition·실행 권한·가동 모드 검사 |

## 새 기능 추가 절차

1. 기존 계획 ID 또는 새 범위를 정하고 module owner·사용자 효과·제외 범위를 적는다.
2. 입력·권한·현재 상태별 허용·원자성·timeout·정리 정책을 먼저 정의한다.
3. 외부 wire 변경이 있으면 버전·호환성 차이·Cloud adapter 변경을 문서화한다.
4. M04에 기능별 handler, 필요한 M05/M12 구현을 추가한다. 공통 분기 하나를 무한히 늘리지 않는다.
5. Jobs의 영속 단계·오류·reconciliation을 연결한다. 상태 머신을 기능마다 새로 만들지 않는다.
6. 실제 적용 결과·audit·outbox·조회 모델을 연결한다.
7. RF 정상/오류/중복/복구 fixture와 의미 있는 시험을 추가한다.
8. feature capability와 배포 설정으로 노출하고 관련 계획·문서·PMT 기록을 갱신한다.

완료란 endpoint가 생긴 것이 아니라 허용 상태·실패·중복·복구·실제 결과가 검증된 상태다.

## interface·plugin 규칙

- .NET DI의 명시적인 등록으로 구현을 고른다. 초기에는 임의 DLL 탐색·동적 스크립트 업로드·외부 플러그인 실행 기능을 만들지 않는다.
- `IVmBackend`, `IInventoryReader`, `IJobStore`, `IResourceLeaseStore`, `ICompletionTransport`, `IImageProvisioner`는 실제 I/O 교체 경계다. 단순 계산 클래스마다 인터페이스를 만들지 않는다.
- backend는 지원 동작과 제약을 capability로 제공한다. 지원하지 않는 동작은 접수 전에 거부한다.
- `CanResizeWhileRunning` 같은 능력은 제품·게스트·현재 상태까지 확인한다. UI 버튼을 숨기는 것으로 서버 정책을 대신하지 않는다.
- 환경별 조립은 시작할 때 검증한다. real 환경에서 mock adapter가 등록됐거나 VMM 관리 VM에 Hyper-V 변경 adapter가 선택됐으면 시작 실패로 처리한다.

## 버전과 migration

외부 wire·PowerShell bridge·DB schema·profile/image·설정·배포 release version을 별도 관리한다. 요청에 적용할 catalog version과 실행 계획은 접수 시 snapshot으로 고정한다. 진행 중 작업이 배포 이후 다른 이미지·스위치·정리 정책으로 바뀌면 안 된다.

DB는 확장→전환→정리 순서를 기본으로 한다. 먼저 호환 필드·조회 객체를 추가하고 새 코드 적용·대조를 끝낸 뒤 이전 필드를 제거한다. 진행 작업과 소비자가 존재하는 버전은 즉시 제거하지 않는다. API/Worker가 지원하지 않는 schema이면 readiness를 실패시킨다.

strict JSON schema에 optional 필드를 추가하는 것도 구버전 소비자에게는 파괴적 변경일 수 있다. Cloud 계약 fixture·parser로 실제 호환성을 판정한다. 같은 의미의 필드를 camelCase/snake_case 별칭으로 무한히 늘리지 않는다.

## Worker를 늘릴 때

처음에는 한 Worker·동시 변경 1개로 시작한다. Worker를 늘리기 전에 다음을 충족한다.

| 조건 | 필요한 변화 |
| --- | --- |
| 공유 원장 | 파일 SQLite 공유 대신 서버 DB 등 검증된 동시 저장 |
| 원자적 점유 | provider별 claim·lease generation·조건부 commit |
| 실제 자원 충돌 | VM/디스크/주소/용량별 예약·이전 실행 종료 확인 |
| 작업 재개 | 중단된 invocation·VMM Job을 새 Worker가 추적 |
| 전체 상한 | 프로세스별 제한 외에 환경/Compute/Storage별 전역 제한 |
| 관측 | worker instance·queue age·lock contention·중복 시도 지표 |

처리량 증가만으로 microservice를 나누지 않는다. 신원·배포 주기·장애 격리·확장 단위가 실제로 다를 때 분리한다. 이때 기존 DB 테이블 공유를 서비스 간 계약으로 영구 유지하지 않고 versioned 메시지/조회 경계를 만든다.

## 변경을 위해 남겨둘 문서

새 결정에는 이유·대안·변경 영향·migration·복구·검증·관련 PMT ID를 기록한다. 이전 결정을 지우지 않는다. [기술 선정](technology-stack.md)의 기본안과 [구조](architecture.md)의 tree를 함께 갱신한다. 현재 제안을 사용자 승인 이력으로 등록하지 않는다.
