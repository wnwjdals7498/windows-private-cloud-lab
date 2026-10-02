# windows-private-cloud-lab

> 상태: 계획 단계. VM 실습·자동화·검증 전.

Windows 11 Pro의 단일 물리 Hyper-V 호스트에서 Windows Server 중첩 Compute와 Rocky 고객 VM을 구축하고 PowerShell·VMM으로 자동화한다. 실행 환경은 허용된 사설망에 한정한다.

AD/DNS 1대·VMM/SQL 1대·Compute 3대·Storage 3대의 인프라 8대로 클러스터를 실습한 뒤, 고객 VM·디스크를 보존해 인프라 4대로 전환한다. 이후 별도 Linux 서비스 4대와 연동한다. 인프라 8대와 Linux 8대 시험은 동시에 하지 않으며, 단일 물리 PC 결과를 다중 물리 호스트 장애 내성으로 표현하지 않는다.

[기능·의존성·구현 순서 상세 계획](docs/implementation-plan.md)에 세부 작업, 선행 조건, 사용자 결정 게이트와 완료 증거를 정리했다. [기존 단계별 계획](docs/plan.md)은 이전 초안으로 남긴다. 승인 요구사항의 원본은 PMT 명세서이며, 상세 계획의 새 구현안은 제안 상태다.

구현 작업은 [AGENTS.md](AGENTS.md)와 [작업별 참조 문서](docs/pmt-docs/README.md)를 따른다. 프레임워크 선정, 목표 폴더 구조, 모듈 책임·통신·동작 예시, 확장·코딩·시험·로깅·운영 기준을 정리했다.

[기능별 상세계획](docs/pmt-docs/implementation-details/README.md)은 36개 행동 계약 카드와 GPT-6 Luna 병렬 작업·통합 계획을 제공한다. 파일 편집 지시 대신 입력/출력의 의미와 Test·Logging·복구 증거로 인계한다.

화면 방향은 [UI 가이드](docs/ui/guide.md)와 [HTML 목업](docs/ui/mockup.html)에 정리했다. 최신 배치의 운영 화면은 Linux 관리자 콘솔에 둔다. 기존 가이드의 PHP 배치와 목업의 가상 수치는 현재 구현·검증 결과가 아니다.
