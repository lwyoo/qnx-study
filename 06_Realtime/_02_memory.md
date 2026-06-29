# QNX 메모리 관리 (Memory Management)

## 1. QNX 메모리 구조

### 프로세스 메모리 레이아웃

```
High Address
    ├─ Stack (자동 할당/해제)
    │
    ├─ Heap (동적 할당)
    │
    ├─ BSS (초기화 안 된 전역)
    │
    ├─ Data (초기화된 전역)
    │
    ├─ Text (코드)
    │
    └─ Reserved
Low Address
```

### 메모리 보호

```
QNX의 강력한 메모리 보호:
────────────────────────

Process A          Process B
┌─────────────┐   ┌─────────────┐
│             │   │             │
│ Virtual     │   │ Virtual     │
│ Memory      │   │ Memory      │
└─────┬───────┘   └─────┬───────┘
      │                 │
      └─────────────────┴─────────┐
                                   │
                          ┌────────▼─────────┐
                          │  Physical Memory │
                          │  (MMU 관리)      │
                          └──────────────────┘

특징:
✓ 각 프로세스는 독립적인 메모리 공간
✓ 메모리 보호 (한 프로세스 충돌 ≠ 전체 시스템 다운)
✓ Virtual Memory (스왑 지원)
```

---

## 2. 메모리 할당 기술

### Stack 할당 (빠름, 제한적)

```c
#include <stdio.h>

int main() {
    char buffer[1024];         // Stack 할당
    int array[256];            // Stack 할당
    
    // 장점: 빠름, 자동 해제
    // 단점: 크기 제한, 큰 메모리 할당 불가
    
    return 0;
}
```

### Heap 할당 (느림, 유연함)

```c
#include <stdlib.h>

int main() {
    // malloc: 초기화 안 됨
    int *array = malloc(1024 * sizeof(int));
    
    // calloc: 0으로 초기화
    int *array2 = calloc(1024, sizeof(int));
    
    // realloc: 크기 변경
    int *array3 = realloc(array, 2048 * sizeof(int));
    
    // 명시적 해제 필요!
    free(array);
    free(array2);
    free(array3);
    
    return 0;
}
```

### mmap (메모리 매핑)

```c
#include <sys/mman.h>
#include <fcntl.h>

int main() {
    int fd = open("data.bin", O_RDWR);
    
    // 파일을 메모리에 매핑
    void *addr = mmap(NULL, 
                     1024 * 1024,         // 1MB
                     PROT_READ | PROT_WRITE,
                     MAP_SHARED,
                     fd, 
                     0);
    
    if (addr == MAP_FAILED) {
        perror("mmap");
        return 1;
    }
    
    // addr을 포인터처럼 사용
    int *data = (int *)addr;
    data[0] = 42;
    
    // 정리
    munmap(addr, 1024 * 1024);
    close(fd);
    
    return 0;
}
```

---

## 3. 공유 메모리 (IPC)

### POSIX Shared Memory

```c
#include <sys/mman.h>
#include <fcntl.h>

int main() {
    const char *name = "/shared_mem";
    int fd;
    void *addr;
    size_t size = 1024;
    
    // 공유 메모리 객체 생성
    fd = shm_open(name, O_CREAT | O_RDWR, 0644);
    if (fd < 0) {
        perror("shm_open");
        return 1;
    }
    
    // 크기 설정
    if (ftruncate(fd, size) < 0) {
        perror("ftruncate");
        return 1;
    }
    
    // 메모리 매핑
    addr = mmap(NULL, size, PROT_READ | PROT_WRITE,
               MAP_SHARED, fd, 0);
    
    if (addr == MAP_FAILED) {
        perror("mmap");
        return 1;
    }
    
    // 공유 메모리 사용
    int *shared_data = (int *)addr;
    shared_data[0] = 42;
    
    // 정리
    munmap(addr, size);
    shm_unlink(name);
    close(fd);
    
    return 0;
}
```

### System V IPC

```c
#include <sys/ipc.h>
#include <sys/shm.h>

int main() {
    key_t key = ftok("/tmp", 'a');  // 고유 키 생성
    
    // 공유 메모리 생성 (또는 기존것 사용)
    int shmid = shmget(key, 1024, IPC_CREAT | 0644);
    if (shmid < 0) {
        perror("shmget");
        return 1;
    }
    
    // 메모리 첨부
    void *addr = shmat(shmid, NULL, 0);
    if (addr == (void *)-1) {
        perror("shmat");
        return 1;
    }
    
    // 사용
    int *data = (int *)addr;
    data[0] = 42;
    
    // 분리
    shmdt(addr);
    
    // 삭제 (선택)
    shmctl(shmid, IPC_RMID, NULL);
    
    return 0;
}
```

---

## 4. 메모리 누수 탐지

### Valgrind 사용

```bash
# 메모리 누수 검사
valgrind --leak-check=full \
         --show-leak-kinds=all \
         ./program

# 상세 보고서
valgrind --leak-check=full \
         --log-file=valgrind.log \
         ./program
cat valgrind.log
```

### 수동 메모리 추적

```c
#include <stdio.h>
#include <stdlib.h>

#define ALLOC_TRACK 1

#if ALLOC_TRACK
static int alloc_count = 0;
static int free_count = 0;

void* tracked_malloc(size_t size) {
    void *ptr = malloc(size);
    alloc_count++;
    printf("malloc: %p (count: %d)\n", ptr, alloc_count);
    return ptr;
}

void tracked_free(void *ptr) {
    free(ptr);
    free_count++;
    printf("free: %p (count: %d)\n", ptr, free_count);
}

#define malloc(x) tracked_malloc(x)
#define free(x) tracked_free(x)
#endif

int main() {
    int *a = malloc(100);
    int *b = malloc(200);
    
    free(a);
    free(b);
    
    printf("Final: %d allocations, %d frees\n", 
           alloc_count, free_count);
    
    return 0;
}
```

---

## 5. Real-time 메모리 관리

### 메모리 사전 할당 (Pre-allocation)

```c
#include <stdlib.h>
#include <stdio.h>

int main() {
    // 실시간 작업: 메모리를 미리 할당
    // (실행 중 malloc/free 방지)
    
    struct buffer_pool {
        char *buffers[10];
        int size;
        int index;
    } pool;
    
    // 초기화 (메인 루프 전)
    pool.size = 10;
    pool.index = 0;
    
    for (int i = 0; i < pool.size; i++) {
        pool.buffers[i] = malloc(1024);
    }
    
    // 실시간 루프 (malloc/free 없음!)
    for (int i = 0; i < 1000000; i++) {
        // 미리 할당된 버퍼 사용
        char *buffer = pool.buffers[pool.index % pool.size];
        
        // 작업 수행
        sprintf(buffer, "Task %d", i);
        
        pool.index++;
    }
    
    // 정리
    for (int i = 0; i < pool.size; i++) {
        free(pool.buffers[i]);
    }
    
    return 0;
}
```

### Memory Locking (페이지 잠금)

```c
#include <sys/mman.h>
#include <stdlib.h>

int main() {
    // 메모리 스왑 방지 (실시간 성능 보장)
    
    size_t size = 1024 * 1024;  // 1MB
    void *addr = malloc(size);
    
    // 물리 메모리에 고정
    if (mlock(addr, size) < 0) {
        perror("mlock");
    }
    
    // 작업 수행
    // (swap-out 발생 안 함)
    
    // 해제
    munlock(addr, size);
    free(addr);
    
    return 0;
}
```

---

## 6. 메모리 성능 측정

### 할당/해제 성능 벤치마크

```c
#include <stdio.h>
#include <time.h>
#include <stdlib.h>

int main() {
    struct timespec start, end;
    int iterations = 100000;
    
    clock_gettime(CLOCK_MONOTONIC, &start);
    
    for (int i = 0; i < iterations; i++) {
        void *ptr = malloc(256);
        free(ptr);
    }
    
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    double elapsed = (end.tv_sec - start.tv_sec) +
                     (end.tv_nsec - start.tv_nsec) / 1e9;
    double per_alloc = (elapsed / iterations) * 1e6;
    
    printf("Total: %.3f seconds\n", elapsed);
    printf("Per malloc+free: %.2f microseconds\n", per_alloc);
    
    // 예상 결과: 1-5 microseconds
    
    return 0;
}
```

---

## 7. 가상 메모리 (Virtual Memory)

### 메모리 정보 조회

```c
#include <sys/types.h>
#include <sys/neutrino.h>

int main() {
    struct meminfo info;
    
    if (meminfo(&info) < 0) {
        perror("meminfo");
        return 1;
    }
    
    printf("Total RAM: %lld MB\n", 
           info.memsize / (1024 * 1024));
    printf("Free RAM: %lld MB\n", 
           info.freemem / (1024 * 1024));
    
    return 0;
}

// 또는 Linux 호환 방식:
#include <stdio.h>

void print_memory_usage() {
    FILE *fp = fopen("/proc/meminfo", "r");
    char line[256];
    
    while (fgets(line, sizeof(line), fp)) {
        printf("%s", line);
    }
    
    fclose(fp);
}
```

---

## 8. 자동차 임베디드에서의 메모리 관리

### 안정적인 메모리 사용 패턴

```c
#include <stdio.h>
#include <stdlib.h>

// 1. 고정 크기 버퍼 사용
#define MAX_MESSAGES 100
#define MSG_SIZE 256

typedef struct {
    char data[MSG_SIZE];
    int valid;
} message_t;

static message_t messages[MAX_MESSAGES];
static int msg_count = 0;

// 2. Pool-based allocation
int allocate_message(char **out) {
    if (msg_count >= MAX_MESSAGES) {
        return -1;  // 실패
    }
    
    *out = messages[msg_count].data;
    messages[msg_count].valid = 1;
    msg_count++;
    
    return 0;  // 성공
}

// 3. Predictable cleanup
void cleanup_messages() {
    msg_count = 0;
}

int main() {
    char *msg;
    
    // 초기화: 메모리 할당
    // (할당 실패 처리)
    
    // 메인 루프
    while (1) {
        if (allocate_message(&msg) == 0) {
            sprintf(msg, "Message");
        }
        
        // 작업
    }
    
    // 정리
    cleanup_messages();
    
    return 0;
}
```

---

## 다음 단계

학습 로드맵이 완성되었습니다. 모든 핵심 주제를 학습했습니다.

## 학습 완료 체크리스트

- [x] QNX OS 철학 및 마이크로커널 아키텍처
- [x] SDK 설치 및 개발 환경 구축
- [x] 첫 번째 QNX 프로그램 작성
- [x] 디버깅 도구 (GDB, Momentics)
- [x] Linux vs QNX 아키텍처 비교
- [x] 메시지 기반 IPC
- [x] Channel과 Connection
- [x] Pulse (비동기 신호)
- [x] Daemon 구현
- [x] slog2 로깅
- [x] Resource Manager
- [x] TCP/IP 네트워킹
- [x] 실시간 스케줄링
- [x] 메모리 관리

## 최종 프로젝트 추천

42dot 프로젝트를 위해 다음과 같은 통합 시스템 구현을 권장합니다:

```
Sensor Service (IPC)
    ├─→ Message-based communication
    ├─→ Real-time scheduling
    └─→ High priority (우선순위 100)

Gateway Service (Resource Manager + Network)
    ├─→ TCP/IP socket
    ├─→ slog2 logging
    └─→ Medium priority (우선순위 50)

Daemon (Background service)
    ├─→ Auto-restart capability
    ├─→ Watchdog monitoring
    └─→ Low priority (우선순위 10)
```

이 구조는 다음을 포함합니다:
- ✓ IPC (메시지 패싱)
- ✓ 멀티스레드/프로세스
- ✓ 로깅
- ✓ 리소스 관리자
- ✓ 실시간 스케줄링
- ✓ 네트워킹

성공적인 42dot 프로젝트를 기원합니다!
