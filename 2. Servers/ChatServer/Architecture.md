# [멀티스레드 기반 채팅 서버] 아키텍처 및 설계 문서

![서버 아키텍쳐](../../Images/TotalServerFlow.png)

## 1. 시스템 아키텍처 (System Architecture)
- 전체 구성도: Client ↔ Server ↔ Database(Redis/MySQL) 흐름 다이어그램
- 스레드 모델: 
  - 네트워크 라이브러리의 IOCP Worker 스레드는 OnConnected, OnDisconnected, OnRecv 등의 콜백 함수를 통해 세션의 상태를 전달
  - 콜백 함수 내부에서 세션의 상태를 기반으로 자원에 접근하거나 콘텐츠 스레드 큐에 삽입하여 Job을 넘김
  - 콘텐츠 스레드는 Job을 기반으로 메시지 처리 진행

![채팅 서버 아키텍쳐](../../Images/ChatServerFlow.png)

## 2. 통신 프로토콜 (Network Protocol)
- 패킷 구조 (Packet Layout):
  - 공통 헤더
  > Code(1), Length(2), Random Key(1), CheckSum(1)
- 주요 패킷 명세 (API):
  > [REQ] LOGIN : Type(2), AccountNumber(8), Id(40, wchar), Nickname(40, wchar), SessionKey(64)

  > [RES] LOGIN : Type(2), Status(1), AccountNumber(8)

  > [REQ] MOVE SECTOR : Type(2), AccountNumber(8), X(2), Y(2)

  > [RES] MOVE SECTOR : Type(2), AccountNumber(8), X(2), Y(2)

  > [REQ] CHAT : Type(2), AccountNumber(8), MessageLength(2), Message(MessageLength/2, wchar)

  > [REQ] CHAT : Type(2), AccountNumber(8), Id(40, wchar), Nickname(40, wchar), MessageLength(2), Message(MessageLength/2, wchar)

- 주요 상호작용 시퀀스 :
  - 채팅 서버에 클라이언트가 접속하게 되면 OnConnected 함수가 호출되게 되고, 이를 기반으로 채팅 서버 Player 객체를 미리 준비한다.
  - 이후 클라이언트는 로그인 요청 패킷을 전송하게 되고, 서버는 세션 키를 받아 Redis에서 확인을 한다.
  - 로그인에 성공하게 되면 로그인 성공 패킷을 클라이언트에게 전송하게 된다.
  - 클라이언트는 이동하고 싶은 섹터에 대한 이동 요청 패킷을 전송하고, 서버는 이동 이후 완료 패킷을 전송한다.
  - 클라이언트는 채팅 메시지 요청 패킷을 전송하고, 서버는 주위 섹터([1, 1] 섹터라면 [0, 0]~[2, 2]의 9개 섹터)에 해당 메시지를 전파한다.

## 3. 데이터 모델 (Data Model)
- RDBMS 스키마 (MySQL): (로그인/계정 관련 시)
  - 핵심 테이블 구조

  > account (AccountNumber, UserId, UserPassword(hashed), UserNickname)

  > status (AccountNumber, Status)

  > whiteIp (Number, Ip)

- 캐시 및 인메모리 구조 (Redis): 
  - 키(Key) 네이밍 규칙 : {"AccountNumber_C" : sessionKey}, 채팅 서버/게임 서버와 같이 다른 서버에도 로그인 시 동일 세션 키를 이용할 것이기 때문에 _C, _G 등을 뒤에 붙여서 키로 활용한다.
  - 데이터 만료(TTL) 및 세션 유지 정책 : TTL은 1분이며, 세션 키를 한 번 확인하면 해당 키를 폐기한다. 세션 키를 재활용 하지 않으므로 세션 유지 정책은 지원하지 않는다.


- 내부 메모리 모델 (In-Memory Data Structures):
  - std::unordered_map<long long, Player*> playerMap[N] + SRWLOCK[N]
    - 네트워크 라이브러리에서 제공해주는 SessionId를 Key로 Player 객체를 가지고 있다.
    - SessionId에 % 연산을 하여 경합을 분산한다.
  - std::unordered_map<long long, long long> accountNumberMap + SRWLOCK
    - 중복 로그인을 막기 위해 AccountNumber와 SessionId를 저장하고 있다.
  - std::vector<Player*> sectorList[SECTOR_Y][SECTOR_X] + SRWLOCK
    - 섹터 내부의 플레이어가 들어있다.
    - 주위 섹터의 플레이어에게 메시지를 전파할 때 이 리스트를 순회한다.
  - CObjectPool<PLAYER> playerPool
    - 플레이어 객체를 재활용하기 위한 풀이다.

## 4. 핵심 비즈니스 로직 (Core Business Logic)
- 상태 관리 (State Machine)
  > 서버 연결 : OnConnected -> (Session)

  > 로그인 요청 : OnRecv -> Login Packet -> Session Key Check -> (Authed - Player)

  > 접속 종료 : OnDisconnected -> (Dead Session)

- 동시성 제어 (Concurrency Control):
  - 특정 자원(예: 같은 방, 같은 계정)에 접근할 때의 Lock 관리 또는 락프리 처리 방식
     - OnRecv를 통해 받는 메시지는 플레이어 별 MessageQ에 넣는다. MessageQ는 락 프리 큐를 이용한다. 
     - 플레이어의 메시지 처리는 한 번에 하나의 스레드만 처리하도록 한다. 세션 별 ProcessingFlag에 CAS 연산을 이용하여 판단한다.
     - 메시지 전송 시 주변 섹터의 플레이어의 이동을 막기 위해 주변 섹터 모두 동기화를 걸고 메시지를 전송한다. 이는 순간의 이동으로 인해 메시지가 유실되거나 2번 이상 오는 상황이 발생할 수 있다.
     - 이 외의 공유 자원에 대해서는 SRWLOCK을 이용하여 동기화를 걸고 사용한다.
  - 외부 I/O(DB 쿼리) 대기 중의 스레드 블로킹 방지 전략
     - Redis 접근 시 I/O Bound로 인해 스레드 블로킹 시 다른 스레드가 작업을 할 수 있도록 스레드의 개수를 늘린다. Concurrent 스레드의 개수는 그대로 유지한다.
