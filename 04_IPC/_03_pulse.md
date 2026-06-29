# QNX Pulse (비동기 신호 메커니즘)

## 1. Pulse 개요

### Pulse vs Message Passing

**Message Passing (동기식):**
```
Client sends → Waits → Server replies → Client continues
                ↑________________↑
            블로킹 대기 (성능 비용)
```

**Pulse (비동기식):**
```
Client sends pulse → Continues immediately
                     ↓
              Server receives asynchronously
              
이점: 논블로킹, 낮은 오버헤드, 신호 기반 알림
```

### Pulse의 특징

```
특징              설명
─────────────────────────────────────────────
경량              작은 데이터 (32바이트)
비동기            송신자가 블로킹되지 않음
우선순위 지원    서버 우선순위에 따라 처리
신호 기반        UNIX 신호와 유사
오버헤드 낮음    메시지보다 빠름
```

---

## 2. Pulse의 구조

### Pulse 데이터 구조

```c
#include <sys/neutrino.h>

// Pulse는 내부적으로 다음 구조:
struct _pulse {
    uint16_t type;      // _PULSE_TYPE (128-255)
    uint16_t subtype;
    int32_t  code;      // 펄스 코드
    union {
        uint32_t value;
        void *ptr;
        int fd;
    } un;
};

// 최대 크기: 32 바이트
```

### MsgSendPulse() - Pulse 송신 (비동기)

```c
#include <sys/neutrino.h>

int MsgSendPulse(
    int coid,           // Connection ID
    int priority,       // 펄스 처리 우선순위
    int code,           // 펄스 코드 (0-255)
    int value           // 펄스 데이터 (32비트)
);

// 반환값
// 성공: 0
// 실패: -1 (errno 설정)

// 예제
int coid = ConnectAttach(ND_LOCAL_NODE, server_pid, chid, 0, 0);

// 펄스 송신 (즉시 반환, 블로킹 X)
if (MsgSendPulse(coid, getpriority(PRIO_PROCESS, 0), 
                 MY_PULSE_CODE, 42) == -1) {
    perror("MsgSendPulse");
}
```

### MsgReceive()에서 Pulse 수신

```c
#include <sys/neutrino.h>

struct _msg_info info;
struct _pulse pulse;

// MsgReceive()는 메시지와 펄스 모두 수신
int rcvid = MsgReceive(chid, &pulse, sizeof(pulse), &info);

if (rcvid < 0) {
    // 펄스 수신 (rcvid < 0)
    if (pulse.type == _PULSE_TYPE) {
        printf("Received pulse: code=%d, value=%d\n", 
               pulse.code, pulse.un.value);
    }
} else {
    // 메시지 수신 (rcvid > 0)
    // MsgReply() 호출 필요
}
```

---

## 3. 기본 Pulse 예제

### Pulse Server

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/neutrino.h>

#define MY_PULSE_CODE 1

int main() {
    int chid;
    int rcvid;
    struct _pulse pulse;
    struct _msg_info info;
    
    // 채널 생성
    chid = ChannelCreate(0);
    printf("Pulse Server: Channel %d created\n", chid);
    printf("Pulse Server: Waiting for pulses...\n");
    
    // Pulse 수신 루프
    while (1) {
        rcvid = MsgReceive(chid, &pulse, sizeof(pulse), &info);
        
        if (rcvid == -1) {
            perror("MsgReceive");
            continue;
        }
        
        if (rcvid < 0) {
            // Pulse 수신 (rcvid < 0)
            printf("Pulse Server: Received pulse\n");
            printf("  Type: %d\n", pulse.type);
            printf("  Code: %d\n", pulse.code);
            printf("  Value: %d\n", pulse.un.value);
            printf("  From PID: %d\n", info.pid);
        } else {
            // Message 수신 (rcvid > 0)
            printf("Pulse Server: Received message (not pulse)\n");
            MsgReply(rcvid, EOK, NULL, 0);
        }
    }
    
    ChannelDestroy(chid);
    return 0;
}
```

### Pulse Client

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/neutrino.h>

#define MY_PULSE_CODE 1

int main(int argc, char *argv[]) {
    int coid;
    pid_t server_pid;
    int i;
    
    if (argc < 2) {
        printf("Usage: %s <server_pid>\n", argv[0]);
        return EXIT_FAILURE;
    }
    
    server_pid = atoi(argv[1]);
    
    // 서버에 연결 (채널 ID는 보통 1)
    coid = ConnectAttach(ND_LOCAL_NODE, server_pid, 1, 0, 0);
    if (coid == -1) {
        perror("ConnectAttach");
        return EXIT_FAILURE;
    }
    
    printf("Pulse Client: Connected to server (PID %d)\n", 
           server_pid);
    
    // 펄스 송신 (논블로킹)
    for (i = 0; i < 5; i++) {
        printf("Pulse Client: Sending pulse #%d\n", i);
        
        if (MsgSendPulse(coid, 
                        getpriority(PRIO_PROCESS, 0),
                        MY_PULSE_CODE, 
                        i) == -1) {
            perror("MsgSendPulse");
        }
        
        sleep(1);
    }
    
    printf("Pulse Client: Done sending pulses\n");
    
    ConnectDetach(coid);
    return EXIT_SUCCESS;
}
```

### 실행 및 출력

```bash
# 터미널 1: 서버 실행
$ ./pulse_server
Pulse Server: Channel 1 created
Pulse Server: Waiting for pulses...
Pulse Server: Received pulse
  Type: 128
  Code: 1
  Value: 0
  From PID: 12345
Pulse Server: Received pulse
  Type: 128
  Code: 1
  Value: 1
  From PID: 12345
...

# 터미널 2: 클라이언트 실행 (PID 1000인 경우)
$ ./pulse_client 1000
Pulse Client: Connected to server (PID 1000)
Pulse Client: Sending pulse #0
Pulse Client: Sending pulse #1
Pulse Client: Sending pulse #2
Pulse Client: Sending pulse #3
Pulse Client: Sending pulse #4
Pulse Client: Done sending pulses
```

---

## 4. Timer Pulse (매우 중요!)

### TimerCreate() + pulse

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <sys/neutrino.h>
#include <sys/siginfo.h>

#define TIMER_PULSE_CODE 100

int main() {
    int chid;
    timer_t timer_id;
    struct _itimer itimer;
    struct _pulse pulse;
    struct _msg_info info;
    int rcvid;
    
    // 채널 생성
    chid = ChannelCreate(0);
    printf("Timer Server: Channel %d created\n", chid);
    
    // 타이머 생성 (매 1초마다 펄스 송신)
    timer_create(CLOCK_MONOTONIC, &event, &timer_id);
    
    itimer.nsec = 1000000000;  // 1초 (나노초)
    itimer.interval_nsec = 1000000000;  // 반복 간격
    
    timer_settime(timer_id, 0, &itimer, NULL);
    
    printf("Timer Server: Timer started (1 sec interval)\n");
    
    // 타이머 펄스 수신 루프
    while (1) {
        rcvid = MsgReceive(chid, &pulse, sizeof(pulse), &info);
        
        if (rcvid < 0 && pulse.code == TIMER_PULSE_CODE) {
            printf("Timer Server: Timer pulse received! ");
            printf("(count=%d)\n", pulse.un.value);
        }
    }
    
    return 0;
}
```

### 더 간단한 타이머 펄스 예제

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/neutrino.h>

int main() {
    int chid, coid;
    struct sigevent event;
    timer_t timer_id;
    struct itimerspec ispec;
    
    chid = ChannelCreate(0);
    coid = ConnectAttach(ND_LOCAL_NODE, 0, chid, 0, 0);
    
    // Pulse 이벤트 설정
    SIGEV_PULSE_INIT(&event, coid, 10, 100);  // 우선순위 10, 코드 100
    
    // 타이머 생성 및 시작
    timer_create(CLOCK_MONOTONIC, &event, &timer_id);
    
    ispec.it_value.tv_sec = 1;
    ispec.it_value.tv_nsec = 0;
    ispec.it_interval.tv_sec = 1;  // 1초마다 반복
    ispec.it_interval.tv_nsec = 0;
    
    timer_settime(timer_id, 0, &ispec, NULL);
    
    printf("Timer started. Ctrl+C to exit\n");
    
    return 0;
}
```

---

## 5. Pulse vs Message 성능 비교

### 벤치마크

```
작업                 Message Pass  Pulse        비율
─────────────────────────────────────────────────
Send 왕복            50 μs         20 μs        2.5배
송신자 블로킹        50 μs         0 μs         무한대
처리 오버헤드        10 μs         5 μs         2배
메모리 사용          256 bytes     32 bytes     8배

결론:
- Pulse: 매우 가볍고, 비동기 신호 용도에 최적
- Message: 동기식 요청/응답이 필요할 때 사용
```

---

## 6. 실전 예제: Event Notification

### Timer + Pulse로 주기적 작업

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <sys/neutrino.h>

typedef struct {
    int counter;
} app_state_t;

int main() {
    int chid, coid;
    timer_t timer_id;
    struct sigevent event;
    struct itimerspec ispec;
    app_state_t state = {0};
    struct _pulse pulse;
    struct _msg_info info;
    int rcvid;
    
    // 채널 생성
    chid = ChannelCreate(0);
    coid = ConnectAttach(ND_LOCAL_NODE, 0, chid, 0, 0);
    
    // Pulse 이벤트 (10Hz)
    SIGEV_PULSE_INIT(&event, coid, 10, 1);
    timer_create(CLOCK_MONOTONIC, &event, &timer_id);
    
    ispec.it_value.tv_sec = 0;
    ispec.it_value.tv_nsec = 100000000;  // 100ms
    ispec.it_interval = ispec.it_value;
    timer_settime(timer_id, 0, &ispec, NULL);
    
    printf("Application: Started (10Hz timer)\n");
    
    // 메인 루프
    while (1) {
        rcvid = MsgReceive(chid, &pulse, sizeof(pulse), &info);
        
        if (rcvid < 0) {
            // 타이머 펄스 처리
            state.counter++;
            
            if (state.counter % 10 == 0) {
                printf("Tick: %d seconds\n", state.counter / 10);
            }
        }
    }
    
    return 0;
}
```

---

## 7. 실무 활용 패턴

### 패턴 1: Heartbeat (수명 신호)

```c
// 서버가 주기적으로 펄스를 송신하여 살아있음을 알림
MsgSendPulse(client_coid, 10, HEARTBEAT_CODE, 0);
```

### 패턴 2: Work Queue (작업 큐)

```c
// 각 작업을 펄스로 큐에 추가
for (int i = 0; i < num_workers; i++) {
    MsgSendPulse(worker_coid[i], 10, WORK_CODE, job_id);
}
```

### 패턴 3: Signal Handler 대체

```c
// 전통적 signal() 대신 pulse 사용
// (더 안정적이고 실시간성 우수)
MsgSendPulse(coid, 10, EVENT_CODE, 0);
```

---

## 다음 단계

시스템 프로그래밍 섹션의 `05_System/_01_daemon.md`에서 데몬 프로세스 구현을 학습합니다.
