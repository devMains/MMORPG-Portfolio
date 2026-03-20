# [Network Library] 아키텍처 문서

## 1. 문서 개요 (Document Overview)
   - 대상 독자 : 라이브러리 사용자를 대상으로 어떤 메커니즘으로 작동되는지 설명하는 문서
   - 전제 지식 : C/C++ 기본 지식, 멀티스레드에 대한 이해

## 2. 아키텍처 철학 (Architecture Philosophy)
   - 설계 원칙 : 리소스 재사용과 풀 기반 메모리 관리, 안정성과 정확성
   - 핵심 가치 (성능/확장성/안정성) : 안정성과 정확성(장시간 작동 시 크래시, 메모리 누수 없이 작동)
   - 트레이드오프 결정 : IOCP(Windows 게임 서버를 목표로 설계), 세션의 락 프리 관리(멀티스레드 환경에서 동기화 학습을 위함), 맞춤형 프로토콜 버퍼(자체적인 프로토콜 이용, 상용 엔진의 네트워크 의존 거부)

## 3. 전체 시스템 구조 (System Overview)
   ### 3.1 레이어 아키텍처
      ┌─────────────────────────────┐
      │   Application Layer         │
      ├─────────────────────────────┤
      │   Network Library API       │
      |   - virtual function        |
      |   - SendPacket              |
      |   - Disconnect              | 
      ├─────────────────────────────┤
      │   Core Components           │
      │   - Protocol Parse          │
      │   - Packet Process          │
      │   - Accept / Disconnect     │
      ├─────────────────────────────┤
      │   I/O Layer (IOCP)          │
      └─────────────────────────────┘
   
   ### 3.2 컴포넌트 관계도
      - 주요 컴포넌트 간 의존성
      - 데이터 흐름
   
   ### 3.3 책임 분리
   - Application Layer : 네트워크 라이브러리를 가져다 쓰는 실제 서버의 로직을 담당. 하위 통신이나 스레드 관리의 복잡성을 몰라도 되며, 전달받은 패킷을 처리하고 응답을 전송한다.
   - Network Library Layer : 사용자가 오버라이딩 할 수 있는 이벤트 콜백과 서버 설정, 시작/종료 함수를 제공. 라이브러리 내부의 복잡한 구현을 캡슐화를 통해 숨긴다.
   - Core Components Layer : 데이터를 논리적인 패킷 단위로 만들고 메모리를 관리. 
   - I/O Layer : Windows의 IOCP를 이용하여 Accept 스레드와 다수의 Worker 스레드를 구동하여 이벤트를 감지하고 Core Layer로 알림.

## 4. 스레드 아키텍처 (Thread Architecture)
   ### 4.1 스레드 모델
      Main Thread
         ↓
      Accept Thread -> 새 연결 수락
         ↓
      IOCP Worker Pool (N개)
         ├─ Worker 1 -> Send/Recv 처리, Packet Parse, 이벤트 콜백 호출(OnConnect, OnDisconnect, OnRecv)
         ├─ Worker 2 
         ├─ Worker 3
         └─ Worker N
      -----------------------------------------
      Content Thread
         ↓   SendPacket()
      Session Message Queue Enqueue
         ↓
      IOCP Worker Thread Send
      -----------------------------------------
      Content Thread
         ↓ Disconnect()
      CancelIo, Disconnect Flag On
         ↓
      IOCP Worker Thread Check -> ReleaseSession()

## 5. 네트워크 I/O 아키텍처 (Network I/O Architecture)
   ### 5.1 비동기 I/O 흐름
      Client 연결
         ↓
      Accept 완료 → IOCP에 알림
         ↓
      WSARecv 비동기 호출
         ↓
      데이터 도착 → IOCP Worker 깨어남
         ↓
      Recv 완료 처리 → 패킷 파싱
         ↓
      OnRecv 호출
   
   ### 5.2 Send/Recv 버퍼 관리
   - RingBuffer의 모든 공간 활용 : WSARecv에 모든 버퍼 공간을 등록
   - Overlapped 구조체와 버퍼 연결 : Send 완료 처리 시 직렬화 버퍼 풀 반환

## 6. 세션 관리 (Session Management)
   ### 6.1 세션 생명주기
      생성 → 연결 → 활성 → 종료 대기(종료 플래그) → 삭제(IoCount = 0)
   
   ### 6.2 세션 구조
      struct OVERLAPPEDEX {
         OVERLAPPED overlapped;
         CSerializedBuffer* buffer[MAX_BUFFER_COUNT];
         int bufferCount;
      }
      
      struct Session {
          SOCKET _socket;
          OVERLAPPED recvOverlapped;
          OVERLAPPEDEX sendOverlapped;
          RingBuffer _recvQ;
          LockFreeQueue<Packet*> _sendQ;
          WSABUF[2] _recvBuf;
          unsigned long long _id;
          long _sendFlag;
          long _disconnected;
          long _ioCount;
      };
   
   ### 6.3 세션 풀
   - 세션 배열(락 프리 세션 관리 시 자료구조의 변경 불가) 접근을 위한 인덱스 풀
   
   ### 6.4 동시성 제어
   - 세션별 참조 카운팅 : 사용 중인 세션의 종료를 금지, 종료 시 Recv를 걸지 않음으로 ioCount를 0으로 유도하는 방식 이용
   - Recv/Send 동시 1회 제한 : Overlapped 구조체 관리 불편, Recv의 메시지 직렬화 보장

## 7. 패킷 처리 아키텍처 (Packet Processing)
   ### 7.1 패킷 구조
      [Header: 5bytes] [Body: N bytes]
      ├─ Code: 1byte
      ├─ Size: 2bytes
      ├─ Random Key : 1byte
      ├─ CheckSum : 1byte
      └─ Payload: N bytes
   
   ### 7.2 패킷 처리 파이프라인
   - 수신 → 링버퍼 저장 → 패킷 경계 감지 → 역직렬화 및 복호화(Random Key + Constant Key) → 처리 → 응답 직렬화/OnRecv 호출
   
## 8. 메모리 관리 아키텍처 (Memory Management)
   **[메모리 풀 문서 바로가기](1.%20NetworkLibrary/Components/TLSMemoryPool.md)**

## 9. 직렬화 아키텍처 (Serialization Architecture) 
   **[직렬화 버퍼 문서 바로가기](1.%20NetworkLibrary/Components/SerializedBuffer.md)**

## 10. 에러 처리 아키텍처 (Error Handling)
   ### 10.1 에러 계층
   - 초기 설정 에러 (초기 네트워크 설정 에러)
   - 네트워크 에러 (연결 끊김)
   - 프로토콜 에러 (잘못된 패킷)
   - 논리 에러 (비즈니스 로직)
   
   ### 10.2 에러 복구 전략
   - 초기 설정 에러 -> 서버 종료 + 로그
   - 연결 에러 -> 세션 정리
   - 패킷 에러 -> 세션 정리 + 로그
   - 크리티컬 에러 -> 덤프 생성 + 서버 종료
   
   ### 10.3 로깅
   - OnError 콜백 함수를 통한 라이브러리 사용자 측 전달

## 11. 성능 최적화 아키텍처 (Performance Optimization)
   ### 11.1 Lock-Free 설계
   - 메모리 풀: TLS 기반 무잠금
   
   ### 11.2 Zero-Copy 최적화
   - WSASend 콜타임이 긴 경우 사용(On/Off 옵션 제공)

   ### 11.3 컨텐츠 로직 영향 최소화
   - SendPacket 함수 내부 세션의 _sendQ에 넣고 반환
   - WSASend의 호출은 IOCP Worker Thread에서 직접 호출
   
   ### 11.4 시스템 콜 최소화
   - 배치 Send/Recv
   - Scatter-Gather I/O

## 12. 보안 아키텍처 (Security Architecture)
   ### 12.1 입력 검증
   - 패킷 크기 제한 : 직렬화 버퍼 크기 제한
   - 유효성 검사 : 프로토콜 유효성 검사
   
   ### 12.2 DoS 방어
   - Rate Limiting : 한 번에 RingBuffer의 크기만큼만 받도록 하여 처리의 상한선 설정
   - 연결 수 제한 : 지정 연결 수 이상 연결을 받지 않음
   
   ### 12.3 메모리 보안
   - 버퍼 오버플로우 방지 : 버퍼 크기 체크 메서드를 통한 사용 전 검사
   - Use-After-Free 방지 : 메시지 참조 카운트를 이용한 메모리 수명 관리

## 13. 모니터링 아키텍처 (Monitoring)
   **[모니터링 문서 바로가기](1.%20NetworkLibrary/Components/Monitoring.md)**

## 14. 설계 패턴 (Design Patterns)
   ### 14.1 사용된 패턴
   - Singleton: ObjectPoolTLS
   - Template Method : Callback Functions
   - Actor : IOCP Worker Thread
   
   ### 14.2 패턴 선택 이유
   - Singleton : 특정 자료형의 풀을 전역으로 하나씩 관리하기 위함
   - Template Method : 네트워크 라이브러리에서 세션에 대한 이벤트를 알려주어야 함
   - Actor : 동일한 코드를 가진 IOCP Worker Thread에서 여러 세션을 처리하기 위해 세션에 대한 정보와 메시지를 IOCP Queue에 삽입한다

## 15. 아키텍처 진화 (Architecture Evolution)
   ### 15.1 초기 설계
   - 콘텐츠와 계층을 분리하여 콘텐츠에서 요청할 수 있는 SendPacket과 Disconnect 함수 제공
   - Send/Recv RingBuffer를 활용한 기본적인 IOCP 라이브러리, 세션에 대한 동기화 객체를 이용하여 동기화를 진행
   - IoCount 기반 세션의 생명 주기 관리
   - 문제점 : 세션 삭제 시 세션과 같이 있는 동기화 객체도 삭제가 됨. 동기화를 걸고 동기화 객체를 삭제하는 것은 말이 되지 않음.
   
   ### 15.2 개선
   - 동기화 객체의 삭제가 문제가 된다면 삭제를 하지 않고, 세션을 재활용하여 해결
   - 세션 ID 상단의 16비트에 세션 배열의 인덱스를 넣어 탐색 없이 즉시 인덱스를 확인
   - 멀티스레드 환경의 동기화 학습을 위해 세션 관리를 락 프리 구조로 변경
     -> 현재 구조에서는 Send, Recv의 완료 통지를 1번씩만 제한하여 경합이 많이 발생하지 않는다. 따라서 실제로는 동기화 객체를 사용하여 관리하는 것이 더 빠를 것이다.
     -> 세션 배열 구조 이용, 배열 인덱스 기반 세션 재활용, Send RingBuffer를 CSerializedBuffer 타입의 락 프리 큐로 변경, Interlocked 기반 세션 상태 확인 및 적용
   - 개선 효과 : 세션의 삭제 시 생기는 동기화 문제를 해결
   
   ### 15.3 현재
   - 간단한 암호화 제공, Send 시 Zero-Copy 옵션 제공
   
   ### 15.4 향후 계획 (v2.0)
   - 계획 중인 개선 사항
