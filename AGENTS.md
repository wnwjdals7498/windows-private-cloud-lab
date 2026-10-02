# 구현 작업 규칙

## 먼저 읽기

- [참조 문서 안내](docs/pmt-docs/README.md)에서 해당 작업 문서만 읽는다. 실행 순서는 [18단계 계획](docs/implementation-plan.md)을 따른다.
- 실제 작업 인계는 [기능 상세 카드](docs/pmt-docs/implementation-details/README.md)와 [Luna 병렬 실행](docs/pmt-docs/implementation-details/parallel-execution.md)을 따른다. 파일 편집 목록을 목적·완료 조건으로 사용하지 않는다.
- 사용자 최신 지시·승인 결정 → PMT 명세 → 해당 참조 문서 순으로 대조한다. 충돌을 발견하면 관련 문서를 함께 수정한다.
- 기존 `docs/plan.md`·UI 목업은 이전 초안이다. 예시·모의·제안·실측을 구분한다.
- 설명은 간단명료하게 쓴다. 기술 사실·제품 지원 정보는 공식 문서만 사용하고 출처·확인일을 남긴다.

## 범위와 기본 방향

- 이 저장소는 Windows 인프라·VM 실행·상태·증거를 소유한다. 회원·권한·MySQL·고객/관리자 화면은 Cloud 저장소가 소유한다.
- 추천 기준은 .NET 10 LTS/ASP.NET Core API, 독립 Windows Worker, Windows PowerShell 5.1 실행 모듈이다. 상세 조건은 [기술 선정](docs/pmt-docs/technology-stack.md)을 따른다.
- 새 기술 선정안은 구현 가이드다. IIS 기술·실제 네트워크·MSSQL 조회 대상 등 기존 사용자 결정 게이트를 문서 작성만으로 통과 처리하지 않는다.
- 인프라8과 Linux8은 동시에 시험하지 않는다. 서비스는 인프라4+Linux4다. 8→4 전환에는 데이터 이관·복원 검증이 필요하다.
- 중첩 고객 VM과 L1 서비스 VM, 실습 성공과 공식 운영 지원을 구분한다.

## 구조·통신

- [폴더/의존 방향](docs/pmt-docs/architecture.md), [모듈 책임](docs/pmt-docs/modules.md)을 지킨다. Domain에 DB·HTTP·PowerShell 의존성을 넣지 않는다.
- API는 인증·검증·영속 접수까지만 한다. VM 변경은 Worker→실행 adapter→고정 PowerShell 경로에서 한다.
- 모듈 내부는 타입 있는 메서드/DTO, API↔Worker는 작업 DB, 외부는 버전 있는 HTTPS JSON 계약으로 연결한다. 내부 모듈마다 HTTP 서비스를 만들지 않는다.
- API↔Cloud는 상호 TLS·등록 출발지를 검증한다. 페이지 조회는 승인된 MSSQL 객체를 사용한다. JWT/HMAC 과거 경로를 복구하지 않는다.
- [계약](docs/pmt-docs/contracts.md)의 Cloud job ID·Windows job ID·VMM Job ID·실제 Hyper-V VM ID를 혼용하지 않는다.
- Cloud 모의 계약의 추가 필드 금지·mode·상태 규칙을 보존한다. 실제 계약 확장은 버전·양쪽 adapter·계약 시험을 함께 변경한다.

## 코드·데이터

- 고정 경로의 스크립트에 검증된 구조화 입력을 전달한다. `Invoke-Expression`, 사용자 입력으로 명령 조립, 임의 저장 경로·스위치 실행을 금지한다.
- 영속 접수 후 응답한다. 멱등 키+본문 해시·DB 고유 제약·VM별 동시 변경 제한을 둔다.
- 외부 호출 중 DB 트랜잭션을 유지하지 않는다. 타임아웃·lease 만료만으로 생성 명령을 재실행하지 않는다. 기존 실행·Job·실제 VM부터 대조한다.
- 완료는 실제 자원 재조회로 판단한다. 전원 상태·SSH 준비·통지 전달 상태를 분리한다.
- VMM 내부 DB를 수정하지 않는다. 앱 DB migration은 배포 단계에서 실행하며 API/Worker 시작 시 자동 변경하지 않는다.
- 비밀·실제 IP·인증서 개인키·런타임 DB·운영 로그·원본 증거는 Git 밖에 둔다.
- 코드 세부 규칙과 시험 범위는 [코딩/검증](docs/pmt-docs/coding-and-testing.md)을 따른다.

## 변경·운영·완료

- 새 기능은 입력→권한→정책→실행→관측→통지→복구→증거를 함께 정의한다. [확장 절차](docs/pmt-docs/extensibility.md)를 따른다.
- 작업을 맡길 때 목적·추가/수정/삭제 행동 범위·goal/non-goal·각 input/output의 의미·Test·Logging·복구·완료 증거를 지정한다. 실제 주소·사양·샘플 값으로 필드의 의미 설명을 대신하지 않는다.
- 병렬 작업은 공유 계약을 먼저 고정한다. 한 모듈의 상태·공유 정의는 한 담당자가 작성하고, 실호스트 변경·PMT·통합은 주 에이전트가 관리한다. 코드 준비와 실호스트 완료를 구분한다.
- 구현할 모듈만 만든다. [목표 tree](docs/pmt-docs/architecture.md)를 빈 프로젝트·미사용 추상화로 한 번에 생성하지 않는다.
- 장애 처리의 기대 동작은 [동작 레퍼런스](docs/pmt-docs/reference-flows.md)와 대조한다.
- 운영 로그·감사 원장·작업 DB는 역할을 분리한다. [로깅/운영](docs/pmt-docs/logging-and-operations.md)의 마스킹·용량·보존·복구 규칙을 적용한다.
- 변경에 필요한 검증만 실행하고 수동/모의/실환경 결과를 구분한다. 파일 작성이나 모의 성공을 VM 기능 완료로 표시하지 않는다.
- PMT가 있는 환경에서는 동일 session으로 resume→start→note/verify→end를 사용한다. `docs/pmt-docs`는 참조 문서이며 PMT 데이터 루트가 아니다.
- 보고는 결과·변경 파일·검증·미검증/남은 조건을 짧게 쓴다. 현재 문서 작업은 배포·인프라 변경·원격 게시를 뜻하지 않는다.
