# 공식 자료와 확인 범위

확인일: 2026-10-02. 기술 사실은 공급자·프로젝트의 공식 문서만 사용했다. 선정 조합·모듈 경계·보존 기간은 이 저장소의 추천안이며 공급자가 보장한 아키텍처로 표현하지 않는다. 설치 직전에는 선택한 제품 버전·OS·patch 조합을 다시 확인한다.

| ID | 공식 자료 | 확인한 범위 / 적용 |
| --- | --- | --- |
| S01 | [Microsoft .NET 지원 정책](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core) | .NET10 LTS·지원 종료 2028-11-14, 지원 patch 유지. API/Worker 런타임 선택 |
| S02 | [Microsoft ASP.NET Core IIS 호스팅](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/) | Hosting Bundle·IIS hosting model. 실제 G04 선택·설치 검증 필요 |
| S03 | [Microsoft 인증서 인증](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/certauth) | TLS 클라이언트 인증·앱 신원 mapping·proxy 경계 |
| S04 | [Microsoft Windows Worker Service](https://learn.microsoft.com/en-us/dotnet/core/extensions/windows-service) | BackgroundService의 Windows Service 실행·운영 |
| S05 | [Microsoft scoped service in BackgroundService](https://learn.microsoft.com/en-us/dotnet/core/extensions/scoped-service) | Worker의 scoped 서비스 수명 분리 |
| S06 | [Microsoft VMM 2022 요구사항](https://learn.microsoft.com/en-us/system-center/vmm/system-requirements?preserve-view=true&view=sc-vmm-2022) | PowerShell5.1·제품 설치 조합·도메인·SQL2022 UR1 조건 |
| S07 | [Microsoft EF Core SQL Server provider](https://learn.microsoft.com/en-us/ef/core/providers/sql-server/) | SQL provider·설정·호환성 검토 |
| S08 | [Microsoft SQLite 제한](https://learn.microsoft.com/en-us/ef/core/providers/sqlite/limitations) | schema·자료형·동시성 차이. SQL과 동일하다고 가정하지 않음 |
| S09 | [Microsoft SQLite 잠금/오류](https://learn.microsoft.com/en-us/dotnet/standard/data/sqlite/database-errors) | 연결 객체 공유 금지·WAL·busy/locked·timeout |
| S10 | [Microsoft provider별 migration](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/providers) | SQLite와 SQL Server migration을 분리해 관리 |
| S11 | [Microsoft OpenAPI](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/openapi/overview) | ASP.NET Core의 OpenAPI 생성 경로 |
| S12 | [Microsoft IHttpClientFactory](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-requests) | typed/named client·HTTP handler 관리 |
| S13 | [Serilog File sink 공식 저장소](https://github.com/serilog/serilog-sinks-file) | JSON 파일·일/크기 rolling·개수/기간 retention·프로세스별 파일 고려 |
| S14 | [Microsoft OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel) | .NET 로그/metric/Activity 기반 관측·exporter |
| S15 | [xUnit.net v3 공식 시작 안내](https://xunit.net/docs/getting-started/v3/getting-started) | C# 시험 기반. 정확한 패키지/runner는 구현 시 고정 |
| S16 | [Pester 공식 시작 안내](https://pester.dev/docs/quick-start) | PowerShell 시험·assert·mock 기반 |
| S17 | [Microsoft PSScriptAnalyzer](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview) | PowerShell 정적 분석 |
| S18 | [Microsoft ASP.NET Core 통합 시험](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests) | TestServer/WebApplicationFactory. 실 IIS TLS 시험은 별도 |
| S19 | [Microsoft PowerShell 인코딩](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_character_encoding?view=powershell-5.1) | Windows PowerShell의 BOM/비ASCII 주의·프로토콜 UTF-8 분리 |
| S20 | [Microsoft 중첩 Hyper-V](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/nested-virtualization) | Compute 메모리·중첩 클러스터 실습/운영 적합성 경계 |
| S21 | [Microsoft S2D 게스트 클러스터](https://learn.microsoft.com/en-us/windows-server/storage/storage-spaces/storage-spaces-direct-in-vm) | 노드·디스크·장애 도메인·호스트 snapshot 제한 |

문서 URL의 기본 view가 설치 버전과 다를 수 있다. 기능 도입 시 공식 버전 선택·지원 OS·모듈 버전을 기록한다. 접근할 수 없는 자료는 검증됨으로 적지 않고 설치 시험 항목으로 남긴다. 개인 블로그·비공식 튜토리얼·검색 요약을 제품 지원 근거로 사용하지 않는다.

계획의 기존 Hyper-V·Rocky·SMB·클러스터 상세 근거는 [구현 계획](../implementation-plan.md)의 해당 절을 따른다. 새로운 라이브러리를 채택하면 공식 문서·정확한 버전·허용 용도·도입 단계를 이 표와 [기술 선정](technology-stack.md)에 추가한다.
