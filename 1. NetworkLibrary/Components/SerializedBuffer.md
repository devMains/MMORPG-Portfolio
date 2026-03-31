# [SerializedBuffer]

## 1. 개요 (Overview)

| 항목 | 상세 내용 |
| :--- | :--- |
| **요약** | 패킷의 헤더와 바디를 하나의 연속된 버퍼에 담아 데이터를 직렬화(Serialize) 및 역직렬화(Deserialize)하는 전용 버퍼 클래스 |
| **핵심 기술** | • **오버로딩 연산자(`<<`, `>>`):** 직관적이고 빠른 패킷 조립/파싱<br>• **참조 카운트(`_ref`):** 비동기 송수신 환경에서의 안전한 메모리 수명 관리 |
| **해결 과제** | • 네트워크 패킷 조립 시 발생하는 반복적인 `memcpy`와 헤더/바디 분리에 따른 번거로움 해소<br>• 잦은 동적 할당(`new`/`delete`)으로 인한 성능 저하 및 메모리 파편화 방지 |

## 2. 핵심 개념 (Core Concepts)

*   **커서 기반(Offset-based) 데이터 처리:** 하나의 선형 버퍼 내에서 읽기 위치(`_readPos`)와 쓰기 위치(`_writePos`) 커서를 이동시키며 데이터를 생산하고 소비합니다.
*   **객체 풀과 수명 관리:** `Alloc`, `Free` 메서드를 통해 내부 오브젝트 풀에서 인스턴스를 재사용하며, `IncRef`, `DecRef`를 통한 참조 카운트 기반으로 다중 스레드(비동기 I/O) 환경에서 버퍼의 수명을 안전하게 관리합니다.
*   **직관적인 인터페이스:** 기본 타입에 대해 `<<`, `>>` 연산자가 오버라이딩되어 있어 `Push`/`Pop` 메서드 호출 없이도 C++ 스트림 방식처럼 매우 직관적인 데이터 입출력이 가능합니다. 사용자 정의 클래스나 구조체도 연산자 오버로딩을 통해 쉽게 확장할 수 있습니다.

## 3. 구현 상세 (Implementation Details)

### 클래스 구조
*   **책임:** 패킷 단위의 직렬화/역직렬화 버퍼 제공, 헤더와 바디 영역의 통합 관리, 객체 풀과 참조 카운트를 통한 수명 관리 규칙 강제.
*   **상태:** 
    *   `_buffer`: 실제 데이터가 저장되는 메모리 주소
    *   `_bufferSize`: 버퍼의 전체 할당 크기
    *   `_useSize`: 현재 기록된 유효 데이터(바디)의 크기
    *   `_readPos`, `_writePos`: 데이터 읽기 및 쓰기 커서 위치
    *   `_ref`: 참조 카운트 (비동기 I/O 중 생존 보장용)
    *   `pool`: 버퍼 재사용을 위한 정적(static) 풀 객체
    *   `encoded`: 페이로드 인코딩/암호화 적용 여부 플래그
*   **작동 로직:** `Alloc`으로 풀에서 버퍼를 받아온 뒤 `Clear`로 내부 상태를 초기화합니다. 이후 `<<` 연산자나 `Push`로 데이터를 쓰고, `SetHeader`를 통해 미리 정의된 헤더 데이터를 버퍼 앞단에 기록합니다.

### 주요 메서드
*   `operator<<`, `operator>>`: 기본 자료형 데이터를 버퍼에 직렬화하거나 역직렬화하며 내부 커서를 자동 이동시킵니다.
*   `Push(char* data, int size)`, `Pop(char* dest, int size)`: 지정한 포인터의 데이터를 바이트 단위로 넣거나 빼며, `_writePos`와 `_readPos`를 이동시킵니다.
*   `Peek(char* dest, int size)`: 포인터에 데이터를 복사하여 확인하지만, 데이터를 소모하지 않으므로 `_readPos`는 이동시키지 않습니다.
*   `SetHeader(char* headerData)`: 전달된 헤더 데이터를 버퍼의 가장 앞단(헤더 영역)에 복사하여 패킷을 완성합니다.
*   `SetUseSize(int size)`: 현재 사용 중인 데이터의 크기를 임의로 조정합니다.
*   `GetHeaderPtr()`, `GetBufferPtr()`: 네트워크 전송(`send`, `WSASend`) 시 사용할 헤더의 시작 위치와 바디의 시작 위치 포인터를 반환합니다.
*   `static Alloc()`, `static Free(CSerializedBuffer* buffer)`: 정적 풀에서 버퍼를 할당받거나 반환합니다. `Free` 호출 시 `_ref`(참조 카운트)가 0인 경우에만 실제 풀로 반환되어 재사용됩니다.

![내부 메커니즘](../../Images/SerializedBuffer.png)

## 4. 사용 예시 (Usage Examples)

### 기본 사용법 (패킷 조립 및 파싱)
```cpp
// [송신부] 버퍼 할당 및 패킷 조립
CSerializedBuffer* buffer = CSerializedBuffer::Alloc();
buffer->Clear();

short type = 2;
long long id = 1;

// 바디 데이터 직렬화
*buffer << type << id;

// 헤더 설정 (미리 정의된 구조체 사용)
char header[HEADER_SIZE];
// ... header 구성 로직 ...
buffer->SetHeader(header);

// 소켓으로 전송 (헤더와 바디를 연속된 메모리로 한 번에 전송)
send(socket, buffer->GetHeaderPtr(), buffer->GetHeaderSize() + buffer->GetUseSize(), 0);

// ----------------------------------------------------

// [수신부] 데이터 역직렬화 (파싱)
short recvType;
long long recvId;

// 버퍼에서 순서대로 데이터 추출
*buffer >> recvType >> recvId;

switch (recvType) {
    case TYPE_LOGIN:
        // 처리 로직
        break;
}
```

### 고급 사용법 (다중 패킷 배치 전송 - Scatter-Gather)
```cpp
CSerializedBuffer* buffers[BUFFER_COUNT];
WSABUF wsabuf[BUFFER_COUNT];

// 전송할 여러 개의 완성된 버퍼를 WSABUF 배열에 매핑
for (int i = 0; i < BUFFER_COUNT; i++) {
    wsabuf[i].buf = buffers[i]->GetHeaderPtr();
    wsabuf[i].len = buffers[i]->GetHeaderSize() + buffers[i]->GetUseSize();
}

// 여러 패킷을 단 한 번의 시스템 콜로 모아서 전송
WSASend(socket, wsabuf, BUFFER_COUNT, ...);
```

## 5. 기술적 도전과 해결 (Technical Challenges)

*   **[문제] 네트워크와 비즈니스 레이어 간의 메모리 소유권 분쟁**
    *   네트워크 I/O 스레드가 수신한 버퍼를 로직 스레드(콘텐츠 코드)로 넘길 때, 비동기 송수신과 맞물려 버퍼를 어느 시점에 해제해야 하는지 모호해지는 문제(Use-after-free 위험)가 발생했습니다.
*   **[해결 1] 참조 카운트(Reference Counting) 도입**
    *   버퍼 자체에 `_ref` 카운트를 내장하여, 버퍼가 여러 큐에 들어가거나 비동기 I/O가 진행 중일 때 생존을 완벽하게 보장하도록 설계했습니다.
*   **[해결 2] 정적 오브젝트 풀(Object Pool) 연동**
    *   패킷이 생성되고 버려질 때마다 발생하는 `new`/`delete` 오버헤드를 막기 위해, 참조 카운트가 0이 되는 시점에 메모리를 해제하지 않고 클래스 내부의 `static pool`로 반환하여 재사용하도록 강제했습니다. 이를 통해 메모리 할당 속도를 극적으로 높이고 파편화를 방지했습니다.

## 6. 트레이드오프 (Trade-offs)

*   **장점:** 콘텐츠 프로그래머 입장에서는 내부 메모리 구조를 몰라도 C++ 스트림 방식(`<<`, `>>`)으로 매우 직관적이고 빠르게 패킷을 조립하고 사용할 수 있습니다.
*   **단점:** 구조체 전체를 통째로 버퍼에 넣을 경우, 패딩 바이트나 불필요한 필드까지 함께 직렬화되어 실제 네트워크 트래픽 낭비가 발생할 수 있습니다. (개별 멤버 변수 단위로 직렬화하는 규율이 필요합니다.)
*   **적합한 사용 케이스:** TCP/UDP 기반의 네트워크 패킷 생성/파싱 파이프라인, 스레드 간 데이터 메시지 전달.

## 7. 향후 개선 방향 (Future Improvements)
*(필요 시 추가 작성 예정)*
