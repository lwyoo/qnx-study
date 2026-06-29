# QNX 실시간 스케줄링 (Scheduler)

## 1. QNX 스케줄링 개요

### 특징

```
특징                설명
─────────────────────────────────────
선점 불가능        현재 실행 중인 프로세스는 중단 불가
우선순위 기반      1-255 우선순위 (높을수록 높은 우선순위)
마이크로초 단위    결정적 컨텍스트 스위칭
FIFO 큐           같은 우선순위는 FIFO 순서
우선순위 상속      메시지 기반 IPC에서 우선순위 상승
```

### 스케줄링 알고리즘

```
QNX Scheduling (기본):
───────────────────────────────────
우선순위 1  ├─ Process A
            └─ Process B (FIFO)
            
우선순위 2  ├─ Process C
            ├─ Process D
            └─ Process E (FIFO)
            
우선순위 255 └─ Process F

실행 순서: F → E/D/C → B/A
(높은 우선순위부터 실행)
```

---

## 2. 스케줄링 알고리즘 종류

### SCHED_FIFO (First In First Out)

```c
#include <sched.h>

struct sched_param param;
param.sched_priority = 10;

sched_setscheduler(pid, SCHED_FIFO, &param);

// 특징:
// - 선점 불가능
// - 타임 슬라이싱 없음
// - 자신보다 높은 우선순위 프로세스로 양보만 가능
```

### SCHED_RR (Round Robin)

```c
#include <sched.h>

struct sched_param param;
param.sched_priority = 10;

sched_setscheduler(pid, SCHED_RR, &param);

// 특징:
// - 타임 슬라이싱 있음
// - 같은 우선순위 프로세스는 시간 배분
```

### SCHED_OTHER (시분할)

```c
// 일반적인 프로세스에 사용 (낮은 우선순위)
sched_setscheduler(pid, SCHED_OTHER, &param);
```

---

## 3. 우선순위 설정

### 우선순위 범위

```c
#include <sched.h>

int max = sched_get_priority_max(SCHED_FIFO);
int min = sched_get_priority_min(SCHED_FIFO);

printf("Priority range: %d to %d\n", min, max);
// 출력: Priority range: 1 to 255
```

### 프로세스 우선순위 설정

```c
#include <sched.h>
#include <unistd.h>

int main() {
    struct sched_param param;
    pid_t pid = getpid();
    
    // 현재 프로세스 우선순위 조회
    sched_getparam(pid, &param);
    printf("Current priority: %d\n", param.sched_priority);
    
    // 우선순위 변경
    param.sched_priority = 20;  // 높은 우선순위
    
    if (sched_setparam(pid, &param) < 0) {
        perror("sched_setparam");
    }
    
    sched_getparam(pid, &param);
    printf("New priority: %d\n", param.sched_priority);
    
    return 0;
}
```

### 스케줄링 정책 변경

```c
#include <sched.h>

int main() {
    struct sched_param param;
    pid_t pid = getpid();
    
    // SCHED_FIFO로 변경 (우선순위 30)
    param.sched_priority = 30;
    
    if (sched_setscheduler(pid, SCHED_FIFO, &param) < 0) {
        perror("sched_setscheduler");
    }
    
    printf("Now running as SCHED_FIFO, priority 30\n");
    
    return 0;
}
```

---

## 4. 실시간 프로세스 구현

### 주기적 실시간 작업 (Task)

```c
#include <stdio.h>
#include <time.h>
#include <sys/neutrino.h>
#include <sched.h>

#define TASK_PERIOD_US 10000  // 10ms

int main() {
    struct timespec ts;
    unsigned elapsed = 0;
    struct sched_param param;
    int iteration = 0;
    
    // 우선순위 설정
    param.sched_priority = 50;
    sched_setscheduler(0, SCHED_FIFO, &param);
    
    printf("Real-time task started (priority 50)\n");
    
    clock_gettime(CLOCK_MONOTONIC, &ts);
    
    // 주기적 루프
    while (1) {
        iteration++;
        
        // 실제 작업 수행
        printf("Iteration %d at %lu.%06lu\n", 
               iteration, ts.tv_sec, ts.tv_nsec / 1000);
        
        // 다음 주기까지 대기
        ts.tv_nsec += TASK_PERIOD_US * 1000;
        
        if (ts.tv_nsec >= 1000000000) {
            ts.tv_sec++;
            ts.tv_nsec -= 1000000000;
        }
        
        // 절대 시간으로 대기 (CLOCK_NANOSLEEP 권장)
        clock_nanosleep(CLOCK_MONOTONIC, 
                       TIMER_ABSTIME, &ts, NULL);
    }
    
    return 0;
}
```

---

## 5. 우선순위 역전 방지 (Priority Inheritance)

### 문제: 우선순위 역전

```
High Priority Task (우선순위 100)
    │
    └─→ Lock 대기
        │
        └─→ Low Priority Task가 소유 중
                │
                └─→ Medium Priority Task 실행 중 (우선순위 50)
                    (Low Priority Task를 막음)

결과: High Priority Task가 Medium과 Low를 기다림 (역전!)
```

### 해결: Priority Inheritance Protocol

```c
#include <pthread.h>
#include <stdio.h>

pthread_mutex_t mutex;

void* high_priority_task(void* arg) {
    pthread_setschedprio(pthread_self(), 100);
    
    printf("High: Attempting to lock mutex...\n");
    pthread_mutex_lock(&mutex);      // 자동 우선순위 상승 발생!
    printf("High: Got lock\n");
    
    // 임계 영역
    sleep(1);
    
    pthread_mutex_unlock(&mutex);
    return NULL;
}

void* low_priority_task(void* arg) {
    pthread_setschedprio(pthread_self(), 10);
    
    printf("Low: Taking lock...\n");
    pthread_mutex_lock(&mutex);
    printf("Low: Got lock (우선순위 일시 상승!)\n");
    
    sleep(2);  // 임계 영역 점유
    
    pthread_mutex_unlock(&mutex);    // 우선순위 복귀
    return NULL;
}

int main() {
    pthread_t h, l;
    pthread_mutexattr_t attr;
    
    // 상속 프로토콜 활성화
    pthread_mutexattr_init(&attr);
    pthread_mutexattr_setprotocol(&attr, 
                                 PTHREAD_PRIO_INHERIT);
    pthread_mutex_init(&mutex, &attr);
    
    pthread_create(&l, NULL, low_priority_task, NULL);
    sleep(1);
    pthread_create(&h, NULL, high_priority_task, NULL);
    
    pthread_join(h, NULL);
    pthread_join(l, NULL);
    
    return 0;
}
```

---

## 6. 스케줄링 성능 측정

### 컨텍스트 스위칭 지연시간 측정

```c
#include <stdio.h>
#include <time.h>
#include <sys/neutrino.h>
#include <sys/neutrino.h>

int main() {
    struct timespec start, end;
    unsigned iterations = 1000000;
    
    clock_gettime(CLOCK_MONOTONIC, &start);
    
    for (int i = 0; i < iterations; i++) {
        // 컨텍스트 스위칭 강제 (sched_yield)
        sched_yield();
    }
    
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    double elapsed = (end.tv_sec - start.tv_sec) +
                     (end.tv_nsec - start.tv_nsec) / 1e9;
    double per_switch = (elapsed / iterations) * 1e6;
    
    printf("Total: %.3f seconds\n", elapsed);
    printf("Per context switch: %.2f microseconds\n", per_switch);
    
    return 0;
}

// 예상 결과:
// Total: 0.100 seconds
// Per context switch: 0.10 microseconds
```

---

## 7. 실전 예제: 주기적 컨트롤러

### 자동차 센서 모니터링

```c
#include <stdio.h>
#include <time.h>
#include <sys/neutrino.h>
#include <sched.h>

#define SENSOR_PERIOD_MS 50     // 50ms (20Hz)
#define CONTROL_PERIOD_MS 100   // 100ms (10Hz)

typedef struct {
    double temperature;
    double pressure;
} sensor_data_t;

void* sensor_task(void* arg) {
    struct sched_param param;
    param.sched_priority = 100;
    sched_setscheduler(0, SCHED_FIFO, &param);
    
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    
    while (1) {
        // 센서 읽기
        sensor_data_t data;
        data.temperature = 25.5;
        data.pressure = 100.0;
        
        printf("Sensor: T=%.1f, P=%.1f\n", 
               data.temperature, data.pressure);
        
        ts.tv_nsec += SENSOR_PERIOD_MS * 1000000;
        if (ts.tv_nsec >= 1000000000) {
            ts.tv_sec++;
            ts.tv_nsec -= 1000000000;
        }
        
        clock_nanosleep(CLOCK_MONOTONIC, 
                       TIMER_ABSTIME, &ts, NULL);
    }
    return NULL;
}

void* control_task(void* arg) {
    struct sched_param param;
    param.sched_priority = 80;  // 낮은 우선순위
    sched_setscheduler(0, SCHED_FIFO, &param);
    
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    
    while (1) {
        // 제어 로직
        printf("Control: Adjusting actuators\n");
        
        ts.tv_nsec += CONTROL_PERIOD_MS * 1000000;
        if (ts.tv_nsec >= 1000000000) {
            ts.tv_sec++;
            ts.tv_nsec -= 1000000000;
        }
        
        clock_nanosleep(CLOCK_MONOTONIC, 
                       TIMER_ABSTIME, &ts, NULL);
    }
    return NULL;
}

int main() {
    pthread_t sensor_tid, control_tid;
    
    pthread_create(&sensor_tid, NULL, sensor_task, NULL);
    pthread_create(&control_tid, NULL, control_task, NULL);
    
    pthread_join(sensor_tid, NULL);
    pthread_join(control_tid, NULL);
    
    return 0;
}
```

---

## 8. CPU 친화성 (CPU Affinity)

### CPU 할당

```c
#include <sys/neutrino.h>

int main() {
    int cpu_id = 0;  // CPU 0에 할당
    
    // 현재 프로세스를 CPU 0에만 할당
    if (sched_setaffinity(0, sizeof(int), &cpu_id) < 0) {
        perror("sched_setaffinity");
    }
    
    printf("Process bound to CPU %d\n", cpu_id);
    
    return 0;
}
```

---

## 다음 단계

`_02_memory.md`에서 QNX의 메모리 관리를 학습합니다.
