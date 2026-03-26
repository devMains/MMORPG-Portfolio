# 모니터링 시스템 (Monitoring & Profiling)

---

## 1. 개요 (Overview)
- 서버의 리소스(CPU, 메모리, 네트워크 등)를 실시간으로 측정하고 기록하여 성능 병목과 장애 원인을 분석할 수 있게 돕는 자체 제작 모니터링 컴포넌트입니다.

- 해결하는 문제
  - 장시간 더미 클라이언트 부하 테스트 시 발생하는 메모리/핸들 누수 추적
  - 특정 동시 접속자 수 구간에서 발생하는 CPU 병목 지점 및 한계치 파악
  - 외부 무거운 프로파일러 툴에 의존하지 않는, 가볍고 독립적인 상시 모니터링 환경 구축
- 핵심 가치
  - Low Overhead: 수집 작업 자체가 서버 본연의 성능(Throughput)에 미치는 영향을 최소화
  - Observability: CSV 출력을 통한 데이터 시각화 및 직관적인 지표 분석

---

## 2. 수집 아키텍처 및 데이터 흐름 (Architecture & Flow)

- 각 수집 모듈이 주기적으로 OS로부터 데이터를 폴링(Polling)하고, 이를 중앙 집중형 클래스가 모아 파일 구조체로 변환한 뒤 디스크에 비동기/배치 방식으로 기록합니다.
- SystemMonitoring & ProcessMonitoring & CpuUsage -> CCollectMonitoringData -> csv 파일 기록 / DB 로깅

---

## 3. 핵심 컴포넌트 (Core Components)

### 3.1. SystemMonitoring & ProcessMonitoring
Windows의 PDH (Performance Data Helper) API를 기반으로 작동하며, OS 전체 상태와 서버 프로세스 단일 상태를 분리하여 수집합니다.

- 주요 수집 지표
  - System : 전체 CPU 사용량, 코어 별 CPU 사용량, 메모리, 논페이지드 메모리
  - Process : 프로세스 전체 CPU 사용량, 프로세스 유저 CPU 사용량, 프로세스 핸들, 프로세스 스레드 개수, 프로세스 논페이지드 메모리
- 설계 포인트
  - 매번 쿼리를 생성하지 않고 초기화 시점에 `PdhAddCounter`로 핸들을 캐싱한 뒤, 주기적으로 `PdhCollectQueryData`만 호출하여 수집 오버헤드를 크게 낮췄습니다.

### 3.2. CpuUsage (GetSystemTime 기반)
PDH를 사용하지 않고, Windows API인 `GetSystemTimeAsFileTime`과 `GetProcessTimes`를 직접 호출하여 CPU 사용률을 계산하는 독립 모듈입니다.

- 설계 이유
  - PDH를 이용하여 서버 컴퓨터의 CPU 사용률 측정 시 오차가 크게 발생하여 다른 방법으로 제작하였습니다.
  - PDH는 다양한 지표를 주지만 쿼리 비용이 상대적으로 무겁습니다.
  - 서버 성능 튜닝에서 가장 민감하고 자주 확인해야 하는 지표가 프로세스 CPU 사용률이므로, 커널 모드 시간과 유저 모드 시간을 틱(Tick) 단위로 직접 계산하여 가장 가볍고 정밀하게 수집하도록 별도로 분리했습니다.

### 3.3. CCollectMonitoringData (데이터 파이프라인)
위 모듈들에서 수집한 Raw 데이터를 모아 약속된 `Struct` 규격으로 변환하고, 엑셀이나 외부 툴(Python Pandas, Grafana 등)에서 쉽게 읽을 수 있도록 CSV 파일로 출력합니다. visitor_struct를 이용하여 struct 멤버의 이름을 얻어냅니다.

- 필수 필요 컴포넌트 : visit_struct.hpp [바로가기](https://github.com/cbeck88/visit_struct/blob/master/include/visit_struct/visit_struct.hpp)

- 사용 예시
```
struct MyStruct {
    uint64_t ts;
    float cpus;
    float mem;
    uint32_t sessions;
};

// visitor_struct에 내가 넣으려는 구조체의 정보를 등록
VISITABLE_STRUCT(MyStruct, ts, cpus, mem, sessions);

CCollectMonitoringData<MyStruct> mon("new.csv");

int main()
{
    MyStruct s;
    s.ts = timeGetTime();
    s.cpus = 66.6;
    s.mem = 44.4;
    s.sessions = 1234;
    mon.Save(s);
}
```

- 사용 결과 (new.csv)

|no|ts|cpus|mem|sessions|
|:---:|:---:|:---:|:---:|:---:|
|1|232323|66.6|44.4|1234|
|2|...|...|...|...|

---

## 4. 적용 사례 및 성과 (Use Cases & Results)

이 모니터링 시스템을 통해 다음과 같은 실제 서버 최적화를 이뤄냈습니다.

1. 모니터링 서버를 이용한 서버 상태 모니터링
- 직접 서버 접근을 하지 않고 외부에서 모니터링 클라이언트를 이용하여 확인할 수 있도록 모니터링 서버를 가동하였습니다.
- 각 서버는 로컬의 모니터링 서버에 연결하여 수집한 모니터링 데이터를 전송합니다.

2. 서버 상태 로깅
- 문제 발생 시 해당 시간에 무슨 상황이 발생하였는지 확인하기 위해 모니터링 데이터를 DB에 로깅합니다.
- 1분 주기로 데이터를 수집하고, 타입을 지정하고 타입 별로 최소/최대/평균 값을 저장합니다.
