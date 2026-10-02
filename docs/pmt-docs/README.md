# 구현 참조 문서

기준일: 2026-10-02 · 대상: windows-private-cloud-lab · 상태: 구현 방향 문서, 런타임 구현 전

**방향: 모듈로 나눈 하나의 코드베이스, API와 Worker 두 실행 프로세스, 교체 가능한 PowerShell/DB adapter.** 64GB 단일 실습 호스트에서 시작하고 기능·실행 대상·저장소·관측 도구가 늘어날 지점을 인터페이스로 둔다.

## 작업별 읽을 문서

| 필요한 내용 | 문서 |
| --- | --- |
| 전체 순서·145개 작업·완료 증거 | [구현 계획](../implementation-plan.md) |
| 프레임워크·라이브러리·선정 이유·대안·도입 시점 | [기술 선정](technology-stack.md) |
| 전체 폴더 tree·배포 위치·코드 의존 방향 | [구조](architecture.md) |
| 기능별 모듈 책임·입출력·저장 소유권·단계 매핑 | [모듈](modules.md) |
| 모듈 통신·ID·상태·DB·API·PowerShell 계약 | [통신과 데이터 계약](contracts.md) |
| 정상·실패·중복·재시작·이관의 순서와 기대 결과 | [동작 레퍼런스](reference-flows.md) |
| 새 기능·backend·DB·버전·worker 확대 방법 | [확장](extensibility.md) |
| C#·PowerShell·SQL·설정·시험·리뷰 규칙 | [코딩과 검증](coding-and-testing.md) |
| 로그 저장·마스킹·보존·회수·경보·배포·복원 | [로깅과 운영](logging-and-operations.md) |
| 공식 제품 근거·확인 범위 | [공식 자료](official-references.md) |

## 확정·추천·미정

| 구분 | 이 문서에서의 의미 |
| --- | --- |
| 확정 | 기존 사용자 승인: 인프라8→4, Linux 별도 시험·4+4, VMM 경계, 상호 TLS, SQL 읽기, SMB·SSH 보호 |
| 추천 기준 | .NET10·API/Worker 분리·EF Core·구조화 로그·모듈 경계·폴더·보존 기간 등 이번 설계의 기본안 |
| 구현 게이트 | 실호스트 자원·제품 조합·네트워크·IIS 기술·실제 인증서·SQL 조회 대상 등 기존 G01~G10 |
| 실측 | 실제 호스트에서 검증하고 증거가 있는 내용. 이번 문서 작성은 실측을 추가하지 않음 |

IIS 기술(G04)과 MSSQL 조회 대상(G06)은 사용자가 실제 시험 결과를 보고 고르기로 한 사항이다. 추천안을 충분히 구체화하되 승인된 선택으로 기록하지 않는다. 명세가 정한 질문 형식과 시점은 실제 구현 단계에서 적용하며 지금 문서 작성을 중단시키지 않는다.

## 원천과 공동 프로젝트

- [AGENTS.md](../../AGENTS.md)는 매 작업의 공통 규칙, 이 폴더는 작업별 상세 참조다.
- PMT 원천은 공유 작업공간의 [Windows 명세](../../../docs/projects/windows-private-cloud-lab/resources/derived/specification.md)와 [RESUME](../../../docs/projects/windows-private-cloud-lab/RESUME.md)다. 엔진의 `--docs-root`는 저장소 밖 `../docs`다.
- [Cloud 계약 안내](../../../cloud-management-portal-lab/docs/pmt-docs/contracts.md)와 [현재 모의 계약](../../../cloud-management-portal-lab/contracts/prototype-v1.json)을 대조했다. 현재 모의 구현이 실제 VMM 연동까지 구현한 것은 아니다.
- 위 공유 작업공간 링크는 단독 clone에서 없을 수 있다. 이 폴더는 자체적으로 방향을 설명한다. 외부 원천이 없으면 읽었다고 보고하거나 확정 결정을 새로 만들지 않는다.
- 변경 시 해당 기능의 계획 ID, 모듈 ID, 계약/레퍼런스, 시험 기준을 함께 갱신한다. 파일명은 이 문서의 목표 구조이며 실제 생성 시점과 변경 근거를 남긴다.

## 첫 구현 범위

P00~P01에서 환경 설정 형식·읽기 전용 점검·계약 필드·검증 기록을 만든다. P05까지 Hyper-V 단독 동작을 검증하고 P06에서 API/Worker를 붙인다. VMM·SQL·클러스터가 필요한 프로젝트를 초기에 모두 생성하지 않는다. 기능 단계별 도입은 [모듈의 단계표](modules.md)를 따른다.
