# 기능별 기술 선정

기준일: 2026-10-02 · 아래는 추천 구현 기준이며 설치·선정 게이트의 완료 선언이 아니다. [참조 안내](README.md)

## 핵심 선택

**C#/.NET 10 LTS로 API·작업 제어를 만들고, Windows PowerShell 5.1로 Hyper-V·VMM·Windows 역할을 호출한다.** 웹 실행 환경과 VMM 모듈 실행 환경을 분리해 업데이트·호환성 문제의 영향을 줄인다.

.NET 10은 확인일 기준 지원 중인 LTS이며 종료 예정일은 2028-11-14다. 정확한 SDK·런타임·라이브러리 패치는 첫 구현 때 공식 지원 OS와 함께 검증해 고정한다. 설치 버전 VMM의 PowerShell 요구는 별도로 따른다. [Microsoft 지원 정책](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core), [VMM 요구사항](https://learn.microsoft.com/en-us/system-center/vmm/system-requirements?preserve-view=true&view=sc-vmm-2022)

ASP.NET Core/Worker의 .NET10과 VMM·Windows PowerShell이 요구하는 .NET Framework는 다른 실행 환경이다. 앱 런타임 도입을 이유로 VMM 제품 전제 조건을 제거하거나 대체하지 않는다.

## 기능별 선택·확장 지점

| 기능 | 추천 프레임워크/도구 | 선택 이유·확장 방법 | 최초 단계 |
| --- | --- | --- | --- |
| 설정·기동 검사 | Microsoft.Extensions.Configuration/Options/DI | 타입 설정·시작 시 검증, 역할별 조립 교체 | P00/P06 |
| L0·Windows 역할 점검 | Windows PowerShell 5.1 + 공식 역할 cmdlet | 실제 관리 기능과 직접 대조, 점검/변경 함수를 분리 | P01 |
| 네트워크·AD·SMB·클러스터 | Hyper-V·NetTCPIP·ADDSDeployment·ActiveDirectory·SmbShare·FailoverClusters 등 설치 역할의 공식 모듈 | 역할별 PowerShell 모듈로 묶고 상태 검출 후 변경 | P02/P07/P10/P11 |
| Rocky 이미지 | Hyper-V cmdlet + Rocky 기본 게스트 도구 | 전체 VHDX 복제를 먼저 검증, 초기화 전달은 adapter화 | P04 |
| 작업 API | ASP.NET Core 10 Minimal APIs + IIS Hosting Bundle | 작은 서비스별 route group·DI·인증 적용, HTTP와 업무 규칙 분리 | P06, G04 |
| 서비스 인증 | IIS TLS + ASP.NET Core 인증서 handler | 체인·용도·유효기간·서비스 신원 검사, 발급자/주체 매핑 교체 | P06 |
| 장기 작업 실행 | .NET Worker Service / BackgroundService + Windows Service | IIS 재순환과 독립, 역할별 실행 루프 분리 | P06 |
| 내부 작업 제어 | 명시적 상태 전이·단계 handler·DB 작업 큐 | 시도·복구를 직접 관찰, 향후 전용 broker 연결 가능 | P05/P06 |
| DB 저장 | EF Core 10 + SQLite/SQL Server provider | 업무 저장 port로 숨기고 provider별 migration·점유 SQL 분리 | P06/P09 |
| 초기 영속 DB | SQLite, 로컬 NTFS, 짧은 트랜잭션 | SQL 설치 전 접수 유실 방지, API+단일 Worker 시험용 | P06, G04 |
| VMM 이후 앱 DB | SQL Server의 별도 앱 DB 추천 | VMM 내부 DB와 분리, 원자적 점유·통지·조회 모델 관리 | P09, G04/G06 |
| VMM 실행 | 설치된 VMM PowerShell + Windows PowerShell 5.1 별도 프로세스 | .NET10에서 레거시 모듈을 직접 로드한다고 가정하지 않음 | P08/P09 |
| API 통지 | IHttpClientFactory + 명시적 outbox 재시도 | 인증서·연결 수명 관리, HTTP 재전송과 업무 재실행 분리 | P09 |
| API 문서 | Microsoft.AspNetCore.OpenApi + System.Text.Json | 문서/DTO 계약 점검, snake_case wire 고정 | P06/P09 |
| 오류 응답 | 공통 error DTO, 내부 AppError | 기존 Cloud error 형태에 맞춤. ProblemDetails는 변환 adapter 없이 혼용하지 않음 | P06 |
| 진단 로그 | Microsoft.Extensions.Logging + Serilog + File sink | 구조화 이벤트·일/크기 순환, 업무 코드는 ILogger에만 의존 | P06 |
| 메트릭·추적 | System.Diagnostics Activity/Meter + OpenTelemetry | 수집기를 바꿔도 계측 호출 유지, 중앙 수집은 이후 도입 | P09/P16 |
| Windows 이벤트 | Windows Event Log | 서비스 시작 실패·로그 저장 장애 같은 로컬 운영 신호 | P06 |
| .NET 시험 | xUnit.net v3 + ASP.NET Core WebApplicationFactory | 상태·계약·HTTP 통합 시험, 실제 IIS/mTLS는 별도 실환경 시험 | P06 |
| PowerShell 시험 | Pester 5 + PSScriptAnalyzer | 정책·모의 cmdlet·예외 경로·정적 품질 검사 | P01/P05 |
| 코드·의존성 고정 | .editorconfig, SDK/global.json, 중앙 NuGet 버전·lock, PowerShell 도구 manifest | 같은 도구로 재현, 업데이트 영향 추적 | P00 이후 해당 코드 도입 시 |

IIS 호스팅, Windows Service, OpenAPI, EF Core provider는 Microsoft의 공식 경로다. 이 프로젝트에서의 조합·모듈화는 설계 제안이다. [IIS](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/), [Windows Worker](https://learn.microsoft.com/en-us/dotnet/core/extensions/windows-service), [OpenAPI](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/openapi/overview), [SQL Server provider](https://learn.microsoft.com/en-us/ef/core/providers/sql-server/)

## 선택을 제한하는 이유

| 대안 | 현재 판단 | 도입할 조건 |
| --- | --- | --- |
| IIS에서 PowerShell을 요청마다 직접 실행 | 웹 요청 수명과 작업 완료·복구가 결합되므로 기본안에서 제외 | 별도 Worker가 불가능한 실측 사유가 있으면 작업 영속성부터 재설계 |
| PHP·Node·Python 수신부 | Cloud 언어를 맞출 수 있으나 Windows 서비스·타입 계약·인증서 연동 운영이 추가됨 | G04에서 유지보수 경험·공식 호스팅·모듈 호출 시험이 더 유리할 때 |
| PowerShell 7을 VMM 공통 런타임으로 사용 | VMM 2022 문서의 5.1 기준과 별도 검증 필요 | 설치된 VMM 모듈의 공식 지원·실동작 확인 뒤 adapter 단위로 변경 |
| RabbitMQ·Kafka·Redis 큐 | 현재는 단일 Worker와 이미 필요한 DB로 내구성 제공 | 다중 호스트·처리량·독립 장애 격리 요구가 측정됐을 때 |
| Hangfire·Quartz·워크플로 엔진 | 초기에는 큐/상태/재시도 체계를 두 개 만들지 않음 | 복잡한 일정·장기 분기 요구와 기존 Job 계약 연결 비용 검토 후 |
| MediatR·범용 Generic Repository | 기능별 handler와 필요한 저장 port만 정의 | 실제 반복·조립 복잡도가 드러날 때; 라이브러리 추가가 목표가 아님 |
| Kubernetes·컨테이너화 | Windows 역할·VMM·중첩 Hyper-V를 대체하는 배포 방식으로 두지 않음 | 별도 실행 환경이 목표에 포함될 때 |
| 중앙 로그 서버·SCOM | 로컬 로그·표준 계측부터 구현 | G09의 지표·자원·보존 요구가 확정되면 연결 |
| DSC·Ansible·Packer | 초기 수동 결과와 cmdlet 절차를 먼저 고정 | 여러 호스트의 구성 편차·이미지 재빌드 비용이 커지면 역할 단위 도입 |

## 저장소 선택과 이동

- P06 기본안은 SQLite다. 파일은 로컬 NTFS에 두고 SMB에 공유하지 않는다. API와 Worker는 각각 짧은 트랜잭션을 사용하며 변경 작업 실행은 한 Worker로 제한한다. WAL·busy timeout·디스크 가득 참을 시험한다.
- SQLite 신원 제한은 파일 ACL 단위다. SQL Server처럼 API/Worker별 테이블 권한이 강제된다고 주장하지 않는다. 초기에는 제한된 두 프로세스만 파일 접근을 허용하고, 세부 DB 역할 분리는 SQL Server 단계에서 검증한다.
- P09 기본안은 같은 SQL 인스턴스의 **별도 `LabControl` 앱 DB**다. VMM의 DB·테이블·서비스 계정과 구분한다. 이것은 앱 원장의 추천 위치이며 **Cloud가 읽을 MSSQL 대상에 대한 G06 결정을 대체하지 않는다**.
- SQLite→SQL Server는 ID·상태·원장·통지함·멱등 tombstone을 보존하는 중단 이관이다. 연결 문자열 변경만으로 완료하지 않는다.
- EF Core migration은 provider별로 유지한다. SQLite의 동시성·자료형·스키마 변경 차이를 실제 DB 시험으로 확인한다. 파일 로그·메모리 큐는 영속 작업 DB의 대체물이 아니다. [SQLite 제한](https://learn.microsoft.com/en-us/ef/core/providers/sqlite/limitations), [SQLite 잠금](https://learn.microsoft.com/en-us/dotnet/standard/data/sqlite/database-errors), [provider별 migration](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/providers)

## 패키지 채택 규칙

필요한 단계에서만 의존성을 추가한다. 정확한 버전·라이선스·공식 지원 범위·다운로드 원천·기본 설정 변경을 manifest/lock과 [공식 자료](official-references.md)에 남긴다. PowerShell Gallery의 자동 최신 설치, 부팅 시 런타임 다운로드, 패치 버전을 문서에 영구 고정하는 방식은 피한다. 운영 배포 전 호환성 시험을 통과한 지원 패치로 갱신한다.
