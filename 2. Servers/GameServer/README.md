<div align="center">

# 🗺️ Game Server

**n x n 격자 기반 동기화 중심 MMORPG 게임 서버**

플레이어 주변 3×3 섹터의 상태를 멀티 스레드로 안정적으로 동기화하는 게임 서버

</div>

## 0. 문서 개요 (Overview)

| 항목 | 내용 |
| :--- | :--- |
| **역할** | n × n 격자 월드에서 플레이어 주변 3×3 섹터의 상태를 실시간으로 동기화하는 멀티 스레드 게임 서버 |
| **핵심 포인트** | 섹터/그룹 단위 월드 파티셔닝, 스레드 간 이동 및 전투 동기화, 네트워크 라이브러리와의 밀접한 연동 |
| **동기화 단계** | ① 단일 스레드 대륙 기반 서버 → ② 심리스 멀티 스레드 서버(이동/3×3 전파) → ③ 공격·피격까지 포함한 완전 멀티 스레드 서버 |
| **관련 변경점** | 스레드 이동 시 메시지 보존, 그룹 간 메시지 전달, 스레드 경계 섹터 락 관리 등 네트워크 라이브러리 확장 |

## 1. 프로젝트 개요

이 게임 서버는 **n × n 격자 공간**에 배치된 플레이어들의 위치를 기준으로,  
각 플레이어 주변 **3×3 섹터** 내 다른 플레이어의 움직임과 상호작용을 실시간으로 보여주는 서버입니다.

멀티 스레드 환경에서의 **동기화**를 핵심 목표로 삼았으며,  
아래와 같이 점진적으로 난이도를 높여가며 서버를 설계·구현했습니다.

1. **대륙 기반 서버**: 하나의 스레드가 담당하는 영역 내에서만 이동 및 3×3 섹터 생성/삭제를 전파
2. **심리스(Seamless) 서버**: 서로 다른 스레드 간 이동과 3×3 섹터 생성/삭제까지 전파
3. **완전 멀티 스레드 서버**: 서로 다른 스레드 간 공격·피격까지 포함한 상호작용 동기화

## 2. 핵심 특징

| 영역 | 설명 |
| :--- | :--- |
| **격자 월드 구조** | n × n 셀을 기본 단위로, 여러 셀을 묶은 섹터를 기준으로 관심 영역을 관리합니다. |
| **3×3 AOI(관심 영역)** | 플레이어를 기준으로 주변 3×3 섹터에 속한 캐릭터들의 이동·공격 정보를 전파합니다. |
| **멀티 스레드 스케일 아웃** | 섹터 그룹(대륙)을 스레드 단위로 분할하여, 코어 수 증가에 따라 선형적인 처리량 확장을 목표로 합니다. |
| **단계적 동기화 설계** | 단일 스레드 영역 → 스레드 간 이동 동기화 → 공격 상호작용까지 포함한 완전 동기화 순으로 난이도를 높여가며 구현했습니다. |
| **네트워크 라이브러리 연동** | 기존 네트워크 라이브러리에 **스레드 간 Job 전달 및 그룹 기능**을 활용·확장하여 게임 서버에 최적화했습니다. |

![그룹 플로우](../../Images/GroupFlow.png)
![그룹 플로우](../../Images/GroupFlowSecond.png)

## 3. 동기화 설계 진화 단계

### 3.1 1단계 – 대륙 기반 게임 서버

> 목표: **스레드 간 공유 상태를 완전히 차단**하고, 한 스레드 안에서만 일관성을 보장하는 구조 검증

- 월드를 여러 개의 **대륙(Continent)** 으로 나누고, **대륙 하나당 하나의 워커 스레드**가 전담합니다.
- 경계를 넘어갈 때는 이전 대륙의 플레이어가 사라지고, 새 대륙의 플레이어가 소환됩니다. 이후 새 대륙에 속한 상태로 해당 스레드에서 계속 업데이트됩니다.
- 동기화는 단일 스레드 내 컨테이너 기반으로 처리되므로, 락 없이도 일관성이 유지됩니다. 스레드 이동 자체는 네트워크 라이브러리의 그룹 기능을 통해 안전하게 이루어집니다.

<img width="320" height="240" alt="Image" src="https://github.com/user-attachments/assets/a00b46c2-02f5-4d89-a10c-e60ecaa4de78" />

- 문제점: 경계를 넘어가는 시점에 이동 요청 패킷이 동시에 도착하면 서버에서 이동 요청이 무시되는 현상
  - 원인: 스레드 이동 시, 이전 스레드에서 받은 메시지를 무시하도록 구현된 네트워크 라이브러리 로직. 각 스레드는 서로 다른 콘텐츠를 실행할 수 있기 때문에, 설계 당시에는 “이전 스레드의 메시지는 다른 콘텐츠에 속한다”고 보고 폐기하는 것이 안전하다고 판단했습니다.
  - 해결: 스레드 이동 시 **이전 스레드의 Pending 메시지를 먼저 처리한 뒤** 스레드 이동 로직을 실행하도록 구조를 변경하였습니다.

- 의사 결정: 네트워크 라이브러리의 그룹 기능 활용
  - 콘텐츠 레벨에서 직접 스레드를 만들고 프레임에 맞춰 돌리는 구조도 가능하지만, “간편하게 사용할 수 있는 라이브러리”라는 목표에 맞춰 이미 존재하는 그룹 기능을 활용했습니다.
  - 네트워크 라이브러리의 그룹 기능에는 그룹 입장/퇴장 콜백, 그룹에 포함된 세션의 메시지 콜백, 지정한 프레임 레이트에 맞는 `Update` 호출 등이 포함되어 있습니다.

- 의사 결정: 이전 그룹 메시지 처리 허용
  - 원래 그룹 기능은 인스턴스 던전, PVP 등 **서로 다른 콘텐츠를 분리**하는 용도로 설계했기 때문에, 그룹 이동 시 이전 그룹의 메시지를 폐기하는 것이 타당한 동작이라고 판단했었습니다.
  - 그러나 현재 게임 서버처럼 **동일한 콘텐츠 내에서 그룹(스레드)을 자주 이동하는 구조**에서는, 이전 그룹에서 쌓인 메시지도 정상적으로 처리되어야 합니다.
  - 따라서 그룹 이동 시 이전 그룹의 메시지에 대해서도 `OnRecv`가 호출되도록 변경했고, 실제로 다른 콘텐츠라면 패킷 타입 등을 통해 콘텐츠 레벨에서 필터링하도록 설계했습니다.

- 스레드 간 이동
```cpp
// 기존에 있던 섹터에서 삭제
RemovePlayerFromSector(player, player->curSector);

// 기존 플레이어에게 삭제 메시지 전송
int preIdx = GetPreSectorIdx(player->preSector.x, player->preSector.y, player->x / dfTILE_SIZE, player->y / dfTILE_SIZE);
SECTOR_AROUND& preSectors = preSectorAround[player->y / dfTILE_SIZE][player->x / dfTILE_SIZE][preIdx];

SendDeleteCharacterToAroundAndDelete(player, preSectors);

int groupX = player->x / groupUnitX;
int groupY = player->y / groupUnitY;

// 스레드 이동
server->RegisterSessionToGroup(player->sessionId, GetGroupIndex(groupX, groupY));
```

### 3.2 2단계 – 심리스(Seamless) 멀티 스레드 서버

> 목표: **대륙/스레드 경계를 넘는 3×3 섹터의 생성/삭제 이벤트 전파**

- 한 스레드에 속한 플레이어가 경계선을 넘을 경우:
  - 기존 스레드 소유 섹터에서 플레이어 제거
  - 새 스레드 소유 섹터로 캐릭터 마이그레이션
  - 인접 3×3 섹터의 다른 플레이어들에게 **Spawn/Despawn 이벤트** 전파
- 스레드 간에는 직접 공유 메모리를 사용하지 않고,
  - **Message Passing** 으로만 상태를 전달합니다.

<img width="320" height="240" alt="Image" src="https://github.com/user-attachments/assets/2e5f734f-09ff-4692-9e8f-0c5fe1b897e4" />

- 문제점: 스레드 간 이동 시 삭제 메시지를 전파하지 않아, 클라이언트에 캐릭터가 남아 있는 현상
  - 원인 1: 이동 시 “내가 사라져야 하는 섹터”를 계산하는 함수 호출에 잘못된 위치 인자를 전달
    - 해결 1: 해당 상황을 재현하여 로직이 도는 순서를 추적했고, 잘못된 인자가 들어가는 지점을 찾아 수정하였습니다.
  - 원인 2: 삭제 메시지를 전파한 이후, 다른 플레이어가 섹터 가시권에 새로 진입하여 생성 메시지를 받습니다. 이후 원래 플레이어가 섹터에서 나가면서 삭제되지만, “나중에 들어온 플레이어”는 삭제 메시지를 받지 못합니다.
    - 해결 2: **섹터 리스트에서 먼저 제거한 뒤에** 삭제 메시지를 전파하도록 순서를 조정하였습니다.

- 의사 결정: 동기화 범위
  - 그룹(스레드) 경계에 위치한 섹터는 다른 스레드에서도 참조될 수 있습니다. 따라서 이 섹터들에 대해서만 동기화를 걸어 사용하도록 설계했습니다.
  - 그룹 경계가 아닌 섹터는 하나의 스레드에서만 접근이 보장되므로, 별도의 락 없이 사용합니다.

- 섹터의 `isLock` 정보를 통해 스레드 경계 섹터를 구분하고, 필요한 경우에만 동기화를 걸어 로직 실행
```cpp
void AddPlayerToSector(PLAYER* player, const SECTOR_POS& sector) {
  if (sector.isLock)
    AcquireSRWLockExclusive(&sectorListLock[sector.y][sector.x]);

  sectorList[sector.y][sector.x].push_back(player);

  if (sector.isLock)
    ReleaseSRWLockExclusive(&sectorListLock[sector.y][sector.x]);
}

void RemovePlayerFromSector(PLAYER* player, const SECTOR_POS& sector) {
  if (sector.isLock)
    AcquireSRWLockExclusive(&sectorListLock[sector.y][sector.x]);

  for (auto it = sectorList[sector.y][sector.x].begin(); it != sectorList[sector.y][sector.x].end(); ++it) {
    if (*it == player) {
      *it = sectorList[sector.y][sector.x].back();
      sectorList[sector.y][sector.x].pop_back();
      break;
    }
  }

  if (sector.isLock)
    ReleaseSRWLockExclusive(&sectorListLock[sector.y][sector.x]);
}

void SendAroundPlayerInfoAndMyInfo(PLAYER* player) {
  SECTOR_AROUND* around = &sectorAround[player->curSector.y][player->curSector.x];
  GetAroundLock(around);
  for (int i = 0; i < around->count; i++) {
    for (int j = 0; j < sectorList[around->sectors[i].y][around->sectors[i].x].size(); j++) {
      // 플레이어 메시지 전파
    }
  }
  ReleaseAroundLock(around);
}
```

### 3.3 3단계 – 완전 멀티 스레드 전투 서버

> 목표: **스레드 간 공격·피격 상호작용** 포함, 멀티 스레드 환경에서 게임 플레이 전체를 일관성 있게 유지

- 각 캐릭터(또는 섹터)에 대해 **오너십(Owner Thread)** 을 정의하여,
  - HP/상태 변화는 반드시 Owner Thread에서만 최종 반영되도록 설계했습니다.
- 공격자가 다른 스레드에 속한 대상을 공격할 경우:
  - 공격 스레드 → 대상 Owner Thread로 **Attack Request Job** 전달
  - Owner Thread에서 피격 판정, HP 감소, 상태 이상 처리
  - 결과를 다시 3×3 섹터 내 플레이어들에게 브로드캐스트
 
<img width="320" height="240" alt="Image" src="https://github.com/user-attachments/assets/2196c399-fea0-4926-af0a-89bd9bc1e013" />

- 의사 결정: 스레드 간 메시지 전달 방식
  - 그룹은 네트워크 라이브러리의 기능이며, 사용자 입장에서는 내부 구현을 알기 어렵습니다. 따라서 사용자가 직접 스레드 간 메시지 큐를 구현하기보다는, **라이브러리 차원에서 메시지 전달 기능을 제공하는 것**이 더 적절하다고 판단했습니다.
  - 멀티 스레드 게임 서버의 목표인 공격/피격 동기화를 위해서는 빠른 메시지 처리가 중요하다고 보고, `Update` 이전에 메시지를 우선 처리하여 **한 프레임 이전 상태 기준으로 로직을 적용**할 수 있도록 설계했습니다.

- 다른 스레드 간 메시지 전달
```cpp
void PushAttackMessageToGroup(short x, short y, int attackType, char dir, int attackId) {
  AttackSpec aSpec;

  if (!GetAttackSpec(attackType, aSpec)) {
    return;
  }

  int groupX = x / (dfRANGE_MOVE_RIGHT / dfGROUP_COUNT_X);
  int groupY = y / (dfRANGE_MOVE_BOTTOM / dfGROUP_COUNT_Y);
	
  for (int i = 0; i < 4; i++) {
    if (dfRANGE_MOVE_LEFT <= rx[i] && rx[i] < dfRANGE_MOVE_RIGHT && dfRANGE_MOVE_TOP <= ry[i] && ry[i] < dfRANGE_MOVE_BOTTOM) {
      int rGroupX = rx[i] / (dfRANGE_MOVE_RIGHT / dfGROUP_COUNT_X);
      int rGroupY = ry[i] / (dfRANGE_MOVE_BOTTOM / dfGROUP_COUNT_Y);

      if (rGroupX != groupX || rGroupY != groupY) {
        int idx = GetGroupIndex(rGroupX, rGroupY);
        // 다른 그룹에 메시지 전송
        server->PushMessageToGroup(buffer, idx);
      }
    }
  }
}

class GameGroup : ICNetServerGroup {
  // 다른 그룹으로부터 받은 메시지 처리
  virtual void OnGroupRecv(CSerializedBuffer* buffer) {
    unsigned char type;
    *buffer >> type;

    switch (type)
    {
    case dfPACKET_SC_ATTACK1:
      AttackOtherGroupProc(x, y, 1, dir, attackId);
      break;
    case dfPACKET_SC_ATTACK2:
      AttackOtherGroupProc(x, y, 2, dir, attackId);
      break;
    case dfPACKET_SC_ATTACK3:
      AttackOtherGroupProc(x, y, 3, dir, attackId);
      break;
    default:
      break;
    }
  }
};
```

## 4. 네트워크 라이브러리 변경점

게임 서버를 구현하며 기존 네트워크 라이브러리에 다음과 같은 기능을 추가/개선했습니다.

| 변경 항목 | 설명 |
| :--- | :--- |
| **세션 스레드 이동 시 이전 메시지 실행 보장** | 스레드 이동 중 도착한 메시지가 전부 무시되면, 콘텐츠에 따라 치명적인 문제가 발생할 수 있습니다. 스레드 이동 시 Pending 메시지를 먼저 처리하거나, 콘텐츠에서 직접 이전 메시지를 처리할 수 있는 인터페이스를 제공하여 유연하게 대응할 수 있도록 했습니다. |
| **스레드 간 Job Posting 지원** | 스레드 간 통신을 위한 함수를 제공하고, 각 스레드 내부 인터페이스를 통해 해당 메시지를 처리할 수 있도록 구현했습니다. 이를 통해 그룹 간 이동, 공격/피격 등의 로직을 안전하게 전달합니다. |

- 네트워크 라이브러리 내부 그룹 로직
```cpp
void GroupProcess() {

  // 다른 그룹에서 보낸 메시지부터 처리
  while (1) {
    CSerializedBuffer* buffer;
    if (!group->recvGroupQ.Dequeue(buffer))
      break;
    group->OnGroupRecv(buffer);
  }

  group->Update();

  // 그룹 내부 메시지 처리
  while (1) {
    CSerializedBuffer* buffer;
    if (!group->msgQ.Dequeue(buffer))
      break;

    char type;
    long long sessionId;
    *buffer >> type >> sessionId;
    switch (type) {
    case ICNetServerGroup::GROUPJOIN:
      // 3. 그룹 이동이 완료되고, PendingQ에 쌓인 메시지를 먼저 처리
      group->OnGroupJoinned(sessionId);
      ProcessPendingQ(sessionId, group);
      break;
    case ICNetServerGroup::GROUPQUIT:
      // 2. 세션의 메시지를 모두 처리한 이후 최종적으로 그룹을 옮김
      group->OnGroupQuitted(sessionId);
      MoveGroup(sessionId, newGroup);
      break;
    case ICNetServerGroup::ONRECV:
      // 1. 그룹 이동 요청 이후 도착한 메시지는 PendingQ에 쌓아둠
      if (session->isGroupMoving)
        session->PendingQ.Enqueue(buffer);
      else
        group->OnRecv(sessionId, buffer);
      break;
    }
  }
}
```

## 5. 관련 문서
- [Architecture & Protocol (아키텍처 및 통신 규약)](./Architecture.md)
- [네트워크 라이브러리 코어 문서](../../1.%20NetworkLibrary/README.md)
