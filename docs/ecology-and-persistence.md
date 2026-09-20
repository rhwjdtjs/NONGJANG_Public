# 원거리 생태와 저장

[문서 처음](../README.md) · [전체 구조](architecture.md) · [검증 범위](development-and-validation.md)

## 화면 밖에서도 이어지는 개체 상태

넓은 섬에서 모든 동물을 항상 같은 정밀도로 움직이면 비용이 커집니다. 근처에서는 실제 Actor와 감각·이동을 사용하고, 먼 개체는 영속 ID를 가진 기록으로 전환해 욕구와 자원 상태를 계산합니다.

```mermaid
flowchart LR
  A["근거리 동물 Actor"] -->|"거리·스트리밍 조건"| B["영속 ID와 상태 기록"]
  B --> C["원거리 욕구·유한 자원 계산"]
  C --> B
  B -->|"지형 준비·기록 검증"| A
  B --> D["월드 저장"]
  D --> B
```

`UNNWorldLedger::SleepActor`가 기록을 만들고 `WakeActor`가 준비된 위치에 복원합니다. `RegionReady`는 World Partition 준비 상태를 확인하며, 복원 과정에서 개체 ID와 물품 소유 기록을 다시 검증합니다. 상태를 전환해도 기존 경험·관계·명령이 다른 개체의 것으로 바뀌지 않도록 연결합니다.

## 유한한 먹이와 수원

`AdvanceRemoteEcology`는 원거리 동물과 서식지 기록, 활성 상태의 자원과 관련 정보를 모아 계산합니다. `FNNEcologySimulation::Advance`, `AdvanceNeeds`, `Regrow`가 욕구 변화와 소비·재생을 다룹니다. 결과를 기록 형식에 맞게 검사한 뒤 반영합니다.

원거리 처리는 화면 안의 물리·감각을 완전히 동일하게 재연하는 방식이 아닙니다. 활성 개체와 데이터 개체의 경계를 구별하고, 유한 자원과 영속 상태의 연속성을 관리합니다. 장기간 개체군 균형, 모든 종의 행동 동등성, 최종 프레임 성능은 별도의 검수 대상입니다.

| 역할 | 원본 프로젝트 근거 |
|---|---|
| 페이징 | `Source/NONGJANG/NNWorldPaging.cpp` · `SleepActor`, `WakeActor`, `ProcessPaging`, `RegionReady` |
| 원거리 통합 | `Source/NONGJANG/NNWorldEcology.cpp` · `AdvanceRemoteEcology` |
| 생태 계산 | `Source/NONGJANG/NNEcologySimulation.cpp` · `Advance`, `AdvanceNeeds`, `Regrow` |
| 형식화된 상태 | `Source/NONGJANG/NNEcologyRecords.cpp` · `FNNEcologyRecordCodec::Capture`, `Apply`, `Read`, `Write` |

## 두 저장 세대와 복원 검증

`UNNSaveSubsystem`은 월드와 플레이어 상태를 캡처해 구조를 검증합니다. 두 저장 세대를 번갈아 기록하고, 저장한 결과를 다시 읽어 월드 ID와 세대를 확인합니다. 로드할 때는 유효한 기록 중 최신 세대를 선택합니다.

2026-09-20 개발본의 저장 스키마는 **10**입니다. 이전 구현 기록의 8·9는 당시 상태이며 현재값으로 제시하지 않습니다.

근거는 `Source/NONGJANG/NNSaveGame.cpp`의 `CaptureCurrentWorld`, `CommitWorld`, `LoadWorld`, `RestoreHomeWorld`와 `NNSaveGame.h`입니다. Actor별 직렬화·복원은 `NNWorldSnapshot.cpp`의 `FNNWorldPersistence::CaptureActor`, `ApplyActor`, `Capture`, `Restore`가 담당합니다.

이 구조만으로 모든 전원 차단이나 저장 매체 손상에 대한 무손실을 보장하지는 않습니다. 실제로 확인한 저장·재실행 시나리오와 아직 시험하지 않은 장애 조건을 나누어 기록합니다.

## 교역소 방문과 물품 소유 상태

최대 2인 리슨 서버 방문에서는 출발 전 기록을 남기고 원래 인벤토리를 잠급니다. 호스트가 입장을 승인한 뒤 서버 기준으로 활동하고, 최종 상태와 귀환 기록을 저장한 후 본인 섬을 복원합니다. 같은 귀환 결과를 반복 수신해도 물품을 다시 지급하지 않도록 단계와 확인을 연결합니다.

```mermaid
flowchart LR
  A["교역소 방문 요청"] --> B["출발 기록 저장·물품 잠금"]
  B --> C["세션 접속·호스트 승인"]
  C --> D["서버 기준 활동"]
  D --> E["호스트 최종 상태 저장"]
  E --> F["방문자 귀환 기록 저장"]
  F --> G["수신 확인·본인 섬 복원"]
```

원본 근거:

- `Source/NONGJANG/NNVisitCoordinator.cpp`: `BeginVisit`, `CanAcceptConnection`, `Handshake`, `RequestReturn`, `AcknowledgeReturn`, `HandleDisconnect`.
- `Source/NONGJANG/NNVisitChannel.cpp`: 소유 PlayerController의 RPC와 방문 데이터 전송.
- `Source/NONGJANG/NNVisitJournal.cpp`: `Prepare`, `Depart`, `ReceiveReturn`, `ResolveForHomeEntry`.
- `Plugins/NNOnline/Source/NNOnline/Private/NNOnlineSessionSubsystem.cpp`: 세션 생성·검색·주소 결정.

## 재실행으로 확인한 범위

2026-09-19 실제 섬에서는 준비된 늑대 관계·명령을 저장하고 별도 프로세스의 이어하기로 확인했습니다. 약 4.6km 거리 전환으로 페이징을 유발하고, 같은 개체의 관계·명령을 복원하는 시나리오입니다. 신뢰를 처음부터 얻는 과정이나 다수 개체의 상시 렌더링 성능 시험은 아닙니다.

2026-09-14 LAN 검수에서는 같은 PC의 독립 게임 프로세스로 방문·물품 버리기·정상 귀환과 방문자 재실행 복구를 확인했습니다. 시험 맵의 교역소와 해금 조건을 준비했으며, 실제 두 PC·Steam 두 계정·강제 종료·전원 차단·Shipping 검증과 구별합니다. 구체적인 수치와 준비 조건은 [검증 문서](development-and-validation.md)에 기록했습니다.
