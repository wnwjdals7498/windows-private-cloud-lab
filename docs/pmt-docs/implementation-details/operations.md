# 운영·인프라 구현 카드

[상세계획 안내](README.md) · [공유 계약](shared-contracts.md) · [병렬 실행](parallel-execution.md)

기준: `docs/implementation-plan.md`의 P10–P13, P16–P17 및 `modules.md`, `reference-flows.md`, `logging-and-operations.md`. 이 문서는 목표 행동 계약이다. 코드·구성·실호스트 완료를 뜻하지 않는다. 작업 선행은 아래에 원래 P ID로 표기하고 G01–G10 및 X01–X04의 결정·외부 인계를 유지한다. 수치·제품 선택은 해당 게이트 및 실측 전 미정이다.

공통 기록 계약: 모든 카드의 진단은 UTC event명, release/mode, 상관 ID, stage/outcome/error code, 필요한 경우 observation version/time을 담는다. 진단 JSONL·DB 작업 원장·감사·outbox·제품 로그·증거는 서로 대체하지 않는다. 비밀·개인키·전체 본문/명령행은 금지하고 내부 주소·UNC도 원본 접근 제한, 공개 증거 마스킹 대상으로 취급한다. 실패한 관측은 성공으로 추정하지 않고 `unknown/stale/needs_attention`을 표현한다. 동작 증거는 모의·계약·실호스트를 구분한다.

## O01 — Storage 클러스터·SMB

| 필드 | 계약 |
|---|---|
| 목적 | Storage 3노드 실습 결과와 Compute의 보호된 SMB 사용을 검증한다. 단일 물리 중첩 구성은 공식 운영지원으로 표시하지 않는다. |
| 추가 범위 | 클러스터 풀/파일 역할, 고객 VM 공유, 암호화·접근 정책 및 VMM 등록 상태라는 외부 행동·상태. |
| 수정 범위 | 저장소 선택·상태 관측·VMM 연결의 허용 조건 및 진단 결과. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. 향후 이 카드에서 자동 삭제 정책을 추가하지 않는다. 이 카드에서 기존 데이터나 공유를 자동 제거하는 정책은 정의하지 않는다. |
| goal | 허용 Compute에서 시험 VM 파일을 SMB로 사용하고 상태를 입증. |
| non-goal | 물리 장애 내성·운영 지원 인증, 기술 선택 게이트 G07 통과 간주. |
| 추적 작업 | P10-01, P10-02, P10-03, P10-04, P10-05, P10-06, P10-07 |
| 책임 모듈 | M11 주책임; M05 실행·VMM, M06 관측, M09 신원/권한. |
| 선행/게이트 | P08-07, G07; 구성 전 G01/G02/G03. 이후 O02, O03. |
| 코드 동작 | 1) 승인된 기술/프로필·노드·디스크·네트워크를 읽고 사전검사한다. 2) 기준 미충족이면 구성 전 중단. 3) 역할/풀/볼륨/공유/암호화/ACL을 제한된 계획대로 적용한다. 4) VMM 등록·Compute 접근을 확인한다. 5) SMB 위치에서 시험 VM 생명주기를 실행하고 실제 자원으로 검증한다. |
| 목표-input | `storageProfile`(필수)은 G07 기술/OS 선택, `nodeInventory`는 노드별 디스크·망 역할(P10-01/02), `preflightResult`는 검사 경고/오류(P10-03), `sharePolicy`는 ACL/암호화(P10-05), `registrationTarget`은 VMM/Compute 연결(P10-06), `testVmPlan`은 승인 profile/image(P10-07)다. 승인 설정·제품 조회·catalog가 원천. 필수값/G07 미확정이면 거부. |
| 목표-output | `selectionDecision`은 선택/검사 판정, `storageObservation`은 역할·pool·volume·share 실제 상태, `accessCheck`는 Compute 접근결과, `registrationRef`는 VMM 연결, `testVmObservation`은 VM/파일 배치·동작이다. M11/M06 생산, 운영자·M05·O02/O03 소비. unavailable/stale/partial은 성공과 분리. |
| 코드동작확인 Test방식 | 정상: 승인된 3노드 실습 구성 준비→사전검사 및 SMB VM 생성/기동/정지→노드·공유 접근·VM 파일 참조 일치 증거. 거부: 필수 디스크/권한/암호화 전제 결여→구성 전 실행→차단 사유와 변경 없음 증거. 중복: 동일 적용 요청→재조회→중복 역할/공유 없음, 같은 상태 보고. 실패·복구: 구성 단계 중 실패 주입→상태 재조회/부분 진행 기록→재실행 전 불명확 상태는 정지, 정합한 복구 증거. 계약·모의 시험과 실호스트는 별도 판정. |
| 코드동작확인 로깅방식 | storage.preflight/configured/share.access_checked/vm.observed; environment/mode, invocation, node role, stage, outcome, error code, observed_at. 진단은 순환 JSONL, 변경은 감사/작업 원장, 제품 로그는 제한 원본, 결과는 manifest/hash 연결 증거. 주소·UNC·계정 마스킹/별칭화. 관측 개수·ACL 결과·기록 누락 검증. |
| 실패·복구 | 구성 전 실패는 부작용 없이 중단. 부분 구성은 자동 삭제 대신 실제 역할/데이터 재조회 후 운영자 조치 상태. 데이터가 있는 pool/share 제거 금지. |
| 완료 조건 | 계약: 불변 입력/상태/오류 정의. 모의: 정상·거부·재실행·부분실패 검증. 실호스트: 실제 제품/노드/SMB·VM 증거와 G07 결정. |
| 인계 정보 | O02의 장애 기준선·독립 복구본 준비, O03의 SMB/Compute 선행 조건, G07 결과·제약·버전·증거 참조. |

## O02 — Storage 장애·독립 복구본

| 필드 | 계약 |
|---|---|
| 목적 | Storage 단일 장애의 실제 영향·복귀를 관측하고 클러스터와 독립된 복구 위치를 확인한다. |
| 추가 범위 | Storage 노드/링크/권한/용량 장애 상태, 복구 관측, 독립 저장소의 복구본·용량 검증. |
| 수정 범위 | 상태·복구 가능 판정 및 운영자 경보. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. 향후 이 카드에서 자동 삭제 정책을 추가하지 않는다. 원본/복구본 자동 정리는 범위 밖. |
| goal | 원인별 데이터 대조·복귀와 3→1 준비 증명. |
| non-goal | S2D 파일/호스트 snapshot을 백업으로 취급하거나 자동 failover 보장. |
| 추적 작업 | P10-08, P10-09 |
| 책임 모듈 | M11 주책임; M06 관측, M10 증거, M05 제품 상태. |
| 선행/게이트 | P10-07, G07, G01; 이후 O03 및 O05. |
| 코드 동작 | 1) 시험 VM·파일 기준선 및 복구 대상 목록을 고정한다. 2) 한 종류 장애씩 주입·관측한다. 3) 데이터/공유/노드 상태와 복구 시각을 대조한다. 4) S2D 풀 및 원본 논리 디스크와 구분되는 볼륨/복구본 목적지와 여유량을 검사한다. 같은 물리PC의 별도 볼륨은 보존이관에 쓸 수 있으나 물리 디스크/호스트 장애 대비 독립 백업은 아니다. |
| 목표-input | `faultPlan`(필수)은 승인 장애/대상, `baselineManifest`는 비교할 VM/파일, `recoveryTarget`은 원본 S2D 풀·논리 디스크와 분리된 볼륨/용량/ACL, `failureDomain`은 물리 디스크/호스트 격리 정도다. 설정·O01 제품조회·G07이 원천. 별도 볼륨은 동일 물리PC라도 보존이관에 허용할 수 있으나 독립 백업으로 표시하지 않는다. 같은 풀/원본 논리디스크면 거부. |
| 목표-output | `faultOutcome`은 장애별 가용/복귀, `dataComparison`은 baseline과 VM·파일 일치, `targetSeparation`은 풀/논리디스크 분리·용량·물리장애 격리 판정, `evidenceRef`는 재현 증거다. M11/M06 생산, O05 소비. 미관측/불일치는 unknown/stale/failed. |
| 코드동작확인 Test방식 | 정상: baseline·독립 목적지 준비→복구본 기록·읽기/복원검증→파일/체크섬 대조. 거부: 목적지가 원본과 같은 S2D 풀/논리 디스크이거나 공간 부족→복사 시작 차단. 별도 볼륨이 같은 물리 호스트에 있으면 이관은 허용할 수 있지만 물리장애 격리로 주장하지 않는다. 중복: 같은 복구 실행 재요청→기존 manifest 대조→덮어쓰기 없이 재사용/충돌 거부. 실패·복구: 노드/링크 등 각 장애를 격리해 자극→관측 손실/복귀 기록→데이터와 서비스 재조회 전 완료 금지. 실호스트 장애는 직렬 수행. |
| 코드동작확인 로깅방식 | storage.fault_test_started/recovered/restore_copy_verified; run/invocation, fault class, resource alias, stage, outcome, observed_at, evidence hash. 작업 원장에 실행 의도/결과, 제품 로그 원본 제한, 증거는 독립성 설명 포함. 내부 경로·주소 마스킹. 예상/실제 데이터 집합·용량·시간 검증. |
| 실패·복구 | 관측 불가 시 장애 시험 중단 및 운영자 확인. 복구본과 원본 모두 삭제하지 않는다. 원본 클러스터 단독으로 복구 가능하다고 판정하지 않는다. |
| 완료 조건 | 계약/모의에서 복구본 분리 규칙 확인; 실호스트에서 장애별 복귀와 풀·논리 디스크와 분리된 복구본 복원 가능성 증명. |
| 인계 정보 | O05에는 독립 목적지, manifest/hash, 용량 및 제한을 전달; G07 및 실측 경과를 첨부. |

## O03 — Compute 클러스터·이동

| 필드 | 계약 |
|---|---|
| 목적 | Compute 3노드의 재현 설정과 시험 VM 계획 이동을 검증한다. 중첩 단일 물리 호스트 클러스터를 운영지원 구성으로 나타내지 않는다. |
| 추가 범위 | 3노드 등록·네트워크/이미지/자원 적합성, 클러스터 멤버십, 계획 이동 상태. |
| 수정 범위 | Compute 위치와 VM 실행 참조·관측 매핑. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. 향후 이 카드에서 자동 삭제 정책을 추가하지 않는다. 노드 제거나 VM 데이터 삭제 없음. |
| goal | 계획 이동에서 VM ID·디스크·IP·게스트 접근을 대조. |
| non-goal | 물리 장애 도메인 보장·제품 지원 주장. |
| 추적 작업 | P11-01, P11-02, P11-03, P11-04, P11-05 |
| 책임 모듈 | M11 주책임; M05 VMM 실행, M06 관측, M10 증거. |
| 선행/게이트 | P08-07, O02의 P10-09, G01/G02/G03/G07; 이후 O04/O05. |
| 코드 동작 | 1) 두 추가 Compute의 버전/이름/도메인/Hyper-V/VMM 등록을 비교한다. 2) 네트워크·이미지·SMB ACL·여유 자원 사전검사. 3) 사전검사 경고/실패와 중첩 한계를 기록하고 진행 판정. 4) 승인된 클러스터 설정을 적용/관측한다. 5) 저장된 시험 VM을 계획 이동하고 전후 자원·게스트를 대조한다. |
| 목표-input | `computeNodeSet`(필수)은 노드 역할/OS/Hyper-V/VMM 버전, `networkProfile`은 이동망, `imageRef`는 이미지, `shareAcl`은 SMB 권한, `capacityObservation`은 노드 여유, `testVmRef`는 대상 VM이다. profile/catalog·제품관측·G01/G03/G07 원천; 버전/망/권한/용량 불명·불일치는 거부. |
| 목표-output | `membershipObservation`은 노드/멤버십·검사, `moveBinding`은 전후 host/VM/storage, `guestReachability`는 전원·고객 주소·SSH 확인이다. M11/M06 생산, O04 장애 기준선·O05 이관 위치가 소비. 미관측은 unknown, 지원 한계는 미검증. |
| 코드동작확인 Test방식 | 정상: 3노드 일치와 SMB baseline→계획 이동→실제 ID·디스크·네트워크·SSH 대조 증거. 거부: 노드 설정/권한/용량 차이→이동 전 차단→preflight 차이 보고. 중복: 같은 목표 노드의 반복 요청→현재 소유 위치 재조회→불필요한 두 번째 이동 없음. 실패·복구: 이동 중 관측 중단→제품 Job/VM 재조회→unknown 유지, 중복 실행 금지, 복귀 증거 기록. 실제 이동은 충돌 카드와 직렬. |
| 코드동작확인 로깅방식 | compute.preflight/membership.observed/vm.move_submitted/vm.move_verified; run, invocation, VMM job, VM ID, source/target role, stage, version/time. 작업원장·제품로그·마스킹 증거 구분. VM ID는 로그/trace에만, metric label 제외. 실제 VM이 목표 노드에 있는지 재조회. |
| 실패·복구 | 제출 결과 모호하면 실행 재호출 금지, VMM Job과 실제 VM을 대조하고 needs_attention. 설정 차이는 수정 승인을 기다리며 자동 정규화하지 않는다. |
| 완료 조건 | 계약·모의 상태전이 및 실패중복 차단; 실호스트에서 이동/VM 접근 증거, 성공과 실습 한계 동시 기록. |
| 인계 정보 | O04의 baseline·멤버십, O05의 VM 위치/ID/SMB 매핑, 8대 실측은 P11-09 자료로 전달. |

## O04 — 클러스터 장애·정상 복귀

| 필드 | 계약 |
|---|---|
| 목적 | Compute 장애·Storage 장애 및 통제 조합의 영향을 분리 시험하고 정상 기준선을 회복한다. |
| 추가 범위 | 역할 정지/복귀·failover·재동기화·잔여 자원·이벤트의 시험 기록. |
| 수정 범위 | 클러스터·VM 관측 상태와 기준선. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. 향후 이 카드에서 자동 삭제 정책을 추가하지 않는다. 장애 시험을 위한 데이터 제거 없음. |
| goal | 장애별 실제 중단/복구·데이터 대조. |
| non-goal | 단일 물리 호스트 장애 내성·운영 SLA. |
| 추적 작업 | P11-06, P11-07, P11-08, P11-09 |
| 책임 모듈 | M11 주책임; M05, M06, M10. |
| 선행/게이트 | O03의 P11-05, O02의 P10-08; G01/G07; 후속 O05. |
| 코드 동작 | 1) baseline/자원 여유/복구 조건을 고정한다. 2) Compute 한 대 정지·이동·복귀를 시험한다. 3) 별도 실행으로 Storage 장애와 Compute 장애를 비교하고 통제 조합만 허용한다. 4) 재동기화/이벤트/VM 데이터/멤버십을 재조회한다. 5) P11-09 결과를 성공·실패·미검증으로 봉인한다. |
| 목표-input | `faultScenario`(필수)는 Compute/Storage 장애 종류·순서, `abortCriteria`는 중단 경계, `baseline`은 O02/O03 VM·데이터·멤버십, `capacityHeadroom`은 실행 전 여유, `priorRunState`는 앞선 복구 여부다. O02/O03 증거·운영 승인 원천; 누락/미회복이면 주입 거부. |
| 목표-output | `faultOutcome`은 영향/복귀, `integrityResult`는 데이터 대조, `resyncState`는 재동기화, `remainingCapacity`는 복구 후 여유, `baselineRestored`는 기준선 여부, `infra8Summary`는 P11-09 실측이다. M11/M06 생산, O05/O08 소비. 누락은 unknown/failed/not-tested. |
| 코드동작확인 Test방식 | 정상: 격리 시험 노드·데이터 준비→장애 한 종류 주입→복귀 및 재동기화 검증→baseline 복원 증거. 거부: 이전 복구 미완료/여유 부족/시나리오 승인 없음→다음 장애 차단. 중복: 같은 run ID 재요청→증거 참조 반환, 장애 재주입 금지. 실패·복구: 복귀/관측 실패 주입→추가 시나리오 중지, 운영자 조치·기준선 미회복 표현. 각 실호스트 장애 시나리오 직렬. |
| 코드동작확인 로깅방식 | cluster.fault_started/role_moved/node_returned/resync_verified/baseline_restored; run, fault class, node role, VM ID, VMM job, stage, outcome, observed_at, evidence hash. DB 원장·진단·제품 원본·공개 요약 구분/마스킹. 기대 데이터 및 기준선 자동 대조. |
| 실패·복구 | 노드 추가 장애는 이전 장애 회복·데이터 확인 전 금지. 상태 불명은 성공 처리 금지. 실습 실패도 증거로 보존한다. |
| 완료 조건 | 계약/모의로 실행 잠금·상태 추적 검증; 실호스트 별도 장애 실험·복귀 증거와 지원 한계 완료. |
| 인계 정보 | O05의 기준 VM·복구 판정 및 자원 실측, O08 관측 모델에 장애/정상 전이 제공. |

## O05 — 데이터 보존 이관

| 필드 | 계약 |
|---|---|
| 목적 | 클러스터에서 독립 Compute 1·Storage 1로 옮겨 기존 고객 VM 데이터·논리 ID·접근을 보존한다. Storage 3→1은 독립 저장소 보존 이관이다. |
| 추가 범위 | 유지 자산 목록·신규 독립 저장소·복구본·복사 검증·바인딩/ID mapping·유지보수 모드. |
| 수정 범위 | VM의 물리 참조·저장 경로·VMM binding 및 관측 mapping. 논리 고객 VM ID는 유지. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. P12-01..08 이관 중 원본/복구본 데이터 자동 삭제 없음. 추가 노드 정지/자동시작은 O06(P12-09)으로 인계한다. |
| goal | 검증된 복구본에서 단독 목적지로 이관하고 부팅·SSH·파일·전원제어를 확인. |
| non-goal | S2D 1노드화, 동일 데이터의 동시 기동, copy 완료만으로 성공 판정. |
| 추적 작업 | P12-01, P12-02, P12-03, P12-04, P12-05, P12-06, P12-07, P12-08 |
| 책임 모듈 | M11 주책임; M03 접수/작업정지, M04 정책, M06 관측, M07 기록, M13 투영. |
| 선행/게이트 | P11-09, P09-10, O02의 P10-09, G07/G08/G01; 이후 O06/O07. |
| 코드 동작 | 1) VM/ID/경로/키/IP/파일 참조 inventory 및 기준 hash 작성. 2) 접수 잠금·진행 작업/outbox 정리, 되돌림 지점 지정. 3) VM 정상종료 및 일관 복구본 검사. 4) S2D 풀 및 원본 논리 디스크와 분리된 단독 볼륨/파일시스템/SMB 목적지 확인. 5) 복사 후 체크섬·권한·부모 디스크 참조 대조. 6) 단독 Compute 등록·스위치/경로/권한 복원. 7) VMM·Windows binding/상태 모델 mapping 원자 갱신. 8) 게스트 데이터·SSH·전원 제어 검증 뒤 운영 재개. |
| 목표-input | `assetManifest`는 VM/논리ID/disk chain/network/key 참조(P12-01), `quiescenceState`는 접수·작업·outbox 정리(P12-02), `restoreManifest`는 검증 복구본, `destinationVolume`은 원본 S2D 풀/논리 디스크와 분리된 볼륨·공간·ACL(P12-04), `bindingPlan`은 구/신 host/share 관계, `rollbackPoint`는 승인 복귀점이다. 작업원장·제품 inventory·O02 원천. 같은 물리PC의 별도 볼륨은 이관 가능하나 물리장애 독립 백업은 아니다. 진행 작업, 동일 풀/논리 디스크, 부족 공간이면 컷오버 거부. |
| 목표-output | `migrationManifest`는 단계/hash/파일수/권한, `bindingHistory`는 논리 ID와 구/신 host/share·실제 ID, `guestVerification`은 부팅/SSH/데이터, `admissionResumeDecision`은 잠금 해제 근거다. M11/M06/M13 생산; O06 정지노드, O07 인프라4 기준, S09 첫 회원 시험에서 소비. 실패/미확정은 미완료. |
| 코드동작확인 Test방식 | 정상: 모의 데이터·풀/논리디스크와 분리된 목적지 볼륨·잠금 준비→이관 단계 수행→hash/게스트내용/논리ID/실제 VM·SSH 대조. 거부: active job, 같은 S2D 풀/원본 논리디스크의 복구본, 부족 공간, 미해결 disk parent→복사/시작 차단. 동일 물리PC의 별도 볼륨은 보존이관에 허용 가능하되 물리장애 백업이라고 보지 않는다. 중복: run 재개→manifest/단계별 외부 상태 대조→복사/등록 중복 없이 이어감. 실패·복구: 복사/등록/원자 mapping 전후 실패 주입→양쪽 상태 재조회, 한쪽만 활성 보장, 사전 검증 복구본으로 복귀 또는 needs_attention. 실호스트 이관/컷오버는 직렬 단일 실행. |
| 코드동작확인 로깅방식 | migration.inventory_frozen/admission_paused/backup_verified/copy_verified/binding_switched/guest_verified; run, logical VM ID, old/new resource alias, stage, hash reference, actor, before/after version. 감사와 작업 원장 영속; 진단은 제한 순환; 증거 manifest 공개본 마스킹. 고객 이름/주소/키/경로는 최소화. 두 VM 동시기동·체크섬 불일치·원장 누락을 검증. |
| 실패·복구 | 컷오버 전 실패는 원본 유지. 컷오버 후 실패 시 신규 목적지 접수/기동을 잠그고 사전 검증 restore point로 복구. 삭제/원본 해제는 검증 후 별도 작업. ID mapping은 원자적, 자동 부분 갱신 금지. |
| 완료 조건 | 계약·모의에서 단계 재개/멱등·롤백 검증; 실호스트에서 독립 저장소·전체 보존/SSH/부팅, 실제 ID 매핑 및 증거 확인. |
| 인계 정보 | O06에 정지 노드 목록/자동 시작 차단 대상; O07에는 인프라4 정상 프로필·데이터/서비스 기준선과 인계 증거. |

## O06 — 추가 노드 정지·재부팅

| 필드 | 계약 |
|---|---|
| 목적 | 전환에서 제외된 Compute·Storage 총 4대를 정지하고 재부팅 후 인프라 4대의 순차 기동과 데이터 가용성을 확인한다. |
| 추가 범위 | 노드별 정지/자동시작 제한, 서비스 준비 기준 순차 부팅, 재부팅 후 가동 목록. |
| 수정 범위 | lab profile의 인프라4 실행/자동시작 상태. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. 향후 이 카드에서 자동 삭제 정책을 추가하지 않는다. VM·디스크·클러스터 데이터 삭제 안 함. |
| goal | 추가 4대가 예약/자동시작으로 되살아나지 않고 인프라4 복구. |
| non-goal | 타이머만으로 의존성 준비 판정. |
| 추적 작업 | P12-09, P12-10 |
| 책임 모듈 | M11 주책임; M06 관측, M10 기록. |
| 선행/게이트 | O05 P12-08, G01/G07; 이후 O07. |
| 코드 동작 | 1) 대상·유지 역할·고객 VM 확인. 2) 추가 노드 안전 종료 및 자동시작/예약 시작 제한. 3) 호스트 재부팅 후 AD/DNS→Storage→VMM/Compute→고객 VM→API의 실제 준비 상태 확인 후 다음 역할 시작. 4) SMB/VMM/VM/데이터를 검증. |
| 목표-input | `keptNodeSet`/`stoppedNodeSet`은 유지/정지 역할, `autostartPolicy`는 추가노드 기동 억제, `readinessChain`은 의존 준비조건, `migrationManifest`는 O05 데이터 검증, `activeWork`는 진행 작업 유무다. lab profile·제품 조회·O05 원천; 역할 모호/작업중/이관 미검증이면 거부. |
| 목표-output | `nodePowerState`/`autostartState`는 추가노드 정지/억제, `readinessResult`는 역할별 시작 허용, `rebootEvidence`는 인프라4·SMB·VMM·VM 복구다. M11/M06 생산; O07 Linux8 허용/4+4 기준, O13 boot/mode 인수 소비. 실패는 대기/수동 조치로 분리. |
| 코드동작확인 Test방식 | 정상: 모의 상태 전이→대상 4대 off/자동시작 제한, 재부팅 후 readiness별 진행→가동 4대·서비스 확인. 거부: 고객 VM 또는 migration 작업 실행 중→정지 차단. 중복: 같은 profile 적용→기존 전원/제한 재조회, 무해한 결과. 실패·복구: 역할 readiness fail→후속 역할 기동 중지, 원인/재개 지점 보존. 실호스트 재부팅은 다른 VM 시험과 직렬. |
| 코드동작확인 로깅방식 | lab.node_shutdown/startup_blocked/readiness_passed/host_recovered; run, mode, role, stage, observed_at, outcome, evidence ref. 작업 원장/감사, Windows Event Log/제품 로그, 증거 분리. 실제 주소 마스킹. 가동 수·역할·중첩 합산 정책 검증. |
| 실패·복구 | 시작 순서 실패는 재시도 전 해당 서비스 실제 상태 검사. 광범위 자동 기동 금지. 잔여 노드/VM은 운영자 주의 상태. |
| 완료 조건 | 모의 profile transition; 실호스트 cold boot 후 순차 준비·SMB/VMM/VM 실증. |
| 인계 정보 | O07의 Linux 별도 시험 허용 시점, 인프라4 자원·주소·준비 상태; O12의 재부팅/가동모드 증거. |

## O07 — Linux8 별도 시험·4+4 인계

| 필드 | 계약 |
|---|---|
| 목적 | 인프라8을 정리한 뒤 Linux8 별도 시험의 외부 증거를 인수하고 서비스4+4 운영 입력을 연결한다. |
| 추가 범위 | 모드 상호배제, Linux8/X03 인계, Linux4 주소·신원·DB 접속 정보, 양방향 통신 검증. |
| 수정 범위 | 허용 lab mode와 4+4 서비스 프로필. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. 향후 이 카드에서 자동 삭제 정책을 추가하지 않는다. 인프라/Linux 데이터 자동 정리 없음. |
| goal | 인프라8과 Linux8 비동시 보장, Cloud 시험 및 4+4 계약 증거 수집. |
| non-goal | Linux DB 이중화/화면/API 구현을 Windows 소유로 가져오지 않음. |
| 추적 작업 | P13-01, P13-02, P13-03, P13-04, P13-05 |
| 책임 모듈 | M11 주책임; M01 profile, M08 통신검증, M09 인증서, M13 SQL 읽기 연결. |
| 선행/게이트 | P12-10, P02-07, X03, X01/X02; Cloud 산출물 미확인 시 P13-02 인계 대기. |
| 코드 동작 | 1) 인프라 작업/VM 정리 및 Linux8 실행 자원 준비, 동시가동 차단을 확인. 2) X03 Cloud 완료 증거를 인수·검토. 3) Linux4의 역할별 주소/이름/인증서/SQL 대상 계약을 받는다. 4) 나머지 Linux4 자동시작 억제. 5) 인프라4 순차 기동 후 API→VMM/MSSQL, VMM→API 및 고객 SSH 통신·신원 확인. |
| 목표-input | `currentMode`는 실제 실행 중인 lab profile, `x03Result`는 Linux8/VIP/배포/DB 검증, `apiContract`는 X01/X02 endpoint/event 의미, `linux4IdentityMap`은 역할별 주소·인증서 참조·G06 SQL 객체, `capacityProfile`은 4+4 자원표다. M11 제품 조회·공식 Cloud 인계·승인 자료 원천. 필수 결과/계약 누락 또는 8+8 충돌이면 전환 거부. |
| 목표-output | `modeTransition`은 상호배제된 전환, `handoffRef`는 X03 증거/버전, `serviceProfile`은 4+4 역할·주소/신원 관계, `pathCheck`는 양방향 handshake/SQL 접근 결과다. M11/M08/M09/M13 생산; O08 관측·O10 신원·O13 mode 관리와 S09 첫 회원 통합시험이 소비한다. 비밀은 안전한 참조만 전달. |
| 코드동작확인 Test방식 | 정상: 가상 profile과 X03 fixture→Linux8 완료 후 Linux4/인프라4 전환→허용 통신·신원 확인. 거부: 인프라8 활성 상태에서 Linux8 시작 요청→차단 및 현재 상태 증거. 중복: 동일 X03 manifest 제출→중복 인계 없이 hash 일치 확인. 실패·복구: 인증서/SQL/네트워크 실패 자극→해당 경로만 실패·상태 unknown, 자격/신뢰 갱신 후 재검증. Linux8과 인프라8 실호스트 시험은 시간상 상호 배타·직렬. |
| 코드동작확인 로깅방식 | mode.transition_blocked/completed, handoff.received, peer.identity_verified, sql.read_checked; mode/run, X ID, contract/release version, target role alias, outcome, observed_at. 비밀 정보는 secret ref도 필요한 경우 최소화. 감사/증거 원본 분류, 공개본 주소/인증서 정보 마스킹. 동시 활성 역할 수·신뢰 결과 검증. |
| 실패·복구 | 외부 X03 미완료를 mock 성공으로 채우지 않는다. mode 충돌 시 신규 기동 금지. 연결 실패에서 운영자가 수동 확인할 항목과 재검증 기준 제공. |
| 완료 조건 | 계약/fixture에서 상호배제 및 인계 유효성; Cloud 실증 자료를 연결하고 실호스트 4+4 통신 증거를 별도 판정. |
| 인계 정보 | O08 관측 source/version·role map; O10 인증서 주체; O13 mode 전환 규칙. X03 결과는 첫 회원 시험 S09의 통합 선행증거다. G01/G05/G06 결정 상태를 보존. |

## O08 — 상태·성능 관측

| 필드 | 계약 |
|---|---|
| 목적 | 실제/요청 상태·관측 신선도·자원·작업 지표를 Cloud 관리자에게 안전하게 제공한다. |
| 추가 범위 | host/VM/job/SMB/capacity/certificate 관측, 지연·오류·queue metric, 관리자 read model. |
| 수정 범위 | 승인된 MSSQL 읽기 projection과 경보 판정. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. 향후 이 카드에서 자동 삭제 정책을 추가하지 않는다. 기록 보존/회수는 원 운영 정책과 분리. |
| goal | stale/unknown/unavailable과 실제 상태를 분명히 표시. |
| non-goal | 관리 UI 자체, SCOM 도입, L2 RAM 이중계산. |
| 추적 작업 | P16-01, P16-02, P16-03 |
| 책임 모듈 | M06 관측, M10 metrics/evidence, M13 읽기 모델. |
| 선행/게이트 | P15-10, G09, X04; M13 선택은 G06; O07 인계와 연계. |
| 코드 동작 | 수집원별 observed_at/version/freshness 기록→요청 상태와 제품 상태를 분리 투영→지연/불일치·resource/queue 지표 집계→Cloud에 승인 필드만 전달→실제/실습/가상 표시·증거 참조 정합성 확인. |
| 목표-input | `resourceObservation`은 host/VM/SMB/capacity/certificate 실상태(M06), `jobObservation`은 원장/outbox 단계, `collectionPolicy`는 G09 위치·주기·보관, `readModelSource`는 G06 승인 SQL 객체, `adminFields`는 X04 안전 표시 필드다. M06/M03/M08·게이트·Cloud 계약이 원천. 단위/관측시각 필수; stale·권한 밖 값은 차단/unknown 처리. |
| 목표-output | `resourceState`는 용량/성능 관측, `vmJobState`는 요청과 실상태, `freshness`는 source/version/observed_at, `metricSample`은 측정값·단위·범위, `safeDiagnostic`은 안전 오류/evidence ref다. M06/M10/M13 생산; O09 복구와 O11 관리자 인계·Cloud read model 소비. unobserved/stale/unavailable 구분. |
| 코드동작확인 Test방식 | 정상: 신선한 관측/계약 fixture→투영→원천 값·시각·단위 일치. 거부: 승인 밖 SQL 객체/필드→쿼리 차단·감사. 중복: 같은 observation version 재입력→결과 중복/퇴행 없음. 실패·복구: 관측원 단절·낡은 version→stale/unavailable 표기, 재연결 후 최신 버전만 수렴. 계약/모의 + G09 실측 별도. |
| 코드동작확인 로깅방식 | inventory.observed/stale, metrics.sampled, readmodel.projected/mismatch; source, role alias, observation_version/time, unit, mode, error code. ID는 logs/traces만, metric labels에서 제외. raw product logs 제한·증거 마스킹. 단위·계산·이중 합산 방지 검증. |
| 실패·복구 | metric exporter 실패는 업무 완료를 막지 않으며 drop/health를 측정. 핵심 상태원천 실패는 unavailable 처리. 오래된 값으로 최신 상태 덮어쓰기 금지. |
| 완료 조건 | 계약: 투영 필드·freshness·권한 정의; 모의: 상태/정렬/권한 검증; 실호스트: G09 지표와 Cloud 화면 X04 상태·증거 대조. |
| 인계 정보 | 관리자 safe field·오류/마스킹 설명·지원 범위; O09/O10/O13 운영 경보 및 인수표. |

## O09 — 작업자·통지 복구

| 필드 | 계약 |
|---|---|
| 목적 | 프로세스/서비스 중단, 통지 전달 오류, SQL/큐/주소/로그 자원 부족 시 잘못된 재실행·성공 전이를 막는다. |
| 추가 범위 | recovering 상태, outbox 재전달·수동 재처리 및 작업 복구 상태. |
| 수정 범위 | 미완료 작업 복구 상태와 통지 전달 상태. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. 향후 이 카드에서 자동 삭제 정책을 추가하지 않는다. 원장/outbox/VM/복구본으로 공간 확보 금지. |
| goal | 기존 작업을 재조회·인계하고 자원 회복 후 재개. |
| non-goal | timeout/lease 만료만으로 재제출, 자동 데이터 삭제. |
| 추적 작업 | P16-04, P16-05 |
| 책임 모듈 | M03/M07 작업·저장, M08 통지, M06 실제 관측. |
| 선행/게이트 | P16-03, O08 관측. 작업·통지복구는 P16-04/05이며 O10의 인증서 교체와 O11의 자원 고갈 인계가 뒤따른다. |
| 코드 동작 | 1) 종료/재시작 후 queued/running/unknown/outbox를 조회. 2) submit_intent와 실제 VMM Job/VM을 reconcile. 3) 통지는 원 event ID/body로 제한 재시도, 소진 시 보류. 4) 원장 쓰기 실패는 접수를 거부하고 진단 로그 실패는 drop 계수/Event Log fallback. 5) 복구된 실제 상태·원장 정합성 확인 뒤 운영자/정책에 따라 재개. |
| 목표-input | `jobRecord`는 원장 상태/lease/submit intent, `executionObservation`은 VMM Job·실VM 상태, `outboxRecord`는 event ID/body hash/attempt, `serviceHealth`는 SQL/worker/API 상태, `retryPolicy`는 통지 시도 한계다. M07/M06/M08 원천; 어떤 외부 실행이라도 미확인 상태면 재제출 금지. |
| 목표-output | `recoveryState`는 recovering/needs_attention, `deliveryState`는 pending/delivered/held, `reconcileResult`는 기존 Job/VM 연결 여부, `retryResult`는 같은 event 재전달 판정이다. M03/M08 생산; 운영자/Cloud 및 O11 인계 소비. 상태 미확정은 needs_attention이며 성공 전이 금지. |
| 코드동작확인 Test방식 | 정상: 미완료 job+기존 VM fixture→worker 재기동→재조회/기존 결과 연결, 통지 전달. 거부: 원장 쓰기 실패·disk budget 고갈→신규 변경→202 없음, 기존 작업 보존. 중복: 같은 event 재전달·ACK 유실→같은 event ID 재전송→업무 반영 1회. 실패·복구: SQL/worker/IIS 중단·역순/장기단절 주입→recovering·경보·원본 보존, 의존원천 회복 후 실제 VM·Job 재검증. 모의/실호스트 별도. |
| 코드동작확인 로깅방식 | job.recovery_started/execution.unknown/completion.retry_scheduled/delivered; job/event/run, attempt, stage, error, budget/observed_at. 진단 JSONL, 원장, 감사, outbox, 제품로그 별도. 비밀/본문 제외; event 본문 hash로 검증. |
| 실패·복구 | 재시도 소진은 needs_attention 및 경보. 새 VM 실행으로 해결하지 않는다. 원장/outbox/tombstone 수동 삭제 금지. 공간 복구는 승인된 로그 순환/보존 절차만. |
| 완료 조건 | 계약·모의 fault injection에서 미중복/복구; 실호스트 서비스 재시작·통지 장애를 실행증거로 분리. |
| 인계 정보 | O10 인증서 오류, O11 자원고갈 운영, O13 복원/운영 인계에 오류코드·수동절차·증거 링크. |

## O10 — 인증서·서비스 신원 교체

| 필드 | 계약 |
|---|---|
| 목적 | mTLS 서버/클라이언트 인증서와 서비스 신원 교체·폐기·접속 거부를 안전하게 검증한다. |
| 추가 범위 | 허용 신뢰/등록 신원 교체 상태, 키 ACL 및 이전 신원 폐기 증거. |
| 수정 범위 | 활성 인증서 binding/trust 및 service principal 매핑. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. 향후 이 카드에서 자동 삭제 정책을 추가하지 않는다. 개인키/신뢰자료 자동 삭제 범위는 없음; 별도 안전한 폐기 정책 준수. |
| goal | 교체 전후 정상/거부를 입증. |
| non-goal | 인증서 발급 권한·CA 키 관리 자체, 인증서 유효성만으로 VM 권한 부여. |
| 추적 작업 | P16-06, P16-07 |
| 책임 모듈 | M09 주책임; M10 보안 감사, M08 통신검증, M11 운영창구. |
| 선행/게이트 | P16-05, G05, X01/X02; O07 연결 신원 map. |
| 코드 동작 | 1) 새 신원 chain/SAN/EKU/등록 출발지/키 ACL/만료를 읽기 검사. 2) 갱신 창에서 수신·발신 binding을 단계 교체. 3) 허용 호출과 만료/CA/name/revocation/ACL/출발지 오류를 각각 검증. 4) 이전 신뢰 철회 후 구 인증서 거부 확인. |
| 목표-input | `certificateRef`는 비밀 없는 신원 참조, `certificateStatus`는 chain/name/EKU/expiry/revocation, `serviceRegistration`은 허용 주체/source, `keyAcl`은 서비스 계정 접근판정, `rotationPlan`은 교체/rollback 순서다. 보안 store·G05·Cloud 등록정보 원천. 상태 누락/불신/권한 불가면 인증·작업 차단; 개인키 입력 금지. |
| 목표-output | `activeIdentity`는 사용 중인 인증서 참조/만료, `rotationStage`는 교체 진행, `trustDecision`은 허용/거부 및 이전 신원 폐기, `peerHealth`는 연결 검증 결과다. M09/M08 생산; O08 관측·O11 관리자 인계·운영자 소비. 실패는 safe error code와 needs_attention으로 표시. |
| 코드동작확인 Test방식 | 정상: 테스트 CA/신원 fixture→새 chain/등록 신원으로 mTLS→인증된 권한 범위 내 연결 성공. 거부: 잘못된 name/EKU/CA/만료/폐기/source/ACL을 각각 자극→거부·감사, 실행 부작용 없음. 중복: 동일 certificate rotation 요청→활성 fingerprint 재조회, 재발급/부작용 없음. 실패·복구: 중간 단계 교체 실패→승인된 이전/신규 신뢰 공존 창 내 복구, 이전 폐기 후 재연결 실패는 명확히 보류. 실호스트 인증서 교체는 직렬·점검창. |
| 코드동작확인 로깅방식 | security.identity_checked/denied/certificate.rotation_started/activated/revoked; service principal alias, certificate thumbprint의 제한된 표시, issuer/name result, stage, correlation, outcome. 원장/감사 접근 제한, 개인키·token 절대 기록 금지. 공개 증거는 식별값 마스킹. 폐기 후 이전 신원 거부와 새 신원 허용 검증. |
| 실패·복구 | 잘못된 인증서는 fail closed. 교체 불명은 변경 접수 중단 및 실제 binding 재조회. 개인키를 로그/증거에 복사하지 않는다. |
| 완료 조건 | 계약·모의 PKI 검사; 실호스트에서 실체인·접속·권한·폐기 시험 및 G05/X 연계 증거. |
| 인계 정보 | O11 자원/관리자 경보에 인증서 상태를 제공하고 O14에 신원 소유자·만료/복원 인계. |

## O11 — 자원 고갈·관리자 인계

| 필드 | 계약 |
|---|---|
| 목적 | 자원 압박에서 접수를 제한하고 Cloud 관리자에게 안전한 상태·증거를 연결한다. |
| 추가 범위 | 큐/주소/저장용량 한도·회수 판정, 관리자 상태·증거·가상데이터 표기. |
| 수정 범위 | 신규 변경 접수 판정 및 관리자 투영. |
| 삭제 범위 | 현재 삭제 없음. 미래에도 VM·백업·미전달 원장 삭제로 공간을 확보하는 동작은 금지한다. |
| goal | 고갈 원인·안전한 회수·관리자 표시 검증. |
| non-goal | Cloud 화면 구현 또는 SCOM 도입. |
| 추적 작업 | P16-08, P16-09 |
| 책임 모듈 | M10 경보/증거, M06 관측, M13 projection, M02 admission. |
| 선행/게이트 | O08 관측, O09 작업 복구, O10 인증서 운영, G09/G10/X04. 선행은 계약/fixture 준비와 P16-01..07 결과이며 단계 전체 인수를 선행으로 요구하지 않음. |
| 코드 동작 | 예산/큐/주소를 관측→한도 전 접수 제한→회수계획 기록 후 허용 로그만 정리→관리자 projection에 실습/가상/실상태와 safe error/evidence ref 제공→Cloud 화면 증거 대조. |
| 목표-input | `resourceBudget`은 disk/log 여유, `queueAge`는 작업 지연, `addressCapacity`는 남은 주소, `retentionExceptions`은 삭제 금지 원장/증거, `thresholdPolicy`는 G09/G10 한도, `adminContract`는 X04 표시/evidence 필드, `freshness`는 최신성이다. M06/M10·Cloud 계약 원천; 미관측은 여유로 보지 않는다. |
| 목표-output | `capacityAlert`는 한도 등급, `admissionDecision`은 허용/거부 이유, `retentionPlan`은 허용 로그 회수, `adminProjection`은 상태/evidence와 가상·실습 표기다. M10/M13 생산, Cloud 관리자 소비. 오래된 원천은 정상으로 표시하지 않는다. |
| 코드동작확인 Test방식 | 정상: 충분한 자원 fixture→투영→원천/관리자 값 일치. 거부: 주소/원장 공간 부족→변경 접수→거부, 데이터 보존. 반복: 같은 회수/표시 요청→중복 삭제 없이 재조회. 실패·복구: exporter/회수/화면 연결 실패→미전달·drop·미관측 표시, 원장/백업 삭제 없음; 회복 후 실제 용량 재측정. 실호스트 임계치 주입은 직렬 수행. |
| 코드동작확인 로깅방식 | resource.threshold_crossed/admission.rejected/retention.completed/admin_projection.mismatch; budget class, mode/run, threshold, outcome, evidence ref. 진단·원장·감사·증거 구분, 비밀/주소 마스킹. 임계치·회수 제외목록을 실제 잔여 데이터로 검증. |
| 실패·복구 | 회수 대상이 없으면 신규 변경 접수를 503으로 제한하는 운영안 적용. 회수 실패는 성공으로 표시하지 않는다. |
| 완료 조건 | 계약/모의 임계치·projection 확인; G09 실측 및 X04 관리자 화면 대조는 별도 실호스트/외부 인수. |
| 인계 정보 | O12 배포 전 상태, O14 요구사항 및 증거 인수 자료. |
## O12 — 배포 버전·반복실행

| 필드 | 계약 |
|---|---|
| 목적 | 실행 코드/설정/이미지/도구/계약/DB 변경의 버전과 반복 적용 결과를 재현 가능하게 한다. |
| 추가 범위 | release manifest, 해시·호환 버전, 단계별 이미 적용/진행/실패/복구 상태. |
| 수정 범위 | release catalog 및 운영 단계 실행 결과. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. 향후 이 카드에서 자동 삭제 정책을 추가하지 않는다. 이전 release artifact/DB 기록 자동 삭제하지 않는다. |
| goal | 같은 release 재실행이 중복 변경 없이 이전 실패에서 이어감. |
| non-goal | 임의 자동 배포·DB rollback 보장. |
| 추적 작업 | P17-01, P17-02 |
| 책임 모듈 | M11 배포 entrypoint, M01 설정, M10 manifest/evidence, M07 상태. |
| 선행/게이트 | P16-09 및 P01/G02, G10; O08–O11의 해당 선행 결과. P16-08과 P16-09는 같은 카드지만 실행 순서 유지. |
| 코드 동작 | 1) release ID·revision·SDK/runtime·NuGet/PS·계약/bridge·migration·image/media hash 수집. 2) 비밀 제외 manifest 검증. 3) 단계별 사전 상태 조회 후 적용. 4) 재실행 시 목표 상태/실제 자원을 대조하고 완료 단계를 건너뛰거나 불일치를 중단. 5) 실패 시 명시적 복구 지점·호환성 기록. |
| 목표-input | `releaseManifest`는 ID/revision/runtime·package/contract·bridge/image hash와 호환범위, `installedState`는 설치·설정·schema 버전, `stepState`는 단계 적용 여부, `migrationApproval`은 승인 DB 변경 경로다. artifact·제품 조회·배포안 원천; hash/버전 누락·호환성 불명은 차단. |
| 목표-output | `verifiedManifest`는 검증 release/hash, `stepResult`는 적용/no-op/실패, `versionDelta`는 현재/목표 차이, `recoveryCheckpoint`는 복구 지점이다. M11/M10/M07 생산; O13 호환검사·O14 증거 인수 소비. 미확인 단계는 미완료. |
| 코드동작확인 Test방식 | 정상: 고정 manifest·빈/예상 상태→적용→모든 version/hash 기록. 거부: manifest 누락/불일치/비호환→변경 전 차단. 중복: 동일 release 재적용→실제 상태 조회→부작용 없는 no-op. 실패·복구: 중간 단계 실패→실행상태 보존·다시 읽기→실패 단계부터 승인된 복구, 중복 설치/마이그레이션 없음. 실호스트 배포는 별도 승인/직렬. |
| 코드동작확인 로깅방식 | release.manifest_verified/step_started/applied/skipped/failed; release ID, revision, artifact hash, schema/contract version, step, outcome, actor. 진단·감사·manifest 증거 분리; 비밀/경로 민감부 제거. 해시 재계산·목록 완전성 검증. |
| 실패·복구 | 파일 rollback이 DB rollback을 의미하지 않는다. 부분 migration은 복원/호환 절차가 명확할 때만 진행. 알 수 없는 현 상태를 덮어쓰지 않는다. |
| 완료 조건 | 계약/모의에서 manifest 검증·no-op/재개; 실호스트는 재현 배포·버전 해시·migration 결과 증거. |
| 인계 정보 | O13 복원 대상 version/schema/image hash, O14 실행 manifest·미검증 항목. |

## O13 — backup/restore·부팅·가동모드

| 필드 | 계약 |
|---|---|
| 목적 | 역할별 백업을 복원하고 AD→Storage→VMM/Compute→고객 VM→API 순서와 Windows8/Linux8/4+4 상호배제 모드를 확인한다. |
| 추가 범위 | 백업목록·복원 순서·복원 후 reconcile, readiness 기반 startup, mode transition/자동시작 제한. |
| 수정 범위 | 복구되는 업무 원장·VM binding·신원 참조·mode 상태를 실제 자원과 재조정. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. 향후 이 카드에서 자동 삭제 정책을 추가하지 않는다. 백업원본·고객 데이터 자동 제거 금지. |
| goal | 검증된 복구로 실제 상태와 원장을 정합시키고 허용 모드를 준수. |
| non-goal | 같은 물리 디스크 복구본을 독립 백업으로 판정, 타이머 기반 준비 판정. |
| 추적 작업 | P17-03, P17-04, P17-05 |
| 책임 모듈 | M11 주책임; M07 원장·M06 reconcile·M10 증거·M01 mode. |
| 선행/게이트 | P17-02, O12; G01/G07/G10; O06/O07 mode 결과. |
| 코드 동작 | 1) AD·SQL/VMM·작업기록·이미지·고객VM·인증서 backup scope/version/매체를 확인. 2) 격리 복원하고 무결성 검사. 3) Worker/API 실행 전에 실제 VMM Job/VM과 복원 원장/outbox를 대조, recovering 표시. 4) 호스트 부팅은 역할 readiness 통과마다 진행. 5) mode 전환 전 이전 모드 VM/작업을 정리하고 상호배제·자동시작 상태 확인. |
| 목표-input | `backupManifest`는 범위/시각/hash/위치(O02/O05 복구본 등), `restoreOrder`는 역할 의존순서, `releaseCompatibility`는 O12의 version/schema/hash 재현 근거, `actualExecutionState`는 실제 Job/VM, `modeState`는 profile/실행/자동시작 목록, `restoreApproval`은 허용 시험범위다. 복구본은 O02/O05·승인 백업, 버전 근거는 O12, 실상태는 M06/제품 조회에서 온다. 같은 물리PC 별도 볼륨은 데이터 보존 복원에 쓸 수 있으나 물리장애 독립본은 아니다. S2D 풀/원본 논리 디스크 분리 및 무결성·호환성 미확인 시 거부. |
| 목표-output | `restoreItemResult`는 항목별 hash/복원, `reconciledBinding`은 원장/VM 연결, `readinessState`는 역할 준비, `activeMode`/`roleCount`는 프로필/가동대수, `dataVerification`은 고객 데이터다. M11/M06/M07 생산, O14 소비. partial/unavailable 분리. |
| 코드동작확인 Test방식 | 정상: 검증 backup fixture→격리 복원→원장/실VM 대조 후 단계적 시작→서비스 준비 증거. 거부: 불완전/손상/hash 불일치/비호환 backup 또는 충돌 모드→복원/기동 차단. 중복: 같은 restore manifest 재요청→복원 대상·기존 VM 재조회→중복 VM 생성 없음. 실패·복구: 각 의존 역할/restore 단계 실패→후속 서비스 중지, 원본 보존·재개 정보 저장, 원장 stale이면 신규 변경 제한. 실호스트 복원·재부팅·모드 전환은 서로 직렬. |
| 코드동작확인 로깅방식 | backup.catalog_verified/restore.started/item_verified/reconcile_required/startup.ready/mode.transition; run/release, backup ID/hash, role, stage, job/VM mapping, observed_at. 원장·감사·증거·제품 로그 분리. 비밀·내부주소 마스킹. backup hash, 서비스 readiness, 동시 가동 수 검증. |
| 실패·복구 | DB 복원 직후 worker를 자동 실행하지 않는다. 실제 VM과 오래된 원장을 먼저 대조. 미검증 restore point에 원본 덮어쓰기 금지. 모드 충돌은 fail closed. |
| 완료 조건 | 계약·모의에서 의존성/중복방지; 실호스트 격리 restore·cold boot·mode transition과 고객 데이터 대조를 각각 인수. |
| 인계 정보 | O14 요구사항 상태표, 백업/복구 manifest, 실호스트 순서·실패 항목·복구 담당과 남은 조건. |

## O14 — 최종 요구사항·증거 인수

| 필드 | 계약 |
|---|---|
| 목적 | 전 요구사항을 구현/시험/미검증/보류로 판정하고 설치·운영·장애·복원 재현에 필요한 증거를 전달한다. |
| 추가 범위 | 요구사항-작업-계약-시험-증거 추적, 공개 마스킹본·manifest·한계·후속조치. |
| 수정 범위 | 판정 상태와 증거 연결/인계 기록. |
| 삭제 범위 | 현재 구현 코드가 없어 삭제 대상 없음. 향후 이 카드에서 자동 삭제 정책을 추가하지 않는다. 불리한/실패한 기록을 감추거나 원본을 지우는 범위 없음. |
| goal | 독립 실행자가 결과·미검증 범위·재현 조건을 판별. |
| non-goal | 문서/모의 결과를 실호스트 완료로 승격하거나 사용자 결정 게이트를 통과 처리. |
| 추적 작업 | P17-06, P17-07 |
| 책임 모듈 | M10 주책임; M11 절차/증거, M01/M07 버전/기록. |
| 선행/게이트 | P17-05, O12/O13, G01–G10 및 X01–X04 판정 유지. |
| 코드 동작 | 1) 작업·요구사항 ID 전체 목록 대조. 2) 각 항목을 evidence hash/run·시험방식·환경·기대/실제·한계와 연결. 3) 기밀/개인정보 마스킹·공개 검사를 한다. 4) 누락/상충은 보류 또는 미검증으로 남기고 후속 담당/조건을 기록. 5) 재현 가능한 절차 및 운영 책임 인계. |
| 목표-input | `traceMatrix`는 요구사항-P-계약, `runEvidence`는 시험방식·환경·기대/실제/hash, `releaseContext`는 O12 버전/mode/product 근거, `gateDecision`은 G 상태, `externalHandoff`는 X 결과, `redactionPolicy`는 공개 허용/원본 보존 규칙이다. 계획·원장·실행증거·공식자료 원천; 증거 없는 완료는 미검증/보류. |
| 목표-output | `requirementStatus`는 구현/통과/미검증/보류, `evidenceIndex`는 run/hash와 공개본, `limitation`은 지원/실습 경계, `openCondition`은 미결 G/X, `handoffGuide`는 재현/복구 순서다. M10/M11 생산; 사용자·운영자·Cloud 소비. 누락/상충은 완료 대신 보류. |
| 코드동작확인 Test방식 | 정상: 완결 manifest fixture→모든 요구사항과 증거 연결→hash/메타/마스킹 검증. 거부: 근거 없는 실호스트 완료 주장·필수 증거 누락→완료 인수 차단. 중복: 같은 실행 증거 재수집→ID/hash 일치, 기존 판정 덮어쓰기 금지. 실패·복구: 공개 검사에서 비밀 탐지/상충 증거→배포 중단, 마스킹·근거 보완 후 재검증. 모의와 실호스트 성취 분리 검사. |
| 코드동작확인 로깅방식 | acceptance.requirement_classified/evidence_verified/redaction_failed/handoff_published; requirement ID, P ID, run/release, test class, evidence hash, reviewer/actor, outcome. 공개본은 안전 필드 allowlist·원본은 제한 저장. 개인키·실제 IP·토큰 검사, 모든 상태의 근거 링크 확인. |
| 실패·복구 | 원본 증거는 보존 권한 내 유지. 누락/실패/미검증을 완료로 승격하지 않는다. 사용자 결정과 Cloud 소유 작업은 별도 open item으로 넘긴다. |
| 완료 조건 | 계약: 추적표·상태 용어 확정; 모의: 누락/마스킹/정합성 검증; 실호스트 인수는 각 P의 실제 증거가 있을 때만 해당 항목에 한정. |
| 인계 정보 | 공개 가능한 결과, 비공개 증거 위치/권한·hash, G01–G10 사용자 결정, X01–X04 외부 결과, 미해결 작업 및 재현 순서. |

## 운영 충돌·직렬 실행

카드 문서·계약·fixture 작업은 코드 의존성이 해결되는 범위에서 병렬화할 수 있다. 실호스트 변경은 단일 물리 호스트/중첩 자원/공유 데이터에 단 하나의 활성 변경만 허용하며, 특히 다음 구간은 직렬 실행한다.

- O01 구성 및 O02 Storage 장애 주입/복귀는 동일 저장소 자원을 점유한다.
- O03 계획 이동, O04 장애/복귀는 클러스터·VM 실행 상태를 변경한다.
- O05 데이터 컷오버와 O06 호스트 재부팅은 접수 잠금부터 검증/해제까지 독점한다.
- O07 Linux8 시험과 인프라8 시험은 동시 가동 금지. 4+4 전환은 양 모드 복구 확인 후 수행한다.
- O09 서비스 복구, O10 인증서 교체, O11 자원 고갈/관리자 인계, O12 배포, O13 복원·부팅·가동모드는 각 점검창에서 다른 실호스트 변경과 겹치지 않는다.

M03 작업 잠금/접수 중단, M11 운영 run 잠금, 실제 제품 상태 재조회로 충돌을 감지한다. 시간초과나 lease 만료는 잠금 해제·재실행의 증거가 아니다. **이 문서의 추적 주책임은 P10 9개, P11 9개, P12 10개, P13 5개, P16 9개, P17 7개로 총 49개다.** 같은 선행/검증 P ID가 다른 카드의 본 추적목록에 중복 배정되지 않았으며, 위에서 선행 참조로 재등장할 수 있다. G01–G10/X01–X04는 원래 이름과 외부 의존성을 유지한다.
