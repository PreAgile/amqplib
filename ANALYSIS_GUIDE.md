# amqplib 오픈소스 분석 가이드

> 체계적인 코드 분석을 위한 단계별 학습 로드맵

## 목차

- [개요](#개요)
- [사전 준비](#사전-준비)
- [Phase 1: 프로젝트 오리엔테이션](#phase-1-프로젝트-오리엔테이션)
- [Phase 2: 사용자 관점 이해](#phase-2-사용자-관점-이해)
- [Phase 3: 진입점 분석](#phase-3-진입점-분석)
- [Phase 4: 프로토콜 계층 이해](#phase-4-프로토콜-계층-이해)
- [Phase 5: 핵심 엔진 분석](#phase-5-핵심-엔진-분석)
- [Phase 6: 고급 기능 분석](#phase-6-고급-기능-분석)
- [Phase 7: 테스트 구조 분석](#phase-7-테스트-구조-분석)
- [Phase 8: 통합 이해](#phase-8-통합-이해)
- [분석 체크리스트](#분석-체크리스트)
- [예상 소요 시간](#예상-소요-시간)

---

## 개요

### 프로젝트 정보

| 항목 | 값 |
|------|-----|
| **이름** | amqplib |
| **설명** | Node.js AMQP 0-9-1 클라이언트 라이브러리 |
| **버전** | 0.10.9 |
| **라이선스** | MIT |
| **Node.js** | ≥18.0.0 |
| **GitHub** | https://github.com/amqp-node/amqplib |

### 아키텍처 레이어

```
┌─────────────────────────────────────────────────────────┐
│  [Layer 5] Public API                                   │
│  channel_api.js, callback_api.js                        │
├─────────────────────────────────────────────────────────┤
│  [Layer 4] Model Layer                                  │
│  lib/channel_model.js, lib/callback_model.js            │
├─────────────────────────────────────────────────────────┤
│  [Layer 3] Channel & Connection                         │
│  lib/channel.js, lib/connection.js                      │
├─────────────────────────────────────────────────────────┤
│  [Layer 2] Protocol Layer                               │
│  lib/codec.js, lib/frame.js, lib/defs.js               │
├─────────────────────────────────────────────────────────┤
│  [Layer 1] Transport Layer                              │
│  lib/connect.js, lib/mux.js, lib/heartbeat.js          │
└─────────────────────────────────────────────────────────┘
```

### 파일 구조 개요

```
amqplib/
├── channel_api.js          # Promise API 진입점
├── callback_api.js         # Callback API 진입점
├── lib/                    # 핵심 구현 (15개 파일)
│   ├── connect.js          # 연결 초기화
│   ├── connection.js       # 연결 관리
│   ├── channel.js          # 채널 기본 클래스
│   ├── channel_model.js    # Promise 채널 래퍼
│   ├── callback_model.js   # Callback 채널 래퍼
│   ├── codec.js            # AMQP 인코딩/디코딩
│   ├── frame.js            # 프레임 파싱
│   ├── defs.js             # [생성됨] 프로토콜 정의
│   ├── api_args.js         # API 인자 변환
│   ├── mux.js              # 스트림 멀티플렉싱
│   ├── heartbeat.js        # 하트비트 관리
│   ├── bitset.js           # 비트셋 유틸리티
│   ├── credentials.js      # 인증 메커니즘
│   ├── error.js            # 커스텀 에러
│   └── format.js           # 디버깅 유틸리티
├── test/                   # 테스트 (11개 파일)
├── examples/               # 사용 예제
└── bin/                    # 빌드 도구
```

---

## 사전 준비

### 필수 배경 지식

1. **Node.js 기초**
   - EventEmitter 패턴
   - Streams (Duplex, PassThrough)
   - Buffer 조작
   - Promise/async-await vs Callback 패턴

2. **AMQP 0-9-1 프로토콜 기초**
   - Connection → Channel → Exchange → Queue 구조
   - Publish/Subscribe 패턴
   - Acknowledgement 개념

### 권장 학습 자료

- [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)
- [AMQP 0-9-1 Complete Reference](https://www.rabbitmq.com/amqp-0-9-1-reference.html)
- [Node.js Streams 문서](https://nodejs.org/api/stream.html)

### 환경 설정

```bash
# RabbitMQ 실행 (Docker)
docker run -d --name amqp.test -p 5672:5672 rabbitmq

# 의존성 설치
npm install

# 테스트 실행 확인
npm test
```

---

## Phase 1: 프로젝트 오리엔테이션

> **목표**: 프로젝트 구조와 빌드 시스템 이해
> **예상 시간**: 30분
> **난이도**: ⭐

### 분석 파일

| 순서 | 파일 | 목적 | LOC |
|------|------|------|-----|
| 1-1 | `package.json` | 의존성, 스크립트, 메타데이터 | 38 |
| 1-2 | `README.md` | 프로젝트 개요, 사용법 | 163 |
| 1-3 | `Makefile` | 빌드 타겟, 테스트 명령 | ~50 |

### 학습 포인트

- [ ] 외부 의존성 2개 파악 (`buffer-more-ints`, `url-parse`)
- [ ] 개발 의존성 역할 이해 (mocha, nyc, biome)
- [ ] `main` 진입점 확인 (`channel_api.js`)
- [ ] npm 스크립트 구조 파악

### 질문하며 읽기

1. 왜 외부 의존성을 최소화했을까?
2. Promise API가 기본인 이유는?
3. 테스트에 RabbitMQ가 필요한 이유는?

---

## Phase 2: 사용자 관점 이해

> **목표**: 라이브러리 사용 방법 익히기
> **예상 시간**: 1시간
> **난이도**: ⭐

### 분석 파일

| 순서 | 파일 | 목적 |
|------|------|------|
| 2-1 | `examples/tutorials/send.js` | 기본 메시지 발송 |
| 2-2 | `examples/tutorials/receive.js` | 기본 메시지 수신 |
| 2-3 | `examples/tutorials/worker.js` | 작업 큐 패턴 |
| 2-4 | `examples/tutorials/emit_log.js` | Pub/Sub 패턴 |
| 2-5 | `examples/tutorials/rpc_client.js` | RPC 패턴 |

### 학습 포인트

- [ ] `connect()` → `createChannel()` → `assertQueue()` 흐름
- [ ] `publish()` vs `sendToQueue()` 차이
- [ ] `consume()` 콜백 구조
- [ ] `ack()`, `nack()` 사용법

### 실습

```bash
# 터미널 1: 수신자
node examples/tutorials/receive.js

# 터미널 2: 발송자
node examples/tutorials/send.js
```

---

## Phase 3: 진입점 분석

> **목표**: Public API 구조 이해
> **예상 시간**: 1시간
> **난이도**: ⭐⭐

### 분석 파일

| 순서 | 파일 | 목적 | LOC |
|------|------|------|-----|
| 3-1 | `channel_api.js` | Promise API 진입점 | ~30 |
| 3-2 | `callback_api.js` | Callback API 진입점 | ~30 |
| 3-3 | `lib/credentials.js` | 인증 메커니즘 | 44 |
| 3-4 | `lib/error.js` | 커스텀 에러 타입 | 23 |

### 핵심 코드 분석

```javascript
// channel_api.js 구조
module.exports = {
  connect: require('./lib/connect').connect,    // 연결 함수
  credentials: require('./lib/credentials'),    // 인증
  IllegalOperationError: require('./lib/error') // 에러
};
```

### 학습 포인트

- [ ] `connect()` 함수가 어디서 오는지 추적
- [ ] Promise API와 Callback API의 차이점
- [ ] 세 가지 인증 방식: `plain`, `amqplain`, `external`
- [ ] `IllegalOperationError` 사용 시점

### 다이어그램

```
사용자 코드
    │
    ▼
channel_api.js ─────────────────────────┐
    │                                   │
    ├── connect() ◄─── lib/connect.js   │
    ├── credentials ◄─ lib/credentials.js
    └── IllegalOperationError ◄─ lib/error.js
```

---

## Phase 4: 프로토콜 계층 이해

> **목표**: AMQP 바이너리 프로토콜 이해
> **예상 시간**: 2-3시간
> **난이도**: ⭐⭐⭐

### 분석 파일

| 순서 | 파일 | 목적 | LOC | 중요도 |
|------|------|------|-----|--------|
| 4-1 | `lib/frame.js` | 프레임 구조 | 181 | 높음 |
| 4-2 | `lib/codec.js` | 인코딩/디코딩 | 376 | 높음 |
| 4-3 | `lib/defs.js` | 프로토콜 정의 | 5,709 | 참조용 |
| 4-4 | `lib/format.js` | 디버깅 유틸 | 31 | 낮음 |

### ⚠️ 주의사항

> `lib/defs.js`는 **자동 생성된 파일**입니다.
> 전체를 읽지 말고 구조만 파악하세요.

### AMQP 프레임 구조

```
┌─────────┬─────────┬──────────┬─────────────┬───────────┐
│  Type   │ Channel │   Size   │   Payload   │ Frame-End │
│ (1byte) │ (2byte) │ (4byte)  │  (variable) │  (1byte)  │
└─────────┴─────────┴──────────┴─────────────┴───────────┘
```

### 학습 포인트

- [ ] 4가지 프레임 타입: METHOD, HEADER, BODY, HEARTBEAT
- [ ] `parseFrame()` 함수 동작 원리
- [ ] Field Table 인코딩 방식 (키-값 쌍)
- [ ] 다양한 데이터 타입: shortstr, longstr, octet, short, long

### 핵심 함수 분석

```javascript
// frame.js - 프레임 파싱
function parseFrame(bin, max) {
  // 1. 헤더 읽기 (7바이트)
  // 2. 페이로드 크기 확인
  // 3. 프레임 끝 마커 검증 (0xce)
  // 4. 타입별 디코딩
}

// codec.js - 필드 테이블
function encodeTable(obj, buffer, offset) {
  // JavaScript 객체 → AMQP 바이너리
}

function decodeTable(buffer) {
  // AMQP 바이너리 → JavaScript 객체
}
```

---

## Phase 5: 핵심 엔진 분석

> **목표**: 연결과 채널 관리 이해
> **예상 시간**: 3-4시간
> **난이도**: ⭐⭐⭐⭐

### 분석 파일

| 순서 | 파일 | 목적 | LOC | 중요도 |
|------|------|------|-----|--------|
| 5-1 | `lib/connect.js` | 연결 초기화 | 187 | 높음 |
| 5-2 | `lib/connection.js` | 연결 관리 | 644 | **최고** |
| 5-3 | `lib/channel.js` | 채널 기본 클래스 | 476 | **최고** |
| 5-4 | `lib/mux.js` | 스트림 멀티플렉싱 | 125 | 중간 |
| 5-5 | `lib/heartbeat.js` | 하트비트 | 91 | 중간 |
| 5-6 | `lib/bitset.js` | 채널 번호 관리 | 146 | 낮음 |

### 권장 분석 순서

```
1. connect.js (진입점)
       │
       ▼
2. connection.js (핵심)
       │
       ├── mux.js (보조)
       ├── heartbeat.js (보조)
       └── bitset.js (보조)
       │
       ▼
3. channel.js (핵심)
```

### 연결 핸드셰이크 시퀀스

```
Client                              Server
   │                                   │
   │──── Protocol Header ─────────────▶│
   │◀──────── Connection.Start ────────│
   │──── Connection.Start-OK ─────────▶│
   │◀──────── Connection.Tune ─────────│
   │──── Connection.Tune-OK ──────────▶│
   │──── Connection.Open ─────────────▶│
   │◀──────── Connection.Open-OK ──────│
   │                                   │
   │         [연결 완료]                │
```

### 학습 포인트

- [ ] URL 파싱 및 소켓 생성 (`connect.js`)
- [ ] 연결 상태 머신 (`connection.js`)
- [ ] 채널 멀티플렉싱 원리
- [ ] RPC 버퍼링 (한 번에 하나의 RPC)
- [ ] 하트비트 타이밍 전략

### 핵심 코드 분석

```javascript
// connection.js - 핵심 상태 머신
Connection.prototype.open = function(allFields, callback) {
  // 1. 프로토콜 헤더 전송
  // 2. Connection.Start 수신 대기
  // 3. 인증 정보 전송
  // 4. Tune 파라미터 협상
  // 5. Connection.Open 전송
  // 6. 완료 콜백 호출
};

// channel.js - RPC 패턴
Channel.prototype._rpc = function(method, fields, expect, callback) {
  // 1. 응답 대기 중이면 큐에 추가
  // 2. 아니면 즉시 전송
  // 3. 응답 도착 시 콜백 호출
};
```

---

## Phase 6: 고급 기능 분석

> **목표**: 고수준 API와 인자 처리 이해
> **예상 시간**: 2시간
> **난이도**: ⭐⭐⭐

### 분석 파일

| 순서 | 파일 | 목적 | LOC |
|------|------|------|-----|
| 6-1 | `lib/channel_model.js` | Promise 래퍼 | 269 |
| 6-2 | `lib/callback_model.js` | Callback 래퍼 | 296 |
| 6-3 | `lib/api_args.js` | 인자 변환 | 321 |

### API 계층 구조

```
사용자 호출
    │
    ▼
channel_model.js (Promise)
callback_model.js (Callback)
    │
    ├── api_args.js (인자 정규화)
    │
    ▼
channel.js (기본 클래스)
    │
    ▼
connection.js (전송)
```

### 학습 포인트

- [ ] `ChannelModel` vs `Channel` 클래스 관계
- [ ] `ConfirmChannel` 구현 방식
- [ ] 인자 기본값 처리 (`api_args.js`)
- [ ] RabbitMQ 확장 옵션 (`x-*` 인자)

### 주요 API 메소드 추적

```javascript
// 예: assertQueue 호출 흐름
ch.assertQueue('myqueue', { durable: true })
    │
    ▼
channel_model.js: Channel.prototype.assertQueue
    │
    ▼
api_args.js: Args.assertQueue (인자 정규화)
    │
    ▼
channel.js: BaseChannel._rpc
    │
    ▼
connection.js: sendMethod
```

---

## Phase 7: 테스트 구조 분석

> **목표**: 테스트 전략과 패턴 이해
> **예상 시간**: 1-2시간
> **난이도**: ⭐⭐

### 분석 파일

| 순서 | 파일 | 목적 | LOC |
|------|------|------|-----|
| 7-1 | `test/util.js` | 테스트 유틸리티 | 214 |
| 7-2 | `test/codec.js` | 인코딩 테스트 | 298 |
| 7-3 | `test/frame.js` | 프레임 테스트 | 232 |
| 7-4 | `test/connection.js` | 연결 테스트 | 432 |
| 7-5 | `test/channel.js` | 채널 테스트 | 504 |
| 7-6 | `test/channel_api.js` | API 테스트 | 639 |

### 테스트 구조 이해

```javascript
// Mocha TDD 스타일
suite('Channel', function() {
  test('open channel', function(done) {
    // 테스트 코드
  });
});
```

### 학습 포인트

- [ ] 단위 테스트 vs 통합 테스트 구분
- [ ] 모킹 전략 (실제 RabbitMQ 필요)
- [ ] 프로퍼티 기반 테스트 (`claire.js` 사용)
- [ ] 비동기 테스트 패턴

### 테스트 실행

```bash
# 전체 테스트
npm test

# 커버리지 리포트
make coverage

# 특정 테스트 파일
npx mocha test/codec.js --ui tdd
```

---

## Phase 8: 통합 이해

> **목표**: 전체 흐름 종합 이해
> **예상 시간**: 1-2시간
> **난이도**: ⭐⭐⭐⭐

### 종합 분석

이 단계에서는 지금까지 분석한 내용을 종합하여 전체 흐름을 이해합니다.

### 전체 데이터 흐름

```
┌──────────────────────────────────────────────────────────────┐
│                        사용자 코드                            │
│  const conn = await amqplib.connect('amqp://localhost');     │
│  const ch = await conn.createChannel();                       │
│  await ch.assertQueue('tasks');                               │
│  ch.sendToQueue('tasks', Buffer.from('hello'));              │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                     channel_api.js                            │
│  connect() 호출 → lib/connect.js로 위임                       │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                      lib/connect.js                           │
│  1. URL 파싱 (url-parse)                                      │
│  2. 소켓 생성 (net/tls)                                       │
│  3. Connection 객체 생성                                      │
│  4. 핸드셰이크 수행                                           │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                    lib/connection.js                          │
│  1. 프로토콜 헤더 전송                                        │
│  2. Start/Start-OK 교환                                       │
│  3. Tune 파라미터 협상                                        │
│  4. Connection.Open/Open-OK                                   │
│  5. ChannelModel 반환                                         │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                   lib/channel_model.js                        │
│  createChannel() → Channel.Open RPC                           │
│  assertQueue() → api_args 변환 → Queue.Declare RPC           │
│  sendToQueue() → Basic.Publish + 메시지 프레임                │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                      lib/channel.js                           │
│  _rpc() → 응답 대기 또는 큐잉                                 │
│  sendMessage() → 헤더 + 바디 프레임 생성                      │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                  lib/codec.js + lib/frame.js                  │
│  JavaScript 객체 → AMQP 바이너리 프레임                       │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                       lib/mux.js                              │
│  여러 채널의 프레임 → 단일 소켓으로 멀티플렉싱                 │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                     TCP/TLS Socket                            │
│                  → RabbitMQ Server                            │
└──────────────────────────────────────────────────────────────┘
```

### 핵심 설계 결정 분석

| 결정 | 이유 | 파일 |
|------|------|------|
| 듀얼 API | 하위 호환성 + 현대적 패턴 | `*_api.js`, `*_model.js` |
| 코드 생성 | AMQP 스펙 변경 대응 용이 | `bin/generate-defs.js` |
| 단일 소켓 멀티플렉싱 | 리소스 효율성 | `mux.js` |
| 순차 RPC | AMQP 스펙 준수 | `channel.js` |
| 최소 의존성 | 유지보수성, 보안 | `package.json` |

---

## 분석 체크리스트

### Phase 1: 프로젝트 오리엔테이션
- [ ] package.json 의존성 파악
- [ ] 빌드 시스템 이해
- [ ] 진입점 확인

### Phase 2: 사용자 관점
- [ ] 기본 예제 실행
- [ ] API 사용 패턴 익히기
- [ ] Pub/Sub, RPC 패턴 이해

### Phase 3: 진입점 분석
- [ ] 두 API 차이점 이해
- [ ] 인증 메커니즘 파악
- [ ] 에러 처리 방식 이해

### Phase 4: 프로토콜 계층
- [ ] 프레임 구조 이해
- [ ] 인코딩/디코딩 로직 파악
- [ ] 데이터 타입 매핑 이해

### Phase 5: 핵심 엔진
- [ ] 연결 핸드셰이크 시퀀스
- [ ] 채널 상태 머신
- [ ] RPC 버퍼링 메커니즘
- [ ] 하트비트 전략

### Phase 6: 고급 기능
- [ ] Promise 래퍼 구현
- [ ] 인자 정규화 로직
- [ ] ConfirmChannel 동작

### Phase 7: 테스트 구조
- [ ] 테스트 프레임워크 이해
- [ ] 단위/통합 테스트 구분
- [ ] 커버리지 확인

### Phase 8: 통합 이해
- [ ] 전체 데이터 흐름 추적
- [ ] 설계 결정 이해
- [ ] 개선점 도출

---

## 예상 소요 시간

| Phase | 예상 시간 | 난이도 | 중요도 |
|-------|----------|--------|--------|
| Phase 1 | 30분 | ⭐ | 필수 |
| Phase 2 | 1시간 | ⭐ | 필수 |
| Phase 3 | 1시간 | ⭐⭐ | 필수 |
| Phase 4 | 2-3시간 | ⭐⭐⭐ | 높음 |
| Phase 5 | 3-4시간 | ⭐⭐⭐⭐ | **최고** |
| Phase 6 | 2시간 | ⭐⭐⭐ | 높음 |
| Phase 7 | 1-2시간 | ⭐⭐ | 중간 |
| Phase 8 | 1-2시간 | ⭐⭐⭐⭐ | 높음 |

**총 예상 시간: 12-16시간** (깊이에 따라 조절)

---

## 부록: 빠른 참조

### 핵심 클래스 관계

```
Connection (lib/connection.js)
    └── ChannelModel (lib/channel_model.js)
            └── Channel (lib/channel.js의 확장)
                    ├── assertQueue()
                    ├── publish()
                    ├── consume()
                    └── ack()
```

### 주요 이벤트

| 이벤트 | 발생 위치 | 의미 |
|--------|----------|------|
| `close` | Connection, Channel | 정상 종료 |
| `error` | Connection, Channel | 에러 발생 |
| `blocked` | Connection | RabbitMQ 리소스 부족 |
| `unblocked` | Connection | 리소스 해제 |

### 파일 크기 순위

| 순위 | 파일 | LOC | 비고 |
|------|------|-----|------|
| 1 | lib/defs.js | 5,709 | 생성된 파일 |
| 2 | lib/connection.js | 644 | 핵심 |
| 3 | lib/channel.js | 476 | 핵심 |
| 4 | lib/codec.js | 376 | 프로토콜 |
| 5 | lib/api_args.js | 321 | API 지원 |

---

> **팁**: 분석하면서 궁금한 점이나 이해하기 어려운 부분이 있으면 해당 파일의 테스트 코드를 함께 보세요. 테스트 코드는 그 모듈이 어떻게 사용되는지 보여주는 가장 좋은 예제입니다.
