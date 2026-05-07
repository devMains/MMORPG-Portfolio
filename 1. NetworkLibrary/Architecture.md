# [Network Library] 아키텍처 문서

## 1. 문서 개요 (Document Overview)

| 항목 | 상세 내용 |
| :--- | :--- |
| **목적** | 본 네트워크 라이브러리를 사용하는 개발자가 내부 동작 메커니즘과 설계 의도를 파악할 수 있도록 작성된 아키텍처 가이드 |
| **핵심 아키텍처** | • Windows IOCP 기반 비동기 I/O<br>• Lock-Free 기반 세션 및 큐 관리<br>• Layered Architecture (관심사 분리) |
| **대상 독자** | C/C++ 및 멀티스레드 동기화 기법에 대한 이해가 있는 서버 개발자 |

## 2. 아키텍처 철학 (Architecture Philosophy)

*   **설계 원칙:** 
    *   **리소스 재사용:** 잦은 동적 할당을 방지하기 위해 객체 풀 기반의 메모리 관리를 강제합니다.
    *   **안정성과 정확성 우선:** 고성능보다 메모리 안전성(Use-after-free 방지)과 스레드 안전성을 우선하여 설계했습니다.
*   **핵심 가치:** 장기간(수십 시간 이상) 구동되는 라이브 서비스 환경에서 크래시나 메모리 누수 없이 동작하는 견고함.
*   **트레이드오프:**
    *   **Windows 특화:** 멀티 플랫폼 지원을 포기하고 Windows 전용 고성능 API인 IOCP에 집중했습니다.
    *   **Lock-Free 세션 관리:** 동기화 객체 기반 관리가 더 직관적일 수 있으나, 극단적인 병렬 환경에서의 동기화 기술 학습을 위해 락프리 구조를 채택했습니다.
    *   **맞춤형 프로토콜 버퍼:** 범용 직렬화 라이브러리(Protobuf 등)나 상용 엔진에 의존하지 않고, 자체적인 패킷 조립 및 파싱 버퍼를 구현하여 종속성을 제거했습니다.

## 3. 전체 시스템 구조 (System Overview)

### 3.1 레이어 아키텍처
네트워크 코어와 비즈니스 로직을 명확히 분리하여, 사용자가 하위 네트워크 복잡성을 몰라도 서버를 개발할 수 있도록 설계했습니다.

```text
┌─────────────────────────────┐
│   Application Layer         │ ← 비즈니스 로직 (사용자 구현 영역)
├─────────────────────────────┤
│   Network Library API       │ ← 가상 함수(Callbacks), SendPacket(), Disconnect()
├─────────────────────────────┤
│   Core Components           │ ← 패킷 파싱, 직렬화 버퍼, 세션 관리
├─────────────────────────────┤
│   I/O Layer (IOCP)          │ ← OS 커널 상호작용, 소켓 이벤트 멀티플렉싱
└─────────────────────────────┘
```

### 3.2 책임 분리
*   **Application Layer:** 라이브러리를 상속받아 사용하는 실제 서버(콘텐츠) 계층입니다. 스레드나 소켓을 몰라도 전달받은 패킷만 처리하면 됩니다.
*   **Network Library API:** 비즈니스 로직과 코어를 연결하는 경계입니다. 사용자가 오버라이딩할 수 있는 이벤트 콜백(`OnConnect`, `OnRecv` 등)과 서버 설정/제어 API를 제공합니다.
*   **Core Components:** 바이트 스트림을 논리적인 패킷 단위로 조립/분해하며, 세션과 메모리를 관리합니다.
*   **I/O Layer:** Windows IOCP를 활용해 Accept 스레드와 다수의 Worker 스레드를 구동하며, 네트워크 이벤트를 감지해 Core Layer로 전달합니다.

## 4. 스레드 아키텍처 (Thread Architecture)

### 스레드 모델 및 데이터 흐름

*   **수신 흐름 (Network → Logic):**
    *   `Accept Thread`: 새 클라이언트 연결 수락
    *   `IOCP Worker Thread (N개)`: 소켓 I/O 완료 통지 대기 → `WSARecv` 처리 → `Packet Parse` → 라이브러리 API 콜백(`OnConnect`, `OnDisconnect`, `OnRecv`) 호출
*   **송신 흐름 (Logic → Network):**
    *   `Content Thread`에서 `SendPacket()` 호출
    *   즉시 전송하지 않고 해당 세션의 `LockFreeQueue(SendQ)`에 패킷 삽입 후 반환 (콘텐츠 스레드 블로킹 방지)
    *   `IOCP Worker Thread`가 큐에서 데이터를 꺼내어 비동기 전송(`WSASend`) 수행
*   **종료 흐름:**
    *   `Content Thread`에서 `Disconnect()` 호출 시 즉시 소켓을 닫지 않고 `CancelIo` 호출 및 `Disconnect Flag` 활성화
    *   `IOCP Worker Thread`에서 남은 비동기 작업(IoCount)을 확인한 뒤 안전하게 세션을 회수(`ReleaseSession`)

## 5. 네트워크 I/O 아키텍처 (Network I/O Architecture)

### 5.1 비동기 I/O 파이프라인
```text
Client 연결 → Accept 스레드 처리 → IOCP 포트에 핸들 등록 
→ WSARecv 대기 → 데이터 수신 완료 시 IOCP Worker 스레드 기상
→ 수신 버퍼 처리 및 패킷 경계 파싱 → 완전한 패킷 완성 시 OnRecv 호출
```

### 5.2 버퍼 관리 최적화
*   **링버퍼 극대화:** `WSARecv` 호출 시 `RingBuffer`의 남은 물리적 빈 공간 2곳(끝부분과 앞부분)을 `WSABUF` 배열에 모두 등록하여 한 번의 시스템 콜로 최대한 많은 데이터를 수신.
*   **Overlapped 연결:** 송신 완료 통지 시 `Overlapped` 구조체에 묶여있던 직렬화 버퍼들의 참조 카운트를 감소시키고 풀로 반환합니다.

## 6. 세션 관리 (Session Management)

### 6.1 세션 생명주기
`생성 → 연결 완료(OnConnect) → 활성 상태(송수신) → 종료 대기(종료 플래그 On) → 삭제(IoCount == 0 달성 시 풀 반환)`

### 6.2 세션 메모리 구조
```cpp
// 송신 비동기 작업을 위한 커스텀 구조체
struct OVERLAPPEDEX {
    OVERLAPPED overlapped;
    CSerializedBuffer* buffer[MAX_BUFFER_COUNT];
    int bufferCount;
}

// 클라이언트 연결 1개를 대변하는 세션 객체
struct Session {
    SOCKET _socket;
    OVERLAPPED recvOverlapped;
    OVERLAPPEDEX sendOverlapped;
    RingBuffer _recvQ;                 // 수신 스트림 조립용 버퍼
    LockFreeQueue<Packet*> _sendQ;     // 논리적 송신 패킷 대기열
    WSABUF _recvBuf[2];
    unsigned long long _id;            // 고유 식별자 (하위 16비트는 배열 인덱스)
    long _sendFlag;                    // 중복 WSASend 방지 플래그
    long _disconnected;                // 종료 상태 플래그
    long _ioCount;                     // 진행 중인 비동기 I/O 작업 수

    long isGroupMoving = 0;            // 그룹 이동 플래그
    ICNetServerGroup* group = 0;       // 그룹 객체 
	 long groupVersion = 0;             // 그룹 버전
	 SRWLOCK groupLock;                 // 그룹 관련 로직 실행 시 동기화 객체
	 CQueueLockFree<CSerializedBuffer*>pendingQ; // 그룹 이동 요청 이후 메시지 저장
};
```

### 6.3 동시성 제어 (Concurrency Control)
*   **참조 카운팅 (IoCount):** 세션 해제 시점의 사용을 막기 위해 `IoCount`를 운용합니다. 진행 중인 비동기 I/O가 0이 될 때만 세션을 풀에 반환합니다.
*   **송수신 동시 1회 제한:** 세션당 동시에 걸려있는 `WSASend`와 `WSARecv`를 각각 1회로 제한하여 패킷 순서(Serialization)를 보장하고 Overlapped 구조체 관리의 복잡성을 낮췄습니다.

```cpp

// 세션 접근 시
bool CNetServerGroup::SendPacket(long long sessionId, CSerializedBuffer* buffer) {
   Session* session = _sessionMap[GetIdx(sessionId)];

   // 참조 카운트 증가
   long d = InterlockedIncrement((long*)&session->ioCount);
   // 다른 스레드가 세션 종료 중이라면 반환
   if ((d & _releaseMask) == _releaseMask) {
      return false;
   }

   // 세션이 변경되었으면 반환
   if (session->id != sessionId) {
      // ioCount 감소 및 정리
      ReleaseSession(session);
      return false;
   }

   // 세션 작업...

   // ioCount 감소 및 정리
   ReleaseSession(session);
}


// 종료 및 릴리즈 작업
void CNetServerGroup::ReleaseSession(Session* session) {
   // 줄인 ioCount가 0인지 확인
   if (!InterlockedDecrement((long*)&session->ioCount)) {
      // 0으로 줄였다면 릴리즈 마스크 값으로 바꿈
      if (!InterlockedCompareExchange((long*)&session->ioCount, _releaseMask, 0)) {
         // 연결 끊김 플래그를 세움
         if (!InterlockedExchange((long*)&session->disconnected, 1)) {
            closesocket(session->s);

            // 그룹에 존재한다면 그룹 퇴장 메시지 전달
            if (session->group != 0) {
               session->group->msgQ.Enqueue(_quitMsg);
            }

            // 연결 끊김 콜백 함수 호출
            OnClientDisconnected(session->id);
            // 세션 반환
            _emptySession.Push(GetIdx(session->id));
         }
      }
   }
}
```

```cpp
unsigned __stdcall CNetServerGroup::WorkerThread(void* param) {
   while (1) {
      DWORD cbTransferred = 0;
      ULONG_PTR completionKey = 0;
      LPOVERLAPPED overlapped = 0;

      int ret = GetQueuedCompletionStatus(_completionPort, &cbTransferred, &completionKey, &overlapped, INFINITE);

      Session* session = (Session*)completionKey;

      if (&session->recvOverlapped == overlapped) {
         // 받은 데이터를 패킷으로 완성하여 OnRecv로 알림
         RecvProcess(session);

         // Recv 완료 통지 이후 다시 WSARecv를 걸어서 1회 제한
         InterlockedIncrement((long*)&session->ioCount);
         if (!RecvPost(session)) {
            InterlockedDecrement((long*)&session->ioCount);
         }
      }
      else if (&session->sendOverlapped.overlapped == overlapped) {
         // 이전 WSASend에서 사용한 직렬화 버퍼를 반환
         for (int i = 0; i < session->sendOverlapped.bufferCnt; i++) {
            CSerializedBuffer::Free(session->sendOverlapped.buffer[i]);
         }
         session->sendOverlapped.bufferCnt = 0;

         // SendPost 내부에서 session의 send flag 검사로 WSASend 1회 제한
         SendPost(session);
      }

      // 완료 통지에 대한 ioCount를 줄이고 정리
      if (session != 0)
         ReleaseSession(session);

      // 그룹 관련 작업 실시
      GroupProcess();
   }
   return 0;
}
```

## 7. 패킷 처리 아키텍처 (Packet Processing)

### 7.1 패킷 구조 (Header: 5 bytes)
```text
[ Code(1) | Size(2) | RandomKey(1) | CheckSum(1) ] [ Payload(N) ]
```

### 7.2 파이프라인
`수신 → 링버퍼(RingBuffer) 적재 → Size 확인 후 패킷 분리 → 역직렬화 및 복호화(Random/Constant Key) → OnRecv 콜백 전달`

## 8. 메모리 관리 아키텍처 (Memory Management)
🔗 **[TLS 메모리 풀 문서 바로가기](./Components/TLSMemoryPool.md)**

## 9. 직렬화 아키텍처 (Serialization Architecture) 
🔗 **[직렬화 버퍼 문서 바로가기](./Components/SerializedBuffer.md)**

## 10. 에러 처리 아키텍처 (Error Handling)

### 10.1 에러 계층 및 복구 전략
*   **초기 설정 에러:** 네트워크 바인딩 실패 등 → 로그 기록 후 서버 프로세스 종료
*   **네트워크 에러:** 연결 끊김, 소켓 에러 → 해당 세션만 종료 프로세스 진행
*   **프로토콜 에러:** 패킷 변조, CheckSum 오류 → 불량 클라이언트로 간주하여 세션 강제 종료 및 로그 기록
*   **크리티컬 에러:** 라이브러리 내부 논리 오류 → 덤프(Dump) 파일 생성 후 안전하게 서버 종료

### 10.2 로깅
모든 예외 상황은 `OnError` 가상 함수 콜백을 통해 Application Layer로 전달되어 일관된 로그 파일에 기록됩니다.

## 11. 성능 최적화 아키텍처 (Performance Optimization)

*   **비동기 I/O 캡슐화:** 콘텐츠 스레드에서 송신 시 직접 I/O를 발생시키지 않고 큐에 적재하여, 게임 로직의 프레임 저하를 방지.
*   **시스템 콜 최소화:** 여러 패킷을 모아서 한 번에 커널로 넘기는 **배치(Batch) 전송**과, 메모리 복사를 제거한 **Scatter-Gather I/O (`WSABUF`)** 적용.
*   **Zero-Copy 옵션:** `WSASend` 복사 오버헤드가 문제가 되는 특정 시나리오를 위해 Zero-Copy 활성화 옵션 제공.

## 12. 보안 아키텍처 (Security Architecture)

*   **입력 검증:** 수신된 패킷의 Size 필드를 검증하여 직렬화 버퍼의 최대 크기 초과 여부 확인 및 프로토콜 구조 유효성 검사.
*   **DoS 방어:** 한 번의 `WSARecv`에서 최대 처리 가능 패킷 개수를 제한하여 악의적인 대용량 트래픽 공격 완화.
*   **메모리 보안 (Use-After-Free 방지):** 락프리 세션 배열과 참조 카운트 기반 수명 관리를 통해, 이미 해제된 세션이나 패킷에 잘못 접근하여 서버가 다운되는 현상 원천 차단.
*   **메모리 누수 방어:** 의도적으로 recv를 하지 않는 세션에 대해 `WSASend`의 완료 통지가 오지 않아서 `SendQ(LockFreeQueue)`에 직렬화 버퍼가 반환되지 않고 쌓이는 현상이 발생할 수 있음. `SendQ(LockFreeQueue)`에 최대 버퍼 개수 제한을 두어 의도적으로 recv를 하지 않는 세션의 연결을 중단함. 또한 컨텐츠의 데이터 생산이 너무 많은 경우에도 자연스럽게 끊김을 유도하여 적절한 서버 상태를 유도할 수 있음.

## 13. 모니터링 아키텍처 (Monitoring)
🔗 **[모니터링 시스템 문서 바로가기](./Monitoring.md)**

## 14. 설계 패턴 (Design Patterns)

*   **Singleton (ObjectPoolTLS):** 세션, 버퍼, 노드 등 자주 생성/소멸되는 자원을 전역에서 유일한 풀 인스턴스로 중앙 관리하기 위해 도입.
*   **Template Method (Callback Functions):** 핵심 I/O 흐름의 뼈대는 라이브러리가 통제하고, `OnConnect`, `OnRecv` 등의 구체적인 비즈니스 행위만 하위 클래스에서 정의하도록 유도.
*   **Actor Model 패턴 적용 (IOCP Worker):** 세션 고유의 큐에 메시지를 던져 넣고 단일 스레드가 이를 순차적으로 꺼내 처리하는(Actor) 유사한 구조를 차용.

## 15. 아키텍처 진화 (Architecture Evolution)

### 15.1 초기 설계의 한계 (v1.0)
*   초기에는 동기화 객체(Mutex/SRWLock)를 사용해 세션 상태를 보호했습니다. 
*   **문제점:** 연결이 종료되어 세션 객체를 메모리에서 삭제해야 할 때, 해당 세션의 동기화 객체를 락으로 점유한 상태에서 객체 자체를 `delete` 해야 하는 논리적/구조적 모순이 발생했습니다.

### 15.2 개선된 설계 (v2.0)
*   **세션 재활용 도입:** 동기화 객체 삭제 문제를 해결하기 위해, 세션 객체를 동적 해제하지 않고 **고정된 세션 배열과 풀을 통해 재활용**하는 구조로 전면 개편했습니다.
*   **Lock-Free 세션 관리:** 세션 ID의 하위 48비트는 시퀀스로, 상위 16비트는 배열 인덱스로 활용하여 **탐색 비용을 제거**했습니다. 상태 관리는 `Interlocked` 함수 기반의 원자적 연산으로 대체했습니다.
*   **효과:** 세션 삭제 시 발생하던 동시성 충돌 문제가 완전히 해결되었으며, 할당 오버헤드도 사라졌습니다. (단, 현재 설계상 송수신 I/O가 각각 1회로 제한되어 있어 실제 락 경합이 적으므로, 구현 난이도 대비 성능 향상보다는 **동시성 제어의 통제**에 더 큰 의의를 둡니다.)

### 15.3 그룹 설계 (v3.0)
*   **그룹 컨텍스트 추가:** 인스턴스 던전, PVP와 같은 분리된 컨텐츠를 가동할 수 있는 그룹 컨텍스트를 추가하였습니다.
*   **동일 컨텐츠 이용 고려:** 그룹 이동 요청 이후 발생한 메시지의 경우 다음 그룹의 입장 시 처리할 수 있도록 하였습니다. 동일 컨텐츠의 경우 이전 그룹의 메시지가 필요한 경우가 존재하였습니다.
