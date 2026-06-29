# QNX Resource Manager (리소스 관리자)

## 1. Resource Manager 개요

### 역할

Resource Manager는 QNX에서 하드웨어 장치, 파일 시스템, 네트워크 등의 **리소스를 추상화**하는 핵심 서비스입니다.

```
Client Process (사용자 프로그램)
    │
    ├─→ open("/dev/device")
    │
    └─→ Resource Manager가 처리
         (실제 하드웨어 드라이버나 서비스로 전달)
```

### 특징

```
특징                설명
─────────────────────────────────────
표준 파일 인터페이스  open(), read(), write(), ioctl()
메시지 기반         내부적으로 메시지 전송
분산 구조           여러 프로세스가 관리 가능
핸들 기반           파일 디스크립터로 접근
확장성              새로운 리소스 추가 용이
```

---

## 2. Resource Manager의 계층 구조

```
┌─────────────────────────────────────────┐
│ User Application                        │
│ (open(), read(), write(), ioctl())      │
└────────────────┬────────────────────────┘
                 │ (파일 작업)
┌────────────────▼────────────────────────┐
│ Resource Manager Framework              │
│ (메시지 처리, 콜백 함수)                │
└────────────────┬────────────────────────┘
                 │ (메시지 기반 IPC)
┌────────────────▼────────────────────────┐
│ Actual Resource                         │
│ (Device, File System, Network)          │
└─────────────────────────────────────────┘
```

---

## 3. Resource Manager 기본 구조

### 콜백 함수 (Callback Functions)

```c
#include <sys/resmgr.h>

typedef struct {
    // I/O 함수
    int (*io_open)(resmgr_context_t *ctp, 
                   io_open_t *msg, 
                   RESMGR_HANDLE_T *handle, 
                   void *extra);
    int (*io_read)(resmgr_context_t *ctp, 
                   io_read_t *msg, 
                   RESMGR_HANDLE_T *handle);
    int (*io_write)(resmgr_context_t *ctp, 
                    io_write_t *msg, 
                    RESMGR_HANDLE_T *handle);
    int (*io_close)(resmgr_context_t *ctp, 
                    void *reserved, 
                    RESMGR_HANDLE_T *handle);
} resmgr_io_funcs_t;
```

---

## 4. 간단한 Resource Manager 예제

### 카운터 디바이스 구현

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/resmgr.h>
#include <sys/iofunc.h>

#define DEVICE_PATH "/dev/counter"

// 디바이스 상태
typedef struct {
    iofunc_attr_t attr;
    int counter;
} device_attr_t;

device_attr_t device;

// ===== Callback Functions =====

// 1. open() 호출 시
static int io_open(resmgr_context_t *ctp, 
                   io_open_t *msg, 
                   RESMGR_HANDLE_T *handle, 
                   void *extra) {
    printf("Device opened\n");
    return EOK;
}

// 2. read() 호출 시
static int io_read(resmgr_context_t *ctp, 
                   io_read_t *msg, 
                   RESMGR_HANDLE_T *handle) {
    int nbytes;
    char buffer[64];
    
    // 현재 카운터 값 읽기
    nbytes = snprintf(buffer, sizeof(buffer), 
                      "Counter: %d\n", device.counter);
    
    // 클라이언트에 데이터 전송
    if (nbytes > msg->i.nbytes) {
        nbytes = msg->i.nbytes;
    }
    
    memcpy(msg->o.io_union_t, buffer, nbytes);
    msg->o.ret_val = nbytes;
    
    printf("Device read: %s", buffer);
    
    return _RESMGR_NPARTS(1);
}

// 3. write() 호출 시
static int io_write(resmgr_context_t *ctp, 
                    io_write_t *msg, 
                    RESMGR_HANDLE_T *handle) {
    int nbytes = msg->i.nbytes;
    char buffer[64];
    
    if (nbytes > sizeof(buffer) - 1) {
        nbytes = sizeof(buffer) - 1;
    }
    
    // 클라이언트로부터 데이터 수신
    memcpy(buffer, msg->i.io_union_t, nbytes);
    buffer[nbytes] = '\0';
    
    // 카운터 업데이트
    int value = atoi(buffer);
    device.counter = value;
    
    msg->o.ret_val = nbytes;
    
    printf("Device write: %s", buffer);
    
    return _RESMGR_NPARTS(0);
}

// 4. close() 호출 시
static int io_close(resmgr_context_t *ctp, 
                    void *reserved, 
                    RESMGR_HANDLE_T *handle) {
    printf("Device closed\n");
    return EOK;
}

// ===== Main =====

int main() {
    dispatch_t *dpp;
    resmgr_attr_t rattr;
    resmgr_connect_funcs_t connect_funcs;
    resmgr_io_funcs_t io_funcs;
    iofunc_attr_t ioattr;
    int id;
    
    printf("Counter Resource Manager started\n");
    
    // 1. Dispatch 생성
    dpp = dispatch_create();
    if (dpp == NULL) {
        perror("dispatch_create");
        return EXIT_FAILURE;
    }
    
    // 2. 콜백 함수 등록
    memset(&connect_funcs, 0, sizeof(connect_funcs));
    memset(&io_funcs, 0, sizeof(io_funcs));
    
    io_funcs.io_open = io_open;
    io_funcs.io_read = io_read;
    io_funcs.io_write = io_write;
    io_funcs.io_close = io_close;
    
    // 3. 리소스 매니저 설정
    memset(&rattr, 0, sizeof(rattr));
    rattr.nparts_max = 1;
    rattr.msg_max_size = 2048;
    
    // 4. 리소스 매니저 등록
    memset(&ioattr, 0, sizeof(ioattr));
    ioattr.mount = NULL;
    ioattr.nlink = 1;
    
    id = resmgr_attach(dpp, &rattr, DEVICE_PATH, 
                       _FTYPE_ANY, 0, 
                       &connect_funcs, &io_funcs, 
                       &ioattr);
    if (id < 0) {
        perror("resmgr_attach");
        return EXIT_FAILURE;
    }
    
    printf("Device registered: %s\n", DEVICE_PATH);
    
    // 5. 메시지 처리 루프
    dispatch_handler(dpp);
    
    return EXIT_SUCCESS;
}
```

### 클라이언트 프로그램

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    int fd;
    char buffer[64];
    int nbytes;
    
    // 디바이스 열기
    fd = open("/dev/counter", O_RDWR);
    if (fd < 0) {
        perror("open");
        return EXIT_FAILURE;
    }
    
    printf("Device opened: %d\n", fd);
    
    // 1. 읽기
    nbytes = read(fd, buffer, sizeof(buffer));
    printf("Read: %s", buffer);
    
    // 2. 쓰기
    write(fd, "42\n", 3);
    
    // 3. 다시 읽기
    nbytes = read(fd, buffer, sizeof(buffer));
    printf("Read: %s", buffer);
    
    // 디바이스 닫기
    close(fd);
    
    return EXIT_SUCCESS;
}
```

### 컴파일 및 실행

```bash
# 리소스 매니저 컴파일
qcc -o counter_resmgr counter_resmgr.c -lasync

# 클라이언트 컴파일
qcc -o counter_client counter_client.c

# 터미널 1: 리소스 매니저 실행
./counter_resmgr &

# 터미널 2: 클라이언트 실행
./counter_client
# 출력:
# Device opened: 3
# Read: Counter: 0
# Read: Counter: 42
```

---

## 5. ioctl을 통한 제어

### ioctl 구현

```c
// Resource Manager에 io_ioctl 콜백 추가
static int io_ioctl(resmgr_context_t *ctp, 
                    io_ioctl_t *msg, 
                    RESMGR_HANDLE_T *handle) {
    switch (msg->i.dcmd) {
    case DCMD_ALL_GETFD:
        // 파일 디스크립터 가져오기
        return iofunc_getfd(ctp, msg, handle);
        
    case 100:  // 카운터 리셋
        device.counter = 0;
        return EOK;
        
    case 101:  // 카운터 증가
        device.counter++;
        return EOK;
        
    case 102:  // 카운터 감소
        device.counter--;
        return EOK;
        
    default:
        return EINVAL;
    }
}
```

### 클라이언트에서 ioctl 사용

```c
#include <sys/ioctl.h>

int fd = open("/dev/counter", O_RDWR);

// 리셋
ioctl(fd, 100, NULL);

// 증가
ioctl(fd, 101, NULL);

// 감소
ioctl(fd, 102, NULL);

close(fd);
```

---

## 6. 연결 및 I/O 함수 분리

### Connect 함수와 I/O 함수

```c
#include <sys/resmgr.h>

// Connect 콜백 (open 전)
static int connect_open(resmgr_context_t *ctp, 
                        io_open_t *msg, 
                        RESMGR_HANDLE_T *handle, 
                        void *extra) {
    printf("Connect: open requested\n");
    return EOK;
}

// I/O 함수들...

int main() {
    resmgr_connect_funcs_t connect_funcs;
    
    memset(&connect_funcs, 0, sizeof(connect_funcs));
    connect_funcs.open = connect_open;
    connect_funcs.close = io_close;
    
    // ... resmgr_attach(...) ...
}
```

---

## 7. 성능 고려사항

### 메모리 할당 최소화

```c
// 좋은 예: 스택에 할당
void io_read_fast(resmgr_context_t *ctp, io_read_t *msg) {
    char buffer[256];  // 스택 할당 (빠름)
    // ...
}

// 나쁜 예: 동적 할당
void io_read_slow(resmgr_context_t *ctp, io_read_t *msg) {
    char *buffer = malloc(256);  // 느림
    // ...
    free(buffer);
}
```

### Context Blocking

```c
// 오래 걸리는 작업은 별도 스레드에서
if (long_operation_needed) {
    // 현재 request는 저장
    // 스레드 풀에서 처리
    queue_work(msg);
    return _RESMGR_CONTINUE;  // 응답 미연기
}
```

---

## 다음 단계

`_04_network.md`에서 TCP/IP 네트워킹을 학습합니다.
