# Windows 사설 클라우드 실습 계획

## 목표

단일 Windows 11 Pro Hyper-V 호스트에서 수동 VM 생성부터 반복 가능한 PowerShell 자동화, 관리포털 연동까지 단계적으로 검증한다. Windows Server·클러스터·VMM·DPM 확장은 환경 확보 후 진행한다.

운영자 화면의 방향은 [UI 가이드](ui/guide.md)와 [HTML 목업](ui/mockup.html)에서 먼저 검토한다.
실제 호스트·VM 실습 화면은 별도 웹앱으로 만들지 않고 PHP·CodeIgniter 관리포털의 관리자 메뉴에 통합한다.

포털 신청부터 첫 Rocky Linux VM 부팅까지의 공동 최소 범위는 [cloud-management-portal-lab의 첫 동작 결과 계획](https://github.com/wnwjdals7498/cloud-management-portal-lab/blob/main/docs/first-working-slice.md)에 정리한다.

## 단계

1. 호스트 CPU, 메모리, 저장공간, 가상화, 네트워크와 Hyper-V 상태를 확인한다. 실제 사용 가능한 자원과 허용된 사설망 범위를 기록한다.
2. Hyper-V Manager에서 테스트 VM 한 대를 수동 생성한다. 가상 스위치, VHDX, CPU·메모리, 게스트 OS 구성을 이해하고 재현 절차를 남긴다.
3. 동일한 VM 생성·조회·상태 변경·삭제를 PowerShell로 재현한다. 반복 실행, 실패 처리, 재시도와 복구를 검증한다.
4. 같은 64GB Windows PC에서 실행되는 관리포털이 그 Hyper-V 호스트의 지정 IP로 서버 측 JWT 인증 HTTPS cURL 요청을 보내고 IIS 수신부가 지정 PowerShell 절차를 호출한다. IIS가 JWT를 발급하고 PHP 포털은 정해질 유효기간 동안 재사용한다. 첫 JWT 발급 요청은 IP별 무작위 비밀번호로 검증하며 IIS에는 단방향 해시만 저장하고 비밀번호를 3개월마다 변경한다. 자체 사설 CA가 지정 IP를 포함한 IIS 서버 인증서를 발급하고 PHP 호출 환경이 CA를 신뢰한다. 신청 즉시 시작한 작업의 상태·로그·실제 Hyper-V 결과를 연결한다. CA 키·인증서 운영, IP 기준·비밀번호 해시/교체, JWT 보관·갱신·검증·기간·허용 범위와 IIS 실행 계정은 구현 전에 정한다.
5. Windows Server VM, Failover Cluster, VMM, DPM에 필요한 라이선스·도메인·SQL Server·호스트 환경을 조사한다. 확보된 범위에서만 확장하고 미실행 항목은 계획 상태로 남긴다.

## 완료 확인

- 단일 호스트에서 VM 생성·상태 변경·삭제의 실제 결과와 자동화 기록이 일치한다.
- 포털에서 시작된 작업을 추적하고 실패 후 복구를 재현할 수 있다.
- 확장 단계는 사용한 환경과 검증 범위를 명시한다.

## 이후 결정

테스트 VM 사양, 로그 세부 형식과 고급 제품 확장 환경은 단계별 착수 전에 결정한다. 첫 게스트 OS는 Rocky Linux 10 계열이다. 정확한 가상 스위치·허용 10대역 서브넷은 사용자가 지정한다. 포털→IIS→PowerShell 경로, HTTPS 강제, 사설 CA 발급·신뢰, JWT 호출자 인증과 IP별 무작위 비밀번호·3개월 교체 원칙은 확정했다. CA 키·인증서 운영, IP 기준·비밀번호 해시/교체, JWT 검증, IIS 실행 계정·상태 조회 방식은 미정이다.
