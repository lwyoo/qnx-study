# QNX 메시지 기반 IPC (Message Passing)

## 1. QNX Message Passing 개요

### 핵심 개념

QNX의 모든 통신은 **메시지 기반**입니다. 파이프, 소켓이 아닌 메시지 전송으로 프로세스 간 통신을 수행합니다.

```
Client                Kernel                 Server
  │                     │                      │
  │ MsgSend()           │                      │
  ├────────────────────▶│                      │
  │                     ├───────────────▶ Message Queue
  │                     │                      │
  │<── SEND Blocked ────│                      │
  │                     │                      │
  │          Scheduler selects Server          │
  │                     ├─────────────────────▶│
  │                     │                MsgReceive()
  │                     │                      │
  │                     │               Process Request
  │                     │                      │
  │                     │◀────── MsgReply() ───┤
  │◀──── Reply ─────────│                      │
  │      Ready          │                      │
  │                     │                      │
  │      Scheduler selects Client              │
```

### 메시지 기반 IPC의 특징


| 특징          | 설명                      |
| ------------- | ------------------------- |
| 동기식        | 송신자가 응답까지 대기    |
| 단순성        | 파이프/소켓 불필요        |
| 스케줄링      | 커널이 직접 스케줄링 관여 |
| 우선순위 상속 | 우선순위 역전 방지        |
| 실시간성      | 커널 레벨에서 보장        |
| 타임아웃      | 시간 제한 설정 가능       |

---

## 2. 기본 메시지 전송 구조

### MsgSend() - 동기식 메시지 송신

```c
#include <sys/neutrino.h>

int MsgSend(
    int coid,              // Channel Connection ID (채널 연결 ID)
    const void *smsg,      // 송신 메시지 버퍼
    int sbytes,            // 송신 메시지 크기 (바이트)
    void *rmsg,            // 수신 메시지 버퍼 (응답)
    int rbytes             // 수신 메시지 크기
);

// 반환값
// 성공: MsgReply()에서 지정한 반환값
// 실패: -1 (errno 설정)
```

### MsgReceive() - 메시지 수신

```c
#include <sys/neutrino.h>

int MsgReceive(
    int chid,              // Channel ID (채널 ID)
    void *msg,             // 수신 메시지 버퍼
    int bytes,             // 버퍼 크기
    struct _msg_info *info // 송신자 정보
);

// 반환값
// 성공: 송신자의 coid (음수이면 펄스)
// 실패: -1 (errno 설정)
```

### MsgReply() - 메시지 응답

```c
#include <sys/neutrino.h>

int MsgReply(
    int rcvid,             // MsgReceive()의 반환값
    int status,            // 상태 코드 (응답)
    const void *msg,       // 응답 메시지 버퍼
    int bytes              // 응답 메시지 크기
);

// 반환값
// 성공: 0
// 실패: -1 (errno 설정)
```

---

## 3. 기본 Server-Client 예제

### Server (메시지 수신자)

```c
// server.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/neutrino.h>
#include <sys/netmgr.h>

#define SERVER_PATH "/dev/server_channel"

typedef struct {
    int value;
    char msg[64];
} request_t;

typedef struct {
    int result;
    int status;
} response_t;

int main() {
    int chid;
    int rcvid;
    request_t req;
    response_t res;
    struct _msg_info msg_info;
    
    // 1. 채널 생성
    chid = ChannelCreate(0);
    if (chid == -1) {
        perror("ChannelCreate failed");
        return EXIT_FAILURE;
    }
    
    printf("Server: Channel created (chid=%d)\n", chid);
    
    // 2. 메시지 수신 루프
    while (1) {
        rcvid = MsgReceive(chid, &req, sizeof(req), &msg_info);
        
        if (rcvid == -1) {
            perror("MsgReceive failed");
            break;
        }
        
        printf("Server: Received msg from PID %d\n", msg_info.pid);
        printf("Server: Request value=%d, msg='%s'\n", 
               req.value, req.msg);
        
        // 3. 요청 처리
        res.result = req.value * 2;
        res.status = 0;
        
        // 4. 응답 전송
        if (MsgReply(rcvid, EOK, &res, sizeof(res)) == -1) {
            perror("MsgReply failed");
        }
    }
    
    ChannelDestroy(chid);
    return EXIT_SUCCESS;
}
```

### Client (메시지 송신자)

```c
// client.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/neutrino.h>
#include <fcntl.h>

typedef struct {
    int value;
    char msg[64];
} request_t;

typedef struct {
    int result;
    int status;
} response_t;

int main() {
    int coid;
    request_t req;
    response_t res;
    
    // 1. 채널 연결
    coid = ConnectAttach(ND_LOCAL_NODE, 0, 1, 0, 0);
    if (coid == -1) {
        perror("ConnectAttach failed");
        return EXIT_FAILURE;
    }
    
    printf("Client: Connected (coid=%d)\n", coid);
    
    // 2. 요청 메시지 작성
    req.value = 42;
    strcpy(req.msg, "Hello from client");
    
    // 3. 메시지 송신 및 응답 수신 (동기)
    if (MsgSend(coid, &req, sizeof(req), 
                &res, sizeof(res)) == -1) {
        perror("MsgSend failed");
        ConnectDetach(coid);
        return EXIT_FAILURE;
    }
    
    // 4. 응답 처리
    printf("Client: Response received\n");
    printf("Client: Result=%d, Status=%d\n", 
           res.result, res.status);
    
    ConnectDetach(coid);
    return EXIT_SUCCESS;
}
```

### 컴파일 및 실행

```bash
# 컴파일
qcc -o server server.c
qcc -o client client.c

# 터미널 1: 서버 실행
./server
# 출력:
# Server: Channel created (chid=1)
# Server: Received msg from PID 12345
# Server: Request value=42, msg='Hello from client'

# 터미널 2: 클라이언트 실행
./client
# 출력:
# Client: Connected (coid=1)
# Client: Response received
# Client: Result=84, Status=0
```

---

## 4. 메시지 구조 설계

### 고정 크기 메시지

```c
typedef struct {
    uint32_t type;        // 메시지 타입
    uint32_t command;     // 명령어
    uint32_t param1;      // 파라미터
    uint32_t param2;
    char data[256];       // 데이터 영역
} message_t;
```

### 가변 크기 메시지 (IOV 기반)

```c
#include <sys/iomsg.h>

typedef struct {
    uint16_t type;
    uint16_t subtype;
    // 가변 부분
} iomsg_t;

// IOV (I/O Vector) 사용
io_msg_t msg;
iov_t iov[2];

SETIOV(&iov[0], &msg, sizeof(msg));
SETIOV(&iov[1], extra_data, extra_size);

MsgSendv(coid, iov, 2, riov, 1);
```

---

## 5. 오류 처리 및 타임아웃

### 타임아웃 설정

```c
#include <sys/neutrino.h>
#include <time.h>

// 타임아웃을 포함한 메시지 송신
int timeout_ms = 1000;  // 1초
struct timespec ts;

clock_gettime(CLOCK_REALTIME, &ts);
ts.tv_nsec += timeout_ms * 1000000;
if (ts.tv_nsec >= 1000000000) {
    ts.tv_sec++;
    ts.tv_nsec -= 1000000000;
}

// 타임아웃을 설정한 송신
int result = MsgSend(coid, &req, sizeof(req),
                     &res, sizeof(res));

if (result == -1 && errno == ETIMEDOUT) {
    printf("Message send timed out\n");
}
```

### 에러 코드 확인

```c
if (MsgSend(coid, &req, sizeof(req), 
            &res, sizeof(res)) == -1) {
    switch (errno) {
    case EBADF:
        printf("Invalid connection ID\n");
        break;
    case ETIMEDOUT:
        printf("Operation timed out\n");
        break;
    case EAGAIN:
        printf("Server not ready\n");
        break;
    default:
        perror("MsgSend failed");
    }
}
```

---

## 6. 메시지 기반 IPC의 장점

### 실시간 성능

```
Linux IPC:
Process A ──┐
            ├→ Kernel (context switch) → 성능 저하
Process B ──┘

QNX Message Passing:
Process A ──→ Kernel ──→ Immediate scheduling ──→ Process B
                         (커널이 직접 결정)
```

### 우선순위 상속 (Priority Inheritance)

```
High Priority Process A
              │
              ├─→ MsgSend()
              │    (대기)
              │
Low Priority Process B  
     (메시지 처리 중)
              │
              └─→ 자동으로 High Priority로 상승
                  (우선순위 역전 방지)
```

---

## 7. 성능 특성

### 메시지 크기 vs 지연시간

```
메시지 크기    지연시간      처리 시간
──────────────────────────────────────
64 bytes       1-5 μs        1-2 μs
256 bytes      2-8 μs        2-3 μs
1024 bytes     4-15 μs       4-5 μs
4096 bytes     10-40 μs      10-15 μs

특징:
- 매우 낮은 지연시간
- 메시지 크기에 비례 증가
- 신뢰성 보장 (커널 관여)
```

---

## 다음 단계

`_02_channel_connection.md`에서 채널과 연결의 자세한 구조를 학습합니다.
