# [RingBuffer]

## 1. 개요 (Overview)
- L4에서 분할된 데이터를 모아서 1개 이상의 완성된 메시지로 만들기 위해 사용하는 L7에서의 버퍼
- 수신용 : 현재 TCP는 스트림 데이터 방식으로 L7에서 Send를 호출한 단위대로 전송하지 않는다. 따라서 데이터가 분할되어 전송될 수 있기 때문에 완성되지 않은 메시지를 들고 있을 필요가 있다. 이 링버퍼는 해당 문제를 위해 제작되었다.
- 송신용 : 서버를 만드는 과정에서 전송하고 싶은 데이터가 생길 때마다 콜 타임이 긴 Send를 호출하게 된다면 콘텐츠의 속도에 영향을 줄 수 있다. 따라서 전송하고 싶은 데이터를 모아서 한 번에 전송할 수 있게 모아준다.

## 2. 핵심 개념 (Core Concepts)
- 데이터를 바이트 단위로 넣고 뺄 수 있는 환형 큐로, 원형 큐의 자료구조를 이용하였다.
- Front, Rear를 이용하여 데이터를 삽입해야 할 위치와 데이터를 꺼낼 수 있는 위치를 기록하고 이용한다.

## 3. 구현 상세 (Implementation Details)
- 클래스 구조
  - 책임 : 네트워크 수신 시 바이트 스트림을 임시 저장, 완전한 패킷으로 조립하기 위한 순환 버퍼
  - 상태 : _buffer(데이터가 저장되는 메모리), _front(데이터를 읽어갈 위치), _rear(데이터를 쓸 위치), _size(_buffer의 크기)
  - 작동 : 생성 시 1회 _buffer를 할당하고 소멸 시 해제한다. 복사 및 대입 생성자를 막아 복사를 방지한다.
- 주요 메서드
  - GetUseSize, GetFreeSize : 전체 큐의 사용 공간과 남는 공간을 반환한다.
  - DirectEnqueueSize, DirectDequeueSize : 배열의 경계 때문에 중간에 끊기는 부분을 인식하거나 front에 의해 데이터를 쓸 수 없거나 rear에 의해 데이터를 읽으면 안되는 부분이 존재한다. 안전하게 Enqueue 및 Dequeue를 하기 위해 가능한 크기를 반환한다.
  - Enqueue(char*, int) : 지정한 문자열을 지정한 크기만큼 넣는다. 넣은 크기만큼 반환한다. _rear를 크기만큼 이동한다.
  - Dequeue(char*, int) : 지정한 위치에 지정한 크기만큼 읽는다. 읽은 크기만큼 반환한다. _front를 크기만큼 이동한다.
  - Peek(char*, int) : 지정한 위치에 지정한 크기만큼 읽는다. 읽은 크기만큼 반환한다. _front를 이동하지 않는다.
  - GetBufferPtr, GetFrontBufferPtr, GetRearBufferPtr : 기본 버퍼의 위치, _front가 가리키는 위치, _rear가 가리키는 위치를 char*로 반환한다.
  - MoveRear(int), MoveFront(int) : _rear와 _front를 이동시킨다.
  - Lock, Unlock : 내부의 SRWLock 객체를 이용하여 AcquireSRWLockExclusive, ReleaseSRWLockExclusive를 래핑하여 필요 시 동기화를 할 수 있다.

![내부 메커니즘](../../Images/CRingBuffer.png)

## 4. 사용 예시 (Usage Examples)
- 기본 사용법
  ```
  char buffer[MAX_BUFFER_SIZE];
  CRingBuffer ringbuffer;

  // 메시지 수신
  int recvSize = recv(socket, buffer, sizeof(buffer), 0);
  ringbuffer.Enqueue(buffer, recvSize);

  // 완성된 메시지 체크
  char header[HEADER_SIZE];
  ringbuffer.Peek(header, HEADER_SIZE);
  if (header->len < ringbuffer.GetUseSize())
    ringbuffer.Dequeue(buffer, ringbuffer.GetUseSize());
  ```
- 고급 사용법
  ```
  char buffer[MAX_BUFFER_SIZE];
  CRingBuffer ringbuffer;

  // 메시지 수신, 포인터 전달로 복사 감소
  int recvSize = recv(socket, ringbuffer.GetRearBufferPtr(), ringbuffer.DirectEnqueueSize(), 0);
  ringbuffer.MoveRear(recvSize);

  // 완성된 메시지 체크
  char header[HEADER_SIZE];
  ringbuffer.Peek(header, HEADER_SIZE);
  if (header->len < ringbuffer.GetUseSize())
    ringbuffer.Dequeue(buffer, ringbuffer.GetUseSize());

  /////////////////////////////////////////////////////////////
  CRingBuffer ringbuffer;
  WSABUF wsabuf[2];

  wsabuf[0].buf = ringbuffer.GetRearBufferPtr();
  wsabuf[0].len = ringbuffer.DirectEnqueueSize();
  if (ringbuffer.GetFreeSize() > ringbuffer.DirectEnqueueSize()) {
    wsabuf[1].buf = ringbuffer.GetBufferPtr();
    wsabuf[1].len = ringbuffer.GetFreeSize() - ringbuffer.DirectEnqueueSize();
  }

  ...
  // 모든 버퍼 위치를 등록할 수 있음
  WSARecv(socket, &wsabuf, bufCnt, ...);
  
  ```

## 5. 기술적 도전과 해결 (Technical Challenges)
- 설계한 링버퍼는 멀티스레드 환경에서 안전하지 않다. 따라서 Lock, Unlock 기능을 추가하여 동기화를 진행하였다.
- 그러나 Enqueue와 Dequeue는 서로 독립적으로 진행될 수 있다고 판단하여 Enqueue하는 스레드 1개, Dequeue하는 스레드 1개에 대해서는 동시 작동을 허용하는 것이 좋다고 생각했다. 이 조건을 지킨다면 굳이 동기화 객체를 쓰지 않아도 멀티스레드 환경에서 안전하게 작동할 수 있다.
- 멀티스레드 환경에서 주요 문제는 함수 진행 중 실시간으로 큐의 상태가 바뀐다는 것이다. 이를 해결하기 위해 함수 호출 시 front와 rear를 변수로 캡쳐 후 로직을 진행한다. 이로써 캡쳐된 큐의 상태를 보고 함수를 진행한다.

## 6. 트레이드오프 (Trade-offs)
- 장점 : 간단하게 미완성된 메시지를 들고 있을 수 있다.
- 단점 : 수신한 데이터를 사용하기 위해서는 다른 버퍼로 복사가 필수이다.
- 적합한 사용 케이스 : 네트워크 수신부

## 7. 향후 개선 방향 (Future Improvements)
- 

