# 코드 구현과 검증 규칙

[AGENTS.md](../../AGENTS.md) · [통신 계약](contracts.md) · 적용 대상: 앞으로 추가할 코드

## 공통

- 한 변경은 하나의 동작·계약 목적을 갖는다. 큰 동작은 계획의 작업 ID 단위로 나눈다.
- 타입·함수·변수는 의미가 드러나는 영어, 설명·운영 문서는 간결한 한국어를 기본으로 한다.
- 외부 DTO, Domain 모델, DB entity, PowerShell 결과 객체를 분리한다. 계층 간 제품 객체·DbContext·IQueryable을 반환하지 않는다.
- bool 하나로 성공·접수·미관측·부분 실패를 표현하지 않는다. 안정적인 오류 코드와 typed result를 사용한다.
- 시간은 UTC, duration은 monotonic clock을 쓴다. 테스트에서는 clock을 주입한다. bytes/MiB·vCPU·초 등 단위를 필드에 명시한다.
- 현재 작업과 무관한 재작성·의존성 추가·폴더 선생성을 피한다. 생성 예정 tree와 구현 존재 여부를 구분한다.

## C# / ASP.NET Core

1. nullable reference types와 분석기를 활성화하고 신규 경고를 방치하지 않는다. 비동기 메서드는 Async 접미사와 CancellationToken을 사용한다.
2. endpoint는 검증·호출·응답 mapping만 수행한다. 실행 정책은 Application/Domain으로 옮긴다.
3. DI composition root는 API/Worker의 Program 및 등록 확장에 둔다. Domain에서 service locator·정적 전역 상태를 사용하지 않는다.
4. BackgroundService에서 scoped DbContext/서비스를 작업 단위로 만든다. DbContext를 singleton이나 병렬 작업에 공유하지 않는다.
5. HTTP·DB timeout과 업무 deadline을 구분한다. HTTP 취소가 이미 접수한 VM 작업 취소를 뜻하지 않는다.
6. 외부 오류는 whitelist한 코드·safe message로 변환하고 내부 원인은 제한된 진단에 남긴다. 예외를 잡고 성공으로 반환하지 않는다.
7. 외부 JSON은 snake_case·UTC·명확한 null 규칙을 사용한다. 알려지지 않은 명령 필드·enum·버전은 계약대로 거부한다.
8. ILogger에는 구조화 template와 허용 필드만 전달한다. 요청 DTO·connection string·credential·제품 객체 통째 destructuring을 금지한다.
9. HTTP client는 IHttpClientFactory로 구성한다. 작업 POST의 자동 retry/hedging은 기본 해제하고 멱등 정책을 아는 호출부가 재전송을 결정한다.
10. 파일·프로세스·DB·네트워크 사용은 Infrastructure에 둔다. 새 기능이 Domain의 제품별 if 분기를 강요하면 adapter 경계를 다시 점검한다.

## PowerShell

1. VMM/Windows 역할 모듈의 실행 기준은 Windows PowerShell 5.1이다. PS7 전용 문법·cmdlet을 공용 스크립트에 넣지 않는다.
2. 공개 함수는 승인 Verb-Noun, 명시 param·ValidateSet/ValidateRange·입력 타입을 사용한다. 파일명·함수 역할을 맞춘다.
3. `Set-StrictMode`와 실패 전파 정책을 명시하고 필요한 cmdlet에 `-ErrorAction Stop`을 적용한다. `$LASTEXITCODE`는 외부 프로세스에 맞게 확인한다.
4. bridge의 stdout에는 최종 결과만 남긴다. 모든 cmdlet 결과를 캡처하고 진행 출력·비ASCII·깊은 객체가 JSON을 깨지 않게 시험한다.
5. `Invoke-Expression`·임의 scriptblock·명령 문자열 조립·`cmd /c` 경유 파일 삭제를 사용하지 않는다. 작업 이름에서 고정 함수로 dispatch한다.
6. 파일 작업은 정규화한 절대 경로가 승인 루트 안에 있는지 검사하고 LiteralPath를 사용한다. `..`, UNC 우회·reparse point·공유 이미지 참조를 점검한다.
7. 삭제 함수는 VM/요청 소유 표식·실제 참조·보존 정책을 확인한다. 이름 prefix만으로 삭제하지 않는다.
8. 점검과 변경을 분리한다. 운영자용 변경 함수는 가능한 경우 SupportsShouldProcess/WhatIf를 제공하되 WhatIf 성공을 실제 검증으로 표시하지 않는다.
9. PSSession·파일 핸들·임시 파일을 finally로 정리한다. 정리 실패가 최초 오류와 잔여 자원 목록을 지우지 않게 한다.
10. `.ps1/.psm1/.psd1`은 UTF-8 BOM, C#/JSON/Markdown은 UTF-8을 기본으로 한다. .editorconfig/gitattributes에서 인코딩·줄바꿈을 명시한다.

## SQL·원장·설정

- 값은 parameter binding으로 전달한다. 테이블/열 이름을 사용자 입력에서 조립하지 않는다.
- EF Core InMemory provider로 실제 SQL 동시성·고유 제약·rollback 통과를 주장하지 않는다. SQLite와 SQL Server 각각 시험한다.
- claim·lease·state version은 조건부 갱신으로 검사하고 변경 행 수를 확인한다. 외부 호출을 재시도 transaction에 넣지 않는다.
- schema migration은 별도 배포 계정·단일 실행원으로 실행한다. API/Worker 운영 계정에 DDL 권한을 주지 않는다.
- VMM DB·앱 DB·Cloud MySQL의 소유 경계를 보존한다. 테스트 편의를 위해 내부 제품 DB에 쓰지 않는다.
- 설정 우선순위는 기본값→역할/환경의 제한된 파일→허용한 배포 override로 고정한다. 웹 요청으로 backend·DB·callback 목적지를 바꾸지 않는다.
- 필수 설정 누락·real/mock 혼합·지원하지 않는 schema·읽을 수 없는 키는 readiness 실패로 드러낸다. 비밀 기본값으로 동작시키지 않는다.
- 참조 경로·신원 같은 값은 allowlist로 검증한다. 비밀은 인증서 저장소·승인된 비밀 저장 방식의 참조만 설정에 둔다.

## 검증 수준

| 수준 | 도구·범위 | 필수 확인 |
| --- | --- | --- |
| 정적 | dotnet build/format, PSScriptAnalyzer | 타입·분석기·PS5.1 문법·민감정보/잘못된 경로 |
| 단위 | xUnit / Pester | 상태 전이·정책·허용 입력·오류 mapping·소유 자원 정리 |
| 계약 | 독립 fixture·schema·Cloud parser·bridge 시험 | field·ID·enum·null·strict 추가 필드·동일 event 충돌 |
| HTTP 통합 | WebApplicationFactory | 영속 접수·인증/인가 연결·오류 응답 |
| DB 통합 | 실제 SQLite와 선택한 SQL Server | 동시 접수·lease·중단·고유 제약·원자적 outbox·migration |
| 프로세스 | 실제 PowerShell bridge | JSON 인코딩·출력 제한·종료·timeout·실행 불명확 |
| 실호스트 | tests/lab의 선택 시나리오 | IIS TLS·제품 모듈·실제 VM·SMB·복원·4+4 |

WebApplicationFactory의 가짜 인증은 IIS 클라이언트 인증서·체인·출발지 검증의 증거가 아니다. Pester cmdlet mock도 실제 Hyper-V/VMM·클러스터 성공의 증거가 아니다. [Microsoft 통합 시험](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests), [Pester](https://pester.dev/docs/quick-start), [PSScriptAnalyzer](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview)

## 변경별 실행 범위

| 변경 | 최소 검증 |
| --- | --- |
| 문서만 수정 | 링크·ID·모듈/단계 mapping·내용 일치·diff 형식 |
| Domain 정책 | 해당 정책의 정상·경계·실패 단위 시험 |
| wire/bridge 변경 | 양쪽 계약·구버전 fixture·오류·직렬화 시험 |
| 저장·동시성 변경 | 실제 provider에서 경쟁·rollback·재시작 시험 |
| PowerShell 실행 변경 | Pester·정적 검사 후 격리된 해당 제품 실동작 |
| 인증/인증서 변경 | 허용/거부·교체·키 ACL·실제 TLS 시험 |
| Storage/이관/삭제 | 승인된 테스트 자산으로 보존·복원·잔여·소유권 시험 |

명령 wrapper는 해당 코드가 생길 때 `tools`에 추가한다. 앞으로 .NET 검증은 `dotnet restore --locked-mode`, `dotnet build`, `dotnet test`, `dotnet format --verify-no-changes`를 기반으로 만든다. PowerShell은 고정한 Pester/PSScriptAnalyzer 버전으로 실행한다. 아직 없는 solution·script의 검증을 실행했다고 보고하지 않는다.

일반 CI는 실습 호스트에 접근하지 않는다. 실제 VM 변경은 대상·데이터·시나리오를 명시한 lab 검증 경로로 분리하고 필요한 권한만 제공한다. 실패한 단계와 미실행 단계를 분명히 보고한다.

## 완료 전 확인

변경 목적·plan/module ID·계약 영향·실패/복구·검증 결과·남은 조건을 기록한다. 모든 함수를 기계적으로 시험하거나 구현을 그대로 복제한 시험을 늘리지 않는다. 새 실패·변경·미해결 우려가 없으면 통과한 검사를 반복 확장하지 않는다.
