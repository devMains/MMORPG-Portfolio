# [멀티스레드 기반 로그인 서버] 아키텍처 및 설계 문서

![서버 아키텍쳐](../../Images/TotalServerFlow.png)

## 1. 시스템 아키텍처 (System Architecture)
- 전체 구성도: Client ↔ Server ↔ Database(Redis/MySQL) 흐름 다이어그램
- 스레드 모델:
  - 네트워크 라이브러리의 IOCP Worker 스레드는 OnConnected, OnDisconnected, OnRecv 등의 콜백 함수를 통해 세션의 상태를 전달
  - 콜백 함수 내부에서 즉시 로그인 처리 함수를 호출 -> 네트워크 라이브러리 스레드에서 직접 처리
  - 로그인 처리 함수 내부에는 DB 접근으로 인한 I/O Bound가 자주 발생하기 때문에 네트워크 스레드의 개수를 많이 생성한다. Concurrent 스레드를 조절하여 CPU 사용량 조절.

![서버 아키텍쳐](../../Images/LoginServerFlow.png)

## 2. 통신 프로토콜 (Network Protocol)
- 패킷 구조 (Packet Layout):
  - 공통 헤더
  > Code(1), Length(2), Random Key(1), CheckSum(1)
- 주요 패킷 명세 (API):
  > [REQ] LOGIN : Type(2), AccountNumber(8), SessionKey(64)

  > [RES] LOGIN : Type(2), AccountNumber(8), Status(1), ID(40, wchat), Nickname(40, wchar), GameServerIp(32, wchar), GameServerPort(2), ChatServerIp(32, wchar), ChatServerPort(2)

- 주요 상호작용 시퀀스 (Sequence Diagram):
  - 클라이언트 접속부터 주요 로직(로그인 성공/채널 입장) 완료까지의 흐름도

## 3. 데이터 모델 (Data Model)
- RDBMS 스키마 (MySQL): (로그인/계정 관련 시)
  - 핵심 테이블 구조

  > account (AccountNumber, UserId, UserPassword(hashed), UserNickname)

  > status (AccountNumber, Status)

  > whiteIp (Number, Ip)

- 캐시 및 인메모리 구조 (Redis): 

  - 키(Key) 네이밍 규칙 : {"AccountNumber_C" : sessionKey}, {"AccountNumber_G" : sessionKey}, 채팅 서버/게임 서버와 같이 다른 서버에도 로그인 시 동일 세션 키를 이용할 것이기 때문에 _C, _G 등을 뒤에 붙여서 키로 활용한다.

  - 데이터 만료(TTL) 및 세션 유지 정책 : TTL은 1분이며, 세션 키를 한 번 확인하면 해당 키를 폐기한다. 세션 키를 재활용 하지 않으므로 세션 유지 정책은 지원하지 않는다.

- 내부 메모리 모델 (In-Memory Data Structures):
  - 유저에 대한 정보는 MySQL에 저장되어 있고, 세션 키에 대한 정보는 Redis에 저장하기 때문에 서버 차원에서 들고 있어야 할 정보는 없다.

## 4. 핵심 비즈니스 로직 (Core Business Logic)
- 상태 관리 (State Machine)
> OnConnected -> OnRecv -> LoginProcess -> OnDisconnect

> 일정 시간 메시지 전송이 없다면 서버에서 세션을 끊는다.

- 기타 (Etc.)
> Redis와 MySQL에 대한 연결은 TLS로 관리하여 스레드마다 가지게 한다.
