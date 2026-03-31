# 🌐 [Network Library] - 자체 제작 C++ 네트워크 라이브러리

## 1. 개요 (Overview)

| 항목 | 상세 내용 |
| :--- | :--- |
| **요약** | 대규모 동시 접속(MMORPG 등) 처리에 최적화된 **Windows IOCP 기반 C++ 네트워크 코어 라이브러리** |
| **목적** | 상용 엔진이나 외부 라이브러리(Boost.Asio 등)에 의존하지 않고, 비동기 I/O와 동시성 제어의 원리를 100% 직접 제어 |
| **주요 성과** | **10K 동시 접속**, **초당 600만(6M) 패킷 처리**, **평균 지연 85ms** 달성 (상세 지표 하단 참조) |

![주요 성능 지표](../Images/Charts.png)

## 2. 목차 (Table of Contents)
- [주요 특징](#3-주요-특징-key-features)
- [아키텍처](#4-아키텍처-architecture)
- [핵심 컴포넌트](#5-핵심-컴포넌트-core-components)
- [API 사용법](#6-api-사용법-usage)
- [성능](#7-성능-performance)
- [기술 스택](#8-기술-스택-tech-stack)
- [사용 사례](#9-사용-사례-use-cases)
- [기술적 의사 결정](#10-기술적-의사결정-technical-decisions)
- [제한 사항](#11-제한사항-및-알려진-이슈-limitations)
- [관련 문서](#12-관련-문서-references)

## 3. 주요 특징 (Key Features)
*   **비동기 I/O 모델:** 완료 통지(Completion Port) 기반의 비동기 네트워크 송수신 구조. 송신 병목 해소를 위한 **Zero-Copy (Direct I/O)** 활성화 옵션 제공.
*   **스레드 및 동시성 제어:** 멀티스레드 환경의 안전한 데이터 처리를 보장하며, 사용자(Application Layer)가 총 워커 스레드 수와 동시 실행(Concurrent) 스레드 수를 유연하게 설정 가능.
*   **패킷 직렬화 보장:** 세션별로 동시에 걸리는 비동기 수신(Recv) 작업을 단 1회로 엄격히 제한하여 패킷 순서의 정합성(Serialization) 보장.
*   **안전한 세션 관리:** 비동기 I/O 작업의 참조 카운트(IoCount)를 기반으로 세션의 생명주기를 완벽히 통제하여 Use-After-Free 크래시 방지.
*   **콜백 기반 에러/이벤트 처리:** 템플릿 메서드 패턴을 활용해 라이브러리 사용자(콘텐츠 개발자)에게 접속/종료/패킷/에러 이벤트를 가상 함수 형태로 깔끔하게 전달.

## 4. 아키텍처 (Architecture)

### 4.1 레이어 계층 구조
```mermaid
graph TD
    A[Application Layer] -.->|상속 및 가상함수 구현| B(Network Library API)
    A -.->|SendPacket / Disconnect 호출| B
    B --- C(Core Components)
    C ---|Protocol Parse / Session Mngr / Buffer| D(I/O Layer - IOCP)
    
    style A fill:#e3f2fd,stroke:#333,stroke-width:2px
    style B fill:#fff3e0,stroke:#333,stroke-width:2px
    style C fill:#e8f5e9,stroke:#333,stroke-width:2px
    style D fill:#fce4ec,stroke:#333,stroke-width:2px
```

### 4.2 스레드 모델
*   **스레드 구성:** `AcceptThread (1)` + `TPS/MonitoringThread (1)` + `IOCP Worker Thread (N)`
*   **동작 파이프라인:** 
    *   **AcceptThread:** 루프를 돌며 클라이언트 접속을 수락하고 `OnConnectionRequest`(IP 필터링 등)를 거쳐 세션을 생성한 뒤 `OnClientConnected` 콜백을 호출합니다.
    *   **IOCP Worker Thread:** 커널로부터 `Overlapped` 객체와 전송 바이트 수를 통지받습니다. 
        *   *수신(Recv) 시:* `RingBuffer` 포인터를 갱신하고 완성된 메시지를 조립하여 `OnRecv` 콜백을 호출한 뒤, 다음 `WSARecv`를 예약합니다.
        *   *송신(Send) 시:* 전송 완료된 직렬화 버퍼들을 메모리 풀에 반환(`Free`)하고, 송신 큐(`_sendQ`)에 남은 패킷이 있다면 `WSASend`를 다시 예약합니다.
        *   *세션 정리:* 모든 I/O 작업이 완료되어 `IoCount`가 0으로 떨어지면 안전하게 세션을 회수합니다.

## 5. 핵심 컴포넌트 (Core Components)

### 5.1 직렬화 버퍼 (Serialized Buffer)
*   **설명:** 프로토콜 명세에 맞춰 바이트 단위로 메시지를 조립하고 파싱하는 커스텀 버퍼입니다. 연산자 오버로딩(`<<`, `>>`)을 지원하여 콘텐츠 로직 작성이 매우 간편합니다.
*   **[상세 문서](./Components/SerializedBuffer.md)**

### 5.2 링 버퍼 (Ring Buffer)
*   **설명:** TCP 스트림 특성상 쪼개지거나 합쳐져서 오는 바이트 스트림을 온전한 하나의 메시지 단위로 조립하기 위해 사용하는 환형 구조의 수신 전용 큐입니다.
*   **[상세 문서](./Components/RingBuffer.md)**

### 5.3 락 프리 큐 (Lock-Free Queue)
*   **설명:** 동기화 객체(Mutex/SRWLock)의 대기 시간 없이, 64비트 CAS 연산과 포인터 마스킹을 활용해 패킷 버퍼를 안전하게 삽입/추출하는 송신 대기 큐입니다.
*   **[상세 문서](./Components/LockFreeQueue.md)**

### 5.4 TLS 메모리 풀 (Thread-Local Memory Pool)
*   **설명:** 스레드 고유의 로컬 스토리지(TLS)를 활용하여 글로벌 힙(Heap)의 락 경합 없이 직렬화 버퍼나 큐 노드를 O(1)에 할당하고 반환하는 재사용 객체 풀입니다.
*   **[상세 문서](./Components/TLSMemoryPool.md)**

## 6. API 사용법 (Usage)

### 6.1 기본 서버 실행 및 이벤트 처리
```cpp
// 1. 라이브러리 코어를 상속받아 콜백 함수를 오버라이딩합니다.
class MyLanServer : public CNetServer {
    bool OnConnectionRequest(long ipAddress, short port) override {
        // IP 필터링 (White List 등)
        return true; 
    }

    void OnClientConnected(long long sessionId) override {
        // 새 세션과 콘텐츠 플레이어 객체 매핑
        PLAYER* newPlayer = new Player();
        _playerMap.insert({sessionId, newPlayer});
    }

    void OnClientDisconnected(long long sessionId) override {
        // 플레이어 객체 정리
        auto it = _playerMap.find(sessionId);
        if (it != _playerMap.end()) {
            delete it->second;
            _playerMap.erase(sessionId);
        }
    }

    void OnRecv(long long sessionId, CSerializedBuffer* buffer) override {
        // 패킷 파싱 및 처리
        auto it = _playerMap.find(sessionId);
        if (it != _playerMap.end()) {
            char packetType;
            *buffer >> packetType;

            switch(packetType) {
                // 비즈니스 로직 분기
            }
        }
        // 사용이 끝난 버퍼는 풀로 반환
        CSerializedBuffer::Free(buffer);
    }

    void OnError(ServerError err, int errCode, LPCWSTR errorText) override {
        // 로깅 등 에러 처리
        Log(errCode, errorText);
    }
};

// 2. 서버 설정 후 Start 호출
int main() {
    MyLanServer server;
    // IP, Port, Total Worker, Concurrent Worker, Max Session, Encrypt, Nagle, ZeroCopy
    server.Start(L"127.0.0.1", 7777, 10, 8, 10000, false, false, true);
    
    // 메인 스레드 대기 루프
    while(true) { Sleep(1000); }
}
```

## 7. 성능 (Performance)
*   **벤치마크 환경:** Intel Xeon E5-2680 v4 (14C/28T), 16GB RAM, Windows Server 2019
*   **벤치마크 결과 상세:** [성능 분석 문서](./Performance.md) 참조

## 8. 기술 스택 (Tech Stack)
*   **언어:** C++ 17
*   **플랫폼:** Windows OS (Windows Sockets 2 - IOCP)
*   **개발 환경:** Microsoft Visual Studio 2022

## 9. 사용 사례 (Use Cases)
이 네트워크 라이브러리를 기반으로 제작한 애플리케이션 계층 서버입니다.
*   [로그인 서버 (인증, 중복 로그인 방지)](../2.%20Servers/LoginServer/README.md)
*   [채팅 서버 (다채널 브로드캐스팅, 공간 분할)](../2.%20Servers/ChatServer/README.md)

## 10. 기술적 의사결정 (Technical Decisions)

### 10.1 왜 Windows IOCP를 선택했는가?
*   **문제 (기존 모델의 한계):** `Select`나 `WSAEventSelect`는 한 번에 관리 가능한 소켓 수(64개) 제한이 있고 O(N) 순회 비용이 듭니다. `WSAAsyncSelect`는 Windows 메시지 루프에 종속되어 부적합하며, 접속마다 스레드를 만드는 구조는 컨텍스트 스위칭의 오버헤드가 극심합니다.
*   **해결 및 결과:** OS 커널 단에서 스레드 풀링과 비동기 완료 통지를 O(1) 수준으로 관리해주는 **IOCP (I/O Completion Port)** 를 채택했습니다. 이를 통해 적은 수(논리 코어 수 수준)의 고정된 스레드만으로 1만 명 이상의 동시 접속을 컨텍스트 스위칭 병목 없이 안정적으로 처리할 수 있는 아키텍처를 완성했습니다.

### 10.2 Zero-Copy 최적화 도입 (Direct I/O)
*   **배경 (Fast I/O의 진실):** 일반적인 환경에서 TCP 송신 버퍼(커널 영역)에 공간이 남아 있다면, `WSASend`는 단순히 유저 데이터를 커널 버퍼로 메모리 복사(`memcpy`)하고 즉시 반환(Fast I/O)합니다. 이는 진정한 의미의 비동기 백그라운드 I/O가 아닙니다.
*   **해결 (Zero-Copy):** 소켓의 송신 버퍼 크기(SO_SNDBUF)를 0으로 설정하여 커널 복사를 원천 차단합니다. 대신 사용자가 전달한 버퍼 메모리에 페이지 락(Page Lock)을 걸어 페이징 아웃을 막은 뒤, 네트워크 드라이버(NIC)가 해당 유저 메모리 주소에서 데이터를 직접 읽어 전송(Direct I/O)하도록 구성했습니다.
*   **결과와 한계 (Trade-off):** 패킷 크기와 송신량이 막대할 때는 메모리 복사 비용을 없애주어 속도 향상을 가져옵니다. 하지만 송신량이 적을 때는 페이지 락을 걸고 푸는 OS 설정 비용이 메모리 복사 비용보다 더 비싸지며 성능이 저하될 수 있습니다. 따라서 이 기능은 네트워크 부하 특성에 맞게 선택할 수 있도록 옵션(Flag)으로 분리해 두었습니다.

## 11. 제한사항 및 알려진 이슈 (Limitations)
*   **64비트 아키텍처 종속성:** 본 라이브러리의 Lock-Free 큐는 ABA 문제 해결을 위해 64비트 포인터의 상위 17비트 미사용 영역을 마스킹하는 기법을 사용합니다. 따라서 향후 128비트 OS가 나오거나, 가상 메모리 주소 체계가 상위 17비트까지 확장되는 환경에서는 오작동을 일으킬 수 있습니다.

## 12. 관련 문서 (References)
*   [코어 컴포넌트 상세 문서](./Components)
*   [아키텍처 문서](./Architecture.md)
*   [부하 테스트 및 성능 분석](./Performance.md)
