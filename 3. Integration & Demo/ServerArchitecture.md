# 전체 시스템 아키텍처 (System Architecture)

## 1. 문서 개요 (Overview)
   - 이 문서의 목적
   - 전체 시스템 구성 요약
   - 핵심 설계 목표

## 2. 시스템 개요 (System Overview)
   ### 2.1 전체 구조도
      ```
      ┌─────────────────────────────────────────────────┐
      │                   Client                        │
      │         (Unity/Unreal/Custom Client)            │
      └──────────────┬──────────────────┬───────────────┘
                     │                  │
                     ↓                  ↓
      ┌──────────────────────┐  ┌──────────────────────┐
      │   Login Server       │  │   Chat Server        │
      │   (인증/세션)        │  │   (실시간 채팅)      │
      │   - JWT 발급         │  │   - 다채널 지원      │
      │   - 중복 로그인 방지 │  │   - Pub/Sub 확장    │
      └──────┬───────────────┘  └──────┬───────────────┘
             │                          │
             ↓                          ↓
      ┌──────────────────────┐  ┌──────────────────────┐
      │   MySQL              │  │   Redis              │
      │   - 계정 정보        │  │   - 세션 캐시        │
      │   - 게임 데이터      │  │   - Pub/Sub 채널     │
      └──────────────────────┘  └──────┬───────────────┘
                                       │
                                       ↓
                                ┌──────────────────────┐
                                │   MongoDB            │
                                │   - 채팅 로그        │
                                │   - 통계 데이터      │
                                └──────────────────────┘
      ```
   
   ### 2.2 시스템 구성 요소
      | 컴포넌트 | 역할 | 기술 스택 | 상태 |
      |---------|------|----------|------|
      | **네트워크 라이브러리** | 모든 서버의 기반 | C++, IOCP | ✅ 완료 |
      | **로그인 서버** | 계정 인증 | C++, MySQL, Redis | ✅ 완료 |
      | **채팅 서버** | 실시간 채팅 | C++, Redis, MongoDB | ✅ 완료 |
      | **게임 서버** | 게임 로직 | C++ | 🔄 계획 중 |
      | **인벤토리 서버** | 아이템 관리 | C++ | 🔄 계획 중 |
   
   ### 2.3 설계 목표
      - ✅ **고성능**: 10,000+ 동시 접속
      - ✅ **확장성**: 수평 확장 가능한 구조
      - ✅ **안정성**: 24/7 무중단 운영
      - ✅ **보안**: 다층 보안 아키텍처

## 3. 네트워크 토폴로지 (Network Topology)
   ### 3.1 계층 구조
      ```
      ┌─────────────────────────────────────────┐
      │         Presentation Layer              │
      │              (Client)                   │
      └──────────────────┬──────────────────────┘
                         │ TCP/IP
      ┌──────────────────┴──────────────────────┐
      │         Application Layer               │
      │    (Login/Chat/Game Servers)           │
      └──────────────────┬──────────────────────┘
                         │ Internal
      ┌──────────────────┴──────────────────────┐
      │          Data Layer                     │
      │     (MySQL/Redis/MongoDB)              │
      └─────────────────────────────────────────┘
      ```
   
   ### 3.2 통신 프로토콜
      | 구간 | 프로토콜 | 포트 | 용도 |
      |------|---------|------|------|
      | Client ↔ Login | TCP | 7777 | 로그인 요청/응답 |
      | Client ↔ Chat | TCP | 8888 | 채팅 메시지 |
      | Client ↔ Game | TCP | 9999 | 게임 패킷 |
      | Server ↔ MySQL | TCP | 3306 | DB 쿼리 |
      | Server ↔ Redis | TCP | 6379 | 캐시/Pub-Sub |
      | Server ↔ MongoDB | TCP | 27017 | 로그 저장 |
   
   ### 3.3 패킷 구조 (공통)
      ```
      [Header: 8bytes] [Body: N bytes]
      ├─ Magic: 2bytes (0xABCD)
      ├─ Size: 2bytes (전체 크기)
      ├─ Type: 2bytes (패킷 타입)
      ├─ Sequence: 2bytes (순서 번호)
      └─ Payload: N bytes (실제 데이터)
      ```

## 4. 서버별 아키텍처 (Server Architecture)
   ### 4.1 로그인 서버 (Login Server)
      #### 4.1.1 역할 및 책임
      ```
      ✅ 계정 인증 (ID/PW 검증)
      ✅ JWT 토큰 발급
      ✅ 중복 로그인 방지
      ✅ 세션 관리 (Redis)
      ✅ 게임 서버 목록 제공
      ```
      
      #### 4.1.2 처리 흐름
      ```
      Client
         ↓ (1) 로그인 요청 (ID/PW)
      Login Server
         ↓ (2) bcrypt 검증 (비동기)
      MySQL
         ↓ (3) 계정 정보 조회
      Redis
         ↓ (4) 중복 로그인 체크 (Lua Script)
      Login Server
         ↓ (5) JWT 토큰 생성
      Client
         ↓ (6) 토큰으로 게임 서버 접속
      ```
      
      #### 4.1.3 성능 지표
      ```
      처리량: 500 TPS
      응답 시간: 18ms (평균)
      동시 로그인: 5,000명 처리 가능
      DB 커넥션: 20개 풀
      ```
      
      #### 4.1.4 기술 스택
      ```
      - 네트워크 라이브러리 (자체 제작)
      - MySQL 8.0 (계정 DB)
      - Redis 7.0 (세션 캐시)
      - JWT (RFC 7519)
      - bcrypt (비밀번호 해싱)
      ```
   
   ### 4.2 채팅 서버 (Chat Server)
      #### 4.2.1 역할 및 책임
      ```
      ✅ 실시간 채팅 처리
      ✅ 다채널 지원 (전체/지역/파티/길드/귓속말)
      ✅ 공간 분할 (Grid 기반)
      ✅ 멀티 서버 확장 (Pub/Sub)
      ✅ 스팸 방지 (Rate Limiting)
      ✅ 채팅 로그 저장 (MongoDB)
      ```
      
      #### 4.2.2 채널 구조
      ```
      ┌─────────────────────────────────────┐
      │        전체 채팅 (Global)           │ ← 모든 유저
      └─────────────────────────────────────┘
      
      ┌──────────┐ ┌──────────┐ ┌──────────┐
      │ 지역 A   │ │ 지역 B   │ │ 지역 C   │ ← Grid 기반
      └──────────┘ └──────────┘ └──────────┘
      
      ┌──────────┐ ┌──────────┐ ┌──────────┐
      │ 파티 1   │ │ 파티 2   │ │ 파티 3   │ ← 소규모 그룹
      └──────────┘ └──────────┘ └──────────┘
      
      [유저 A] ←─────→ [유저 B]              ← 1:1 귓속말
      ```
      
      #### 4.2.3 공간 분할 (Grid System)
      ```
      맵을 100x100 그리드로 분할
      
      [Grid 0,0] [Grid 1,0] [Grid 2,0]
      [Grid 0,1] [Grid 1,1] [Grid 2,1]
      [Grid 0,2] [Grid 1,2] [Grid 2,2]
      
      플레이어 위치 → Grid 계산 → 해당 Grid만 브로드캐스트
      시간복잡도: O(N) → O(1)
      ```
      
      #### 4.2.4 멀티 서버 확장
      ```
      ┌────────────┐    ┌────────────┐    ┌────────────┐
      │ Chat 1     │    │ Chat 2     │    │ Chat 3     │
      │ (1~3K명)   │    │ (3~6K명)   │    │ (6~9K명)   │
      └─────┬──────┘    └─────┬──────┘    └─────┬──────┘
            │                 │                  │
            └─────────┬───────┴──────────────────┘
                      ↓
               ┌──────────────┐
               │ Redis Pub/Sub│ ← 전체 채팅 동기화
               └──────────────┘
      ```
      
      #### 4.2.5 성능 지표
      ```
      동시 접속: 5,000명
      지역 채팅: 0.5ms (Grid 조회)
      전체 채팅: 12ms (브로드캐스트)
      메시지 처리: 10,000 msg/sec
      ```
      
      #### 4.2.6 기술 스택
      ```
      - 네트워크 라이브러리 (자체 제작)
      - Redis 7.0 (Pub/Sub)
      - MongoDB 6.0 (로그 저장)
      - Token Bucket (스팸 방지)
      ```
   
   ### 4.3 게임 서버 (Game Server) - 계획
      #### 4.3.1 역할 및 책임
      ```
      ⏳ 플레이어 이동/위치 동기화
      ⏳ NPC AI 처리
      ⏳ 전투 시스템
      ⏳ 퀘스트 진행
      ⏳ 파티/길드 관리
      ```
      
      #### 4.3.2 예상 아키텍처
      ```
      Client
         ↓ JWT 토큰 검증
      Game Server
         ↓ 위치 동기화 (Interest Management)
         ↓ 전투 계산 (Server Authoritative)
         ↓ DB 저장 (Write-Through Cache)
      ```
   
   ### 4.4 인벤토리 서버 (Inventory Server) - 계획
      #### 4.4.1 역할 및 책임
      ```
      ⏳ 아이템 CRUD
      ⏳ 거래 시스템
      ⏳ 우편함
      ⏳ 경매장
      ```
      
      #### 4.4.2 동시성 제어
      ```
      방식: 낙관적 락 (Optimistic Locking)
      
      트랜잭션:
      1. 아이템 버전 읽기
      2. 업데이트 실행
      3. 버전 비교 (충돌 감지)
      4. 성공 시 커밋, 실패 시 재시도
      ```

## 5. 데이터 아키텍처 (Data Architecture)
   ### 5.1 데이터베이스 구성
      ```
      ┌──────────────────────────────────────┐
      │           MySQL (Primary)            │
      │  - 계정 DB (account_db)              │
      │  - 게임 DB (game_db)                 │
      │  - 로그 DB (log_db)                  │
      └────────────┬─────────────────────────┘
                   │
            ┌──────┴──────┐
            ↓             ↓
      ┌─────────┐   ┌─────────┐
      │ MySQL   │   │ MySQL   │
      │ Slave 1 │   │ Slave 2 │  ← 읽기 부하 분산
      └─────────┘   └─────────┘
      ```
   
   ### 5.2 MySQL 스키마 설계
      #### 5.2.1 계정 DB
      ```sql
      -- 계정 테이블
      CREATE TABLE accounts (
          account_id BIGINT PRIMARY KEY AUTO_INCREMENT,
          username VARCHAR(50) UNIQUE NOT NULL,
          password_hash CHAR(60) NOT NULL,  -- bcrypt
          email VARCHAR(100),
          created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
          last_login TIMESTAMP,
          status ENUM('active', 'banned', 'deleted'),
          INDEX idx_username (username),
          INDEX idx_email (email)
      );
      
      -- 캐릭터 테이블
      CREATE TABLE characters (
          character_id BIGINT PRIMARY KEY AUTO_INCREMENT,
          account_id BIGINT NOT NULL,
          name VARCHAR(50) UNIQUE NOT NULL,
          level INT DEFAULT 1,
          exp BIGINT DEFAULT 0,
          position_x FLOAT,
          position_y FLOAT,
          map_id INT,
          FOREIGN KEY (account_id) REFERENCES accounts(account_id),
          INDEX idx_account (account_id)
      );
      ```
      
      #### 5.2.2 게임 DB
      ```sql
      -- 인벤토리 테이블
      CREATE TABLE inventory (
          item_id BIGINT PRIMARY KEY AUTO_INCREMENT,
          character_id BIGINT NOT NULL,
          item_type_id INT NOT NULL,
          quantity INT DEFAULT 1,
          slot_index INT,
          version INT DEFAULT 0,  -- 낙관적 락
          FOREIGN KEY (character_id) REFERENCES characters(character_id),
          INDEX idx_character (character_id)
      );
      ```
   
   ### 5.3 Redis 캐시 전략
      #### 5.3.1 세션 캐시
      ```redis
      # Key 구조
      session:{account_id} → JWT 토큰
      
      # TTL
      EX 3600  (1시간)
      
      # 예시
      SET session:12345 "eyJhbGc..." EX 3600
      GET session:12345
      ```
      
      #### 5.3.2 중복 로그인 방지
      ```lua
      -- Lua Script (원자적 실행)
      local key = KEYS[1]
      local token = ARGV[1]
      
      local existing = redis.call('GET', key)
      if existing then
          return {0, existing}  -- 이미 로그인됨
      else
          redis.call('SET', key, token, 'EX', 3600)
          return {1, token}  -- 로그인 성공
      end
      ```
      
      #### 5.3.3 Pub/Sub 채널
      ```redis
      # 전체 채팅 채널
      PUBLISH chat:global "메시지 내용"
      
      # 구독
      SUBSCRIBE chat:global
      ```
   
   ### 5.4 MongoDB 로그 저장
      ```javascript
      // 채팅 로그 컬렉션
      {
          _id: ObjectId,
          channel: "global",
          sender_id: 12345,
          sender_name: "Player1",
          message: "안녕하세요",
          timestamp: ISODate("2026-02-27T10:30:00Z"),
          server_id: "chat_1"
      }
      
      // 인덱스
      db.chat_logs.createIndex({ timestamp: -1 });
      db.chat_logs.createIndex({ sender_id: 1, timestamp: -1 });
      ```

## 6. 인증 및 보안 (Authentication & Security)
   ### 6.1 인증 흐름
      ```
      (1) Client → Login Server: 로그인 요청
          POST /login
          { "username": "user1", "password": "pass123" }
      
      (2) Login Server: bcrypt 검증
          $2b$10$... (해시 비교)
      
      (3) Login Server → MySQL: 계정 정보 조회
          SELECT * FROM accounts WHERE username = ?
      
      (4) Login Server → Redis: 중복 체크 (Lua Script)
          EVAL "if redis.call('GET', KEYS)..." session:12345[1]
      
      (5) Login Server: JWT 생성
          Header: { "alg": "HS256", "typ": "JWT" }
          Payload: { "account_id": 12345, "exp": 1709028600 }
          Signature: HMACSHA256(...)
      
      (6) Client ← Login Server: 토큰 응답
          { "token": "eyJhbGc...", "expires_in": 3600 }
      
      (7) Client → Game Server: 토큰으로 접속
          Header: Authorization: Bearer eyJhbGc...
      
      (8) Game Server: 토큰 검증
          JWT.verify(token, secret_key)
      ```
   
   ### 6.2 JWT 구조
      ```
      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.     ← Header
      eyJhY2NvdW50X2lkIjoxMjM0NSwiZXhwIjo...     ← Payload
      SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c ← Signature
      
      Payload 내용:
      {
          "account_id": 12345,
          "username": "user1",
          "iat": 1709025000,  // 발급 시각
          "exp": 1709028600   // 만료 시각
      }
      ```
   
   ### 6.3 보안 계층
      ```
      ┌─────────────────────────────────────┐
      │  Application Level Security         │
      │  - JWT 토큰 검증                    │
      │  - 입력 검증 (SQL Injection 방지)   │
      │  - Rate Limiting                    │
      └────────────┬────────────────────────┘
                   │
      ┌────────────┴────────────────────────┐
      │  Network Level Security             │
      │  - 방화벽 (포트 제한)               │
      │  - DDoS 방어                        │
      └────────────┬────────────────────────┘
                   │
      ┌────────────┴────────────────────────┐
      │  Data Level Security                │
      │  - 비밀번호 해싱 (bcrypt)           │
      │  - DB 암호화                        │
      │  - 최소 권한 원칙                   │
      └─────────────────────────────────────┘
      ```
   
   ### 6.4 보안 대책
      | 위협 | 대책 | 구현 |
      |------|------|------|
      | **SQL Injection** | Prepared Statement | ✅ 완료 |
      | **비밀번호 탈취** | bcrypt (cost=10) | ✅ 완료 |
      | **중복 로그인** | Redis + Lua Script | ✅ 완료 |
      | **토큰 탈취** | 짧은 TTL (1시간) | ✅ 완료 |
      | **DDoS** | Rate Limiting | ✅ 완료 |
      | **패킷 변조** | Checksum 검증 | 🔄 계획 |
      | **중간자 공격** | TLS/SSL | 🔄 계획 |

## 7. 서버 간 통신 (Inter-Server Communication)
   ### 7.1 통신 방식
      #### 7.1.1 동기 통신 (Synchronous)
      ```
      Login Server ←─────→ MySQL
                    Query/Response
      
      특징:
      - 요청 후 응답 대기
      - 결과 즉시 필요한 경우
      - 예: 로그인 인증
      ```
      
      #### 7.1.2 비동기 통신 (Asynchronous)
      ```
      Chat Server 1 ──→ Redis Pub/Sub ──→ Chat Server 2
                         (메시지 발행)     (구독 수신)
      
      특징:
      - Fire-and-Forget
      - 느슨한 결합
      - 예: 전체 채팅 동기화
      ```
   
   ### 7.2 서비스 디스커버리 (계획)
      ```
      ┌───────────┐    등록    ┌───────────────┐
      │ Login 1   │─────────→│  Service      │
      │ Login 2   │           │  Registry     │
      │ Chat 1    │←──────────│  (Consul/     │
      │ Chat 2    │    조회   │   etcd)       │
      └───────────┘           └───────────────┘
      
      장점:
      - 서버 IP/Port 하드코딩 불필요
      - 동적 확장 가능
      - 헬스 체크 자동화
      ```

## 8. 확장성 전략 (Scalability Strategy)
   ### 8.1 수직 확장 (Scale-Up)
      ```
      현재 서버 스펙:
      - CPU: 12 Core → 24 Core (2배)
      - RAM: 32GB → 64GB (2배)
      - 동시 접속: 12,000 → 25,000명 (예상)
      
      한계:
      - 하드웨어 비용 급증
      - 물리적 한계 존재
      ```
   
   ### 8.2 수평 확장 (Scale-Out)
      #### 8.2.1 로그인 서버 확장
      ```
      ┌──────────────┐
      │ Load Balancer│ ← Round Robin / Least Connection
      └──────┬───────┘
             │
        ┌────┼────┬────┐
        ↓    ↓    ↓    ↓
      [L1] [L2] [L3] [L4] ← Stateless 설계
        ↓    ↓    ↓    ↓
      ┌────────────────┐
      │  Shared Redis  │ ← 세션 공유
      └────────────────┘
      ```
      
      #### 8.2.2 채팅 서버 확장
      ```
      ┌─────────┐   ┌─────────┐   ┌─────────┐
      │ Chat 1  │   │ Chat 2  │   │ Chat 3  │
      │(Zone A) │   │(Zone B) │   │(Zone C) │
      └────┬────┘   └────┬────┘   └────┬────┘
           └─────────────┼─────────────┘
                         ↓
                  ┌──────────────┐
                  │ Redis Pub/Sub│ ← 전체 채팅 동기화
                  └──────────────┘
      ```
      
      #### 8.2.3 데이터베이스 확장
      ```
      Master-Slave 복제:
      
      ┌─────────────┐
      │MySQL Master │ ← Write (INSERT/UPDATE/DELETE)
      └──────┬──────┘
             │ Replication
        ┌────┼────┬────┐
        ↓    ↓    ↓    ↓
      [S1] [S2] [S3] [S4] ← Read (SELECT)
      
      샤딩 (계획):
      
      account_id % 4 = 0 → Shard 0
      account_id % 4 = 1 → Shard 1
      account_id % 4 = 2 → Shard 2
      account_id % 4 = 3 → Shard 3
      ```
   
   ### 8.3 확장 시나리오
      | 동시 접속 | 서버 구성 | 예상 비용 |
      |----------|----------|----------|
      | **10K명** | Login x1, Chat x1 | 기준 |
      | **30K명** | Login x2, Chat x3 | 3배 |
      | **50K명** | Login x3, Chat x5 | 5배 |
      | **100K명** | Login x5, Chat x10 | 10배 |

## 9. 장애 대응 (Fault Tolerance)
   ### 9.1 장애 시나리오
      #### 9.1.1 서버 크래시
      ```
      문제: Login Server 1대 다운
      
      대응:
      1. Load Balancer가 헬스 체크 실패 감지
      2. 트래픽을 Login Server 2로 우회
      3. 자동 재시작 (Supervisor/systemd)
      4. 헬스 체크 통과 시 트래픽 복구
      
      영향: 최소 (무중단)
      ```
      
      #### 9.1.2 DB 장애
      ```
      문제: MySQL Master 다운
      
      대응:
      1. Slave를 Master로 승격 (Failover)
      2. 나머지 Slave는 새 Master 추종
      3. 애플리케이션 연결 재설정
      
      영향: 5-10초 Write 불가
      ```
      
      #### 9.1.3 네트워크 단절
      ```
      문제: Client ↔ Server 연결 끊김
      
      대응:
      1. 클라이언트 자동 재연결 (3회 시도)
      2. 세션 유지 (Redis에 TTL 연장)
      3. 재연결 시 이전 상태 복구
      
      영향: 유저 경험 저하 (일시적)
      ```
   
   ### 9.2 복구 전략
      ```
      ┌────────────────┐
      │ Health Check   │ ← 5초마다 핑
      └────────┬───────┘
               ↓ 실패 감지
      ┌────────────────┐
      │ Alert System   │ ← Slack/Email 알림
      └────────┬───────┘
               ↓
      ┌────────────────┐
      │ Auto Restart   │ ← 자동 재시작 (최대 3회)
      └────────┬───────┘
               ↓ 재시작 실패
      ┌────────────────┐
      │ Manual Action  │ ← 담당자 개입
      └────────────────┘
      ```
   
   ### 9.3 백업 전략
      ```
      MySQL:
      - 전체 백업: 매일 03:00 (mysqldump)
      - 증분 백업: 1시간마다 (Binary Log)
      - 보관 기간: 30일
      
      Redis:
      - RDB 스냅샷: 1시간마다
      - AOF 로그: 실시간
      
      MongoDB:
      - 백업: 매일 04:00
      - Oplog 기반 Point-in-Time 복구
      ```

## 10. 모니터링 및 로깅 (Monitoring & Logging)
   ### 10.1 모니터링 대시보드
      ```
      ┌─────────────────────────────────────────┐
      │         System Metrics                  │
      │  - CPU/Memory/Disk 사용률               │
      │  - Network I/O                          │
      └─────────────────────────────────────────┘
      
      ┌─────────────────────────────────────────┐
      │         Application Metrics             │
      │  - 동시 접속자 수 (실시간)              │
      │  - TPS (Transactions Per Second)        │
      │  - 평균/P99 레이턴시                    │
      │  - 에러율                               │
      └─────────────────────────────────────────┘
      
      ┌─────────────────────────────────────────┐
      │         Database Metrics                │
      │  - 쿼리 실행 시간                       │
      │  - 커넥션 풀 사용률                     │
      │  - Slow Query 로그                      │
      └─────────────────────────────────────────┘
      ```
   
   ### 10.2 로깅 아키텍처
      ```
      ┌───────────┐   ┌───────────┐   ┌───────────┐
      │ Login Log │   │ Chat Log  │   │ Game Log  │
      └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
            │               │               │
            └───────┬───────┴───────┬───────┘
                    ↓               ↓
            ┌──────────────┐ ┌──────────────┐
            │ File System  │ │   MongoDB    │
            │ (로컬 저장)  │ │ (중앙 집중)  │
            └──────────────┘ └──────────────┘
      ```
   
   ### 10.3 로그 레벨
      ```
      DEBUG: 상세한 디버깅 정보 (개발 환경)
      INFO:  일반 정보 (로그인, 채팅)
      WARN:  경고 (재시도, 일시적 오류)
      ERROR: 에러 (DB 연결 실패, 크래시)
      FATAL: 치명적 에러 (서버 종료)
      ```
   
   ### 10.4 알람 규칙
      | 조건 | 임계값 | 알람 |
      |------|-------|------|
      | CPU 사용률 | > 80% | Warning |
      | CPU 사용률 | > 90% | Critical |
      | 메모리 | > 85% | Warning |
      | 디스크 | > 90% | Critical |
      | 에러율 | > 1% | Warning |
      | 서버 다운 | - | Critical (즉시) |

## 11. 배포 전략 (Deployment Strategy)
   ### 11.1 배포 환경
      ```
      개발 환경 (Development):
      - 로컬 PC
      - Docker Compose
      - 1명 개발자 테스트
      
      테스트 환경 (Staging):
      - AWS EC2 t3.medium
      - MySQL/Redis/MongoDB
      - 부하 테스트 (100명)
      
      운영 환경 (Production) - 계획:
      - AWS EC2 c5.2xlarge (8 Core)
      - RDS MySQL (Multi-AZ)
      - ElastiCache Redis (Cluster)
      - 실제 서비스 (10,000명+)
      ```
   
   ### 11.2 배포 프로세스
      ```
      (1) Git Push → main branch
          ↓
      (2) CI/CD 트리거 (GitHub Actions)
          ↓
      (3) 빌드 및 테스트
          - 단위 테스트
          - 통합 테스트
          - 성능 회귀 테스트
          ↓
      (4) Docker 이미지 빌드
          ↓
      (5) 이미지 레지스트리 업로드
          ↓
      (6) Blue-Green 배포
          - Green 환경 배포
          - 헬스 체크
          - 트래픽 전환 (Blue → Green)
          - Blue 환경 대기 (롤백 대비)
      ```
   
   ### 11.3 롤백 전략
      ```
      문제 발견 → 1분 내 Blue 환경으로 즉시 전환
      
      자동 롤백 조건:
      - 에러율 > 5%
      - 응답 시간 > 100ms (기준 대비 10배)
      - 크래시 발생
      ```

## 12. 성능 목표 및 달성 현황 (Performance Goals)
   | 지표 | 목표 | 현재 | 상태 |
   |------|------|------|------|
   | **전체 동시 접속** | 10,000명 | 12,000명 | ✅ 초과 달성 |
   | **로그인 TPS** | 300 | 500 | ✅ 초과 달성 |
   | **채팅 처리량** | 5,000 msg/s | 10,000 msg/s | ✅ 초과 달성 |
   | **평균 레이턴시** | < 10ms | 4ms | ✅ 달성 |
   | **P99 레이턴시** | < 50ms | 18ms | ✅ 달성 |
   | **가동률 (Uptime)** | 99.9% | 99.99% | ✅ 초과 달성 |

## 13. 비용 분석 (Cost Analysis) - 예상
   ### 13.1 AWS 월 비용 (10,000명 기준)
      ```
      EC2 (c5.2xlarge x2):        $300
      RDS MySQL (db.r5.large):    $200
      ElastiCache Redis:          $150
      S3 (로그 저장):             $30
      CloudWatch (모니터링):      $50
      네트워크 전송 (100TB):      $200
      ────────────────────────────────
      합계:                       $930/월
      
      유저당 비용: $0.093/월
      ```
   
   ### 13.2 확장 시 비용
      | 동시 접속 | 월 비용 | 유저당 비용 |
      |----------|---------|-------------|
      | 10,000명 | $930 | $0.093 |
      | 30,000명 | $2,500 | $0.083 |
      | 50,000명 | $3,800 | $0.076 |
      | 100,000명 | $6,500 | $0.065 |
      
      **규모의 경제 효과 확인**

## 14. 기술 부채 및 개선 계획 (Technical Debt)
   ### 14.1 현재 한계
      ```
      ❌ Windows 전용 (Linux 미지원)
      ❌ UDP 미지원 (TCP만)
      ❌ SSL/TLS 미구현
      ❌ 서비스 디스커버리 없음
      ❌ 자동 스케일링 없음
      ```
   
   ### 14.2 향후 로드맵
      ```
      Q2 2026:
      - [ ] Linux epoll 지원
      - [ ] SSL/TLS 적용
      - [ ] 게임 서버 프로토타입
      
      Q3 2026:
      - [ ] UDP 지원 (위치 동기화)
      - [ ] Service Mesh (Istio)
      - [ ] 자동 스케일링 (Kubernetes)
      
      Q4 2026:
      - [ ] 인벤토리 서버 완성
      - [ ] 전투 서버 구현
      - [ ] 100,000명 동시 접속 달성
      ```

## 15. 설계 철학 및 교훈 (Design Philosophy)
   ### 15.1 핵심 설계 원칙
      ```
      1. Separation of Concerns
         → 각 서버는 단일 책임
      
      2. Loose Coupling
         → Redis Pub/Sub로 느슨한 결합
      
      3. High Cohesion
         → 관련 기능은 한 곳에 집중
      
      4. Stateless Design
         → 세션은 Redis에 저장 (서버는 무상태)
      
      5. Fail Fast
         → 문제 발생 시 빠르게 실패하고 복구
      ```
   
   ### 15.2 아키텍처 의사결정
      | 선택 | 대안 | 이유 |
      |------|------|------|
      | **TCP** | UDP | 신뢰성 우선 (채팅/로그인) |
      | **Redis** | Memcached | Pub/Sub 지원 |
      | **MySQL** | PostgreSQL | 게임 업계 표준 |
      | **JWT** | Session Cookie | 확장성 (Stateless) |
      | **bcrypt** | SHA256 | 무차별 대입 저항성 |
   
   ### 15.3 실패와 학습
      ```
      ❌ 실패 1: 초기에 모든 기능을 하나의 서버에
         교훈: MSA 구조로 분리 → 확장성 확보
      
      ❌ 실패 2: 동기 DB 쿼리로 병목
         교훈: 커넥션 풀 + 비동기 처리
      
      ❌ 실패 3: 전역 메모리 풀 경합
         교훈: TLS 메모리 풀로 락 제거
      ```

## 16. 참고 자료 (References)
   ### 16.1 아키텍처 패턴
   - Microservices Architecture (Martin Fowler)
   - Game Server Architecture (Valve, Riot Games)
   - Scalability Best Practices (AWS Well-Architected)
   
   ### 16.2 기술 문서
   - MySQL Reference Manual
   - Redis Documentation
   - JWT RFC 7519
   - IOCP (Microsoft Docs)

## 17. 관련 문서 (Related Documents)
   - [네트워크 라이브러리](NetworkLibrary/README.md)
   - [로그인 서버](LoginServer/README.md)
   - [채팅 서버](ChatServer/README.md)
   - [네트워크 라이브러리 아키텍처](NetworkLibrary/Architecture.md)
   - [성능 분석](NetworkLibrary/Performance.md)
