# windows-private-cloud-lab

> 상태: 계획 단계. VM 실습·자동화·검증 전.

Windows 11 Pro의 단일 Hyper-V 호스트에서 VM 한 대를 수동으로 만들고 같은 과정을 PowerShell로 자동화한다. 이후 관리포털의 VM 신청·상태 변경 요청과 연결한다. 실행 환경은 허용된 사설망에 한정한다.

Windows Server VM, Failover Cluster, VMM, DPM은 필요한 환경을 확보한 뒤 확장한다. 현재 단일 물리 PC 실습 결과를 다중 물리 호스트 장애 내성으로 표현하지 않는다.

[단계별 계획](docs/plan.md)에 진행 상태와 검증 결과를 기록하고, 구조나 기능이 바뀌면 이 README를 갱신한다.

화면 방향은 [UI 가이드](docs/ui/guide.md)와 [HTML 목업](docs/ui/mockup.html)에 정리했다. 목업의 호스트·VM 수치는 가상 데이터이며 실제 검증 결과가 아니다.
