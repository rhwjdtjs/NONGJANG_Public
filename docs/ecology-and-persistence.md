# 원거리 생태와 저장

[문서 처음](../README.md) · [전체 구조](architecture.md) · [검증 범위](development-and-validation.md)

기준일: 2026-10-01. 원본 코드의 구조와 실제로 확인한 저장·재실행 시나리오를 정리했습니다.

## 화면 밖의 개체 상태

근처 동물은 Actor의 감각·이동을 사용하고 먼 개체는 영속 ID를 가진 기록으로 전환합니다. 원거리에서는 욕구와 유한 자원을 계산합니다.

```mermaid
flowchart LR
  A["근거리 동물 Actor"] -->|"거리·스트리밍 조건"| B["영속 ID와 상태 기록"]
  B --> C["원거리 욕구·유한 자원 계산"]
  C --> B
  B -->|"지형 준비·기록 검증"| A
  B --> D["월드 저장"]
  D --> B
```

`UNNWorldLedger::SleepActor`가 기록을 만들고 `WakeActor`가 준비된 위치에 복원합니다. `RegionReady`는 World Partition 준비 상태를 확인합니다. 복원 때 개체 ID와 물품 소유 기록을 검증하며 경험·관계·명령도 함께 연결합니다.

## 유한 자원과 개체군 회복

`AdvanceRemoteEcology`는 원거리 동물과 서식지, 활성 자원의 정보를 모아 계산합니다. `FNNEcologySimulation::Advance`, `AdvanceNeeds`, `Regrow`가 욕구 변화와 소비·재생을 다루며 결과를 검사한 뒤 반영합니다.

9월 말에는 상한 사체를 일정 시간이 지난 뒤 같은 종의 새 개체 상태로 홈에 돌려놓는 회복 규칙을 추가했습니다. 현재 소스는 고기량에 따라 회복 대기 시간을 조정하고 관계·명령·경험을 초기화합니다. 섬의 개체군을 유지하는 게임 규칙이며 실제 번식 과정이나 장기 균형은 추가 작업입니다.

원거리 계산은 화면 안의 물리·감각을 같은 정밀도로 재연하지 않습니다. 근거리와 원거리 사이의 상태 보존, 전 종의 행동과 장시간 자원·포식 균형을 별도로 검수합니다.

| 역할 | 원본 프로젝트 근거 |
|---|---|
| 페이징 | `NNWorldPaging.cpp` · `SleepActor`, `WakeActor`, `ProcessPaging`, `RegionReady` |
| 원거리 통합 | `NNWorldEcology.cpp` · `AdvanceRemoteEcology` |
| 생태 계산 | `NNEcologySimulation.cpp` · `Advance`, `AdvanceNeeds`, `Regrow`, `Recover` |
| 상태 읽기와 쓰기 | `NNEcologyRecords.cpp` · `FNNEcologyRecordCodec::Capture`, `Apply`, `Read`, `Write` |

## 두 저장 세대

`UNNSaveSubsystem`은 월드와 플레이어 상태를 캡처해 검증합니다. 두 저장 세대를 번갈아 기록하고 저장한 결과를 다시 읽어 월드 ID와 세대를 확인합니다. 로드할 때는 유효한 기록 중 최신 세대를 선택합니다. 2026-10-01 확인한 `NNSaveGame.h`의 `CurrentSchema`는 10입니다.

근거는 `NNSaveGame.cpp`의 `CaptureCurrentWorld`, `CommitWorld`, `LoadWorld`, `RestoreHomeWorld`입니다. Actor별 직렬화·복원은 `NNWorldSnapshot.cpp`의 `FNNWorldPersistence::CaptureActor`, `ApplyActor`, `Capture`, `Restore`가 담당합니다.

정상 저장과 새 프로세스의 복원은 여러 시나리오에서 확인했습니다. 신규 총기·운반·차량 상태를 포함한 통합 저장과 강제 종료·전원 차단·저장 매체 손상 검수는 남아 있습니다.

## 다른 섬 방문과 물품 소유

정식판의 최대 2인 리슨 서버 방문은 출발 전 기록을 남기고 원래 인벤토리를 잠급니다. 호스트 승인 후 서버 기준으로 활동하고 최종 상태와 귀환 기록을 저장한 뒤 본인 섬을 복원합니다. 귀환 결과를 반복 수신해도 물품을 다시 지급하지 않도록 단계와 확인을 연결합니다.

```mermaid
flowchart LR
  A["방문 요청"] --> B["출발 기록 저장·물품 잠금"]
  B --> C["세션 접속·호스트 승인"]
  C --> D["서버 기준 활동"]
  D --> E["최종 상태·귀환 기록 저장"]
  E --> F["수신 확인·본인 섬 복원"]
```

원본의 `NNVisitCoordinator.cpp`가 입장·승인·귀환·연결 중단을 처리합니다. `NNVisitChannel.cpp`는 소유 PlayerController의 RPC, `NNVisitJournal.cpp`는 출발·귀환·복원 기록을 다룹니다. 세션 생성·검색·주소 결정은 `Plugins/NNOnline/Source/NNOnline/Private/NNOnlineSessionSubsystem.cpp`에 있습니다.

[공개 데모](release-plan.md)는 싱글 전용으로 설계했습니다. 데모 제한과 정식판 방문 기능의 배포·검수 범위를 구분합니다.

## 재실행으로 확인한 범위

2026-09-19 실제 섬에서는 준비된 늑대 관계·명령을 저장하고 별도 프로세스의 이어하기로 확인했습니다. 약 4.6km 거리 전환으로 페이징을 유발한 뒤 같은 개체의 관계·명령을 복원했습니다. 신뢰를 처음부터 얻는 과정은 이 검수에 포함하지 않았습니다.

2026-09-14 LAN 검수는 같은 PC의 독립 게임 프로세스와 준비된 시험 맵을 사용했습니다. 방문·물품 버리기·정상 귀환과 방문자 재실행 복구를 확인했습니다. 실제 두 PC·Steam 두 계정·강제 종료·Shipping 빌드는 추가 검수 대상입니다. 날짜별 조건은 [검증 문서](development-and-validation.md)에 기록했습니다.

본문의 파일은 별도 표시한 플러그인 경로를 제외하면 비공개 원본의 `Source/NONGJANG/` 아래에 있습니다.
