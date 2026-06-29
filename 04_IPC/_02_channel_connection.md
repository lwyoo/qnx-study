# QNX Channel과 Connection 상세 분석

## 1. Channel (채널) 개요

### Channel의 역할

Channel은 서버가 메시지를 받기 위해 생성하는 **수신 지점**입니다.

```
Server Process
    │
    ├─→ ChannelCreate() ──→ chid=1 (채널 ID)
    │                        │
    │                        └─→ 메시지 받을 준비
    │
    └─→ MsgReceive(chid=1) ──→ 대기 상태
         (멈춤)
```

### Channel 생성

```c
#include <sys/neutrino.h>

int ChannelCreate(unsigned flags)
{
    // flags:
    // 0: 기본 채널
    // _NTO_CHF_DISCONNECT: 연결 해제 시 펄스 수신
    // _NTO_CHF_REPLY_UPPERCUT: 응답 우선순위 상승
    // ...
}

// 예제
int chid = ChannelCreate(0);
if (chid == -1) {
    perror("ChannelCreate failed");
}
```

---

## 2. Connection (연결) 개요

### Connection의 역할

Connection은 클라이언트가 서버의 채널에 연결하기 위한 **연결 포인트**입니다.

```
Client Process
    │
    ├─→ ConnectAttach(node, pid, chid, ...)
    │
    └─→ coid (Connection ID) 획득
         │
         └─→ MsgSend(coid)로 메시지 송신 가능
```

### Connection 생성

```c
#include <sys/neutrino.h>

int ConnectAttach(
    unsigned nd,            // 노드 ID (로컬: ND_LOCAL_NODE)
    pid_t pid,              // 서버의 PID
    int chid,               // 서버의 Channel ID
    unsigned index,         // 연결 인덱스 (보통 0)
    int flags               // 플래그
)
{
    // 예제: 로컬 호스트, PID 1000, chid 1에 연결
    int coid = ConnectAttach(ND_LOCAL_NODE, 1000, 1, 0, 0);
    
    if (coid == -1) {
        perror("ConnectAttach failed");
    }
}
```

---

## 3. Channel 구조 상세

### Channel의 내부 구조

```
Channel (chid = 1)
├─ Maximum clients: 최대 연결 수
├─ Current connections: 현재 연결 수
├─ Message queue: 메시지 큐 (비어있음 = MsgReceive 대기 중)
├─ Owner PID: 채널 소유 프로세스
└─ Flags: 채널 옵션
```

### Channel의 상태 조회

```c
#include <sys/neutrino.h>

struct _channel_info {
    pid_t pid;           // 채널 소유자 PID
    unsigned flags;      // 채널 플래그
    unsigned chid;       // 채널 ID
    unsigned numconn;    // 연결된 클라이언트 수
};

struct _channel_info info;
ChannelInfo(chid, &info);

printf("Channel owner PID: %d\n", info.pid);
printf("Connected clients: %d\n", info.numconn);
```

### Channel 상태 모니터링

```c
// 채널에 연결된 모든 연결 정보 조회
struct _connection_info {
    pid_t pid;           // 클라이언트 PID
    pid_t server_pid;    // 서버 PID
    int server_chid;     // 채널 ID
};

struct _connection_info conn_info;
ConnectInfo(coid, &conn_info);
```

---

## 4. Connection 구조 상세

### Connection의 내부 구조

```
Connection (coid = 1)
├─ Client PID: 클라이언트 프로세스 ID
├─ Server PID: 서버 프로세스 ID
├─ Server chid: 서버의 채널 ID
├─ Flags: 연결 옵션
├─ Queue: 대기 중인 메시지
└─ Timeout: 타임아웃 설정
```

### Connection 정보 조회

```c
#include <sys/neutrino.h>

struct _connection_info {
    pid_t pid;           // 클라이언트 PID
    pid_t server_pid;    // 서버 PID
    int server_chid;     // 채널 ID
};

struct _connection_info info;
ConnectInfo(coid, &info);

printf("Client PID: %d\n", getpid());
printf("Server PID: %d\n", info.server_pid);
printf("Server chid: %d\n", info.server_chid);
```

---

## 5. 다중 Channel 서버

### 다중 Channel 설계

```c
// 서버가 여러 채널을 통해 다양한 클라이언트 처리

Server Process
    │
    ├─→ ChannelCreate() ──→ chid=1 (UI 채널)
    ├─→ ChannelCreate() ──→ chid=2 (제어 채널)
    └─→ ChannelCreate() ──→ chid=3 (진단 채널)

Client A ──→ chid=1 (UI 요청)
Client B ──→ chid=2 (제어 요청)
Client C ──→ chid=3 (진단 요청)
```

### 다중 Channel 예제

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/neutrino.h>
#include <sys/select.h>

#define CONTROL_CHANNEL 0
#define DATA_CHANNEL    1
#define NUM_CHANNELS    2

typedef struct {
    uint32_t type;
    uint32_t data;
} message_t;

int main() {
    int chid[NUM_CHANNELS];
    int rcvid;
    message_t msg;
    struct _msg_info info;
    fd_set rfds;
    int i;
    
    // 채널 생성
    for (i = 0; i < NUM_CHANNELS; i++) {
        chid[i] = ChannelCreate(0);
        printf("Channel %d created: %d\n", i, chid[i]);
    }
    
    // select()로 여러 채널 모니터링
    while (1) {
        FD_ZERO(&rfds);
        
        // 모든 채널을 select에 등록
        for (i = 0; i < NUM_CHANNELS; i++) {
            FD_SET(chid[i], &rfds);
        }
        
        // select()로 메시지 수신 준비 확인
        // (표준 select()에서는 chid를 직접 사용 불가)
        // 대신 각 채널별로 MsgReceive() 호출
        
        for (i = 0; i < NUM_CHANNELS; i++) {
            rcvid = MsgReceive(chid[i], &msg, sizeof(msg), 
                              &info);
            
            if (rcvid != -1) {
                printf("Channel %d: received from PID %d\n", 
                       i, info.pid);
                
                // 채널별 처리
                switch (i) {
                case CONTROL_CHANNEL:
                    printf("  Control command: %d\n", msg.data);
                    break;
                case DATA_CHANNEL:
                    printf("  Data: %d\n", msg.data);
                    break;
                }
                
                // 응답
                MsgReply(rcvid, EOK, &msg, sizeof(msg));
            }
        }
    }
    
    return 0;
}
```

---

## 6. Channel 연결 해제

### Connection 해제

```c
#include <sys/neutrino.h>

// 클라이언트에서 연결 해제
int ConnectDetach(int coid)
{
    // coid를 더 이상 사용할 수 없음
    // 해당 연결의 자원 반환
}

// 예제
int coid = ConnectAttach(...);
// ... 메시지 송수신 ...
ConnectDetach(coid);
```

### Channel 제거

```c
#include <sys/neutrino.h>

// 서버에서 채널 제거
int ChannelDestroy(int chid)
{
    // 채널의 모든 연결 강제 종료
    // 대기 중인 MsgReceive() 깨우기
}

// 예제
int chid = ChannelCreate(0);
// ... 메시지 처리 ...
ChannelDestroy(chid);
```

---

## 7. Named Channel (이름 기반 채널)

### Resmgr (Resource Manager)를 통한 Named Channel

```c
#include <sys/neutrino.h>
#include <sys/resmgr.h>

// 채널을 파일 시스템에 등록
void register_named_channel(const char *path, int chid) {
    int fd;
    
    // 추가 학습: 05_System/_03_resource_manager.md
}

// 클라이언트에서 이름으로 연결
int connect_named_channel(const char *path) {
    // open()으로 파일을 열면 내부적으로 연결됨
    // ioctl()로 메시지 송수신
}
```

---

## 8. Channel과 Priority (우선순위)

### 우선순위 기반 메시지 처리

```
High Priority Client A ──┐
                         ├─→ 메시지 큐
                         │   (우선순위 순 정렬)
Low Priority Client B ──┘

Server의 MsgReceive()는 우선순위가 높은 메시지부터 받음
```

### 우선순위 설정

```c
#include <sys/neutrino.h>
#include <sys/resource.h>

// 프로세스 우선순위 설정
int priority = 10;
int policy = SCHED_FIFO;

int pid = getpid();
setpriority(PRIO_PROCESS, pid, priority);
```

---

## 9. 실전 예제: Echo Server

### Echo Server 구현

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/neutrino.h>

#define MAX_MSG_SIZE 256

typedef struct {
    char data[MAX_MSG_SIZE];
    int len;
} echo_msg_t;

int main() {
    int chid, rcvid;
    echo_msg_t msg;
    struct _msg_info info;
    
    // 채널 생성
    chid = ChannelCreate(0);
    printf("Echo Server: Channel %d created\n", chid);
    printf("Echo Server: Waiting for messages...\n");
    
    // 메시지 처리 루프
    while (1) {
        rcvid = MsgReceive(chid, &msg, sizeof(msg), &info);
        
        if (rcvid == -1) {
            perror("MsgReceive");
            continue;
        }
        
        // 클라이언트로부터 받은 메시지 그대로 반환
        printf("Echo Server: Received '%s' from PID %d\n", 
               msg.data, info.pid);
        
        // 응답 (동일한 메시지)
        if (MsgReply(rcvid, EOK, &msg, msg.len) == -1) {
            perror("MsgReply");
        }
    }
    
    ChannelDestroy(chid);
    return 0;
}
```

### Echo Client 구현

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/neutrino.h>

#define MAX_MSG_SIZE 256

typedef struct {
    char data[MAX_MSG_SIZE];
    int len;
} echo_msg_t;

int main(int argc, char *argv[]) {
    int coid;
    echo_msg_t req, res;
    
    if (argc < 2) {
        printf("Usage: %s <message>\n", argv[0]);
        return EXIT_FAILURE;
    }
    
    // 서버에 연결
    coid = ConnectAttach(ND_LOCAL_NODE, 0, 1, 0, 0);
    if (coid == -1) {
        perror("ConnectAttach");
        return EXIT_FAILURE;
    }
    
    // 메시지 준비
    strcpy(req.data, argv[1]);
    req.len = strlen(req.data) + 1;
    
    // 서버로 송신
    if (MsgSend(coid, &req, sizeof(req), 
                &res, sizeof(res)) == -1) {
        perror("MsgSend");
        ConnectDetach(coid);
        return EXIT_FAILURE;
    }
    
    // 응답 출력
    printf("Echo Client: Received echo: '%s'\n", res.data);
    
    ConnectDetach(coid);
    return EXIT_SUCCESS;
}
```

---

## 다음 단계

`_03_pulse.md`에서 비동기 신호 메커니즘인 Pulse를 학습합니다.
