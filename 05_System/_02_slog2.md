# QNX slog2 (고급 로깅 시스템)

## 1. slog2 개요

### slog2 vs syslog

| 특징 | syslog | slog2 |
|------|--------|-------|
| **구조** | 텍스트 기반 | 바이너리 구조화 |
| **성능** | 낮은 오버헤드 | 매우 높은 성능 |
| **저장** | 파일 시스템 | 순환 버퍼 |
| **검색** | grep 필요 | 구조화된 필터 |
| **실시간** | 제한적 | 완벽한 실시간 |
| **QNX 최적화** | 아님 | 최고 최적화 |

### slog2의 특징

```
특징                설명
─────────────────────────────────────
구조화된 로그      필드 기반 로그
순환 버퍼          고정 메모리 사용
실시간 필터링      성능 손실 없음
다중 버퍼          동시성 지원
낮은 오버헤드      < 1 μs per log
```

---

## 2. slog2 기본 구조

### slog2 아키텍처

```
Application Process
    │
    ├─→ slog2_printf()
    │
    └─→ Shared Memory (slog2 buffer)
                │
                ├─→ slogger2 daemon (수집)
                │
                └─→ /dev/slog2
                     (또는 파일)
```

### slog2 헤더 포함

```c
#include <sys/slog2.h>

// slog2 완전 사용
#define SLOG2_CODE 1

slog2_info_t info;
slog2_register(&info, NULL, SLOG2_ALLOC_CODE);
```

---

## 3. 기본 slog2 예제

### 간단한 slog2 로깅

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/slog2.h>
#include <stdarg.h>

// slog2 완전 사용을 위한 등록
uint32_t slog2_code = _SLOG_INFO;

int main() {
    // slog2 초기화
    slog2_info_t info;
    
    if (slog2_register(&info, NULL, _SLOG_ALLOC_CODE) < 0) {
        fprintf(stderr, "slog2_register failed\n");
        return EXIT_FAILURE;
    }
    
    // 로그 메시지
    slog2_info(&info, 0, "Application started");
    slog2_info(&info, 0, "Processing %d items", 42);
    slog2_warn(&info, 0, "Low memory: %d%%", 85);
    slog2_error(&info, 0, "Failed to open file: %s", "/etc/config");
    slog2_critical(&info, 0, "Critical failure!");
    
    // 로그 레벨 지정
    slog2(&info, 0, _SLOG_DEBUG, "Debug message");
    slog2(&info, 0, _SLOG_INFO, "Info message");
    slog2(&info, 0, _SLOG_WARNING, "Warning message");
    slog2(&info, 0, _SLOG_ERROR, "Error message");
    
    return EXIT_SUCCESS;
}
```

### 컴파일 및 실행

```bash
# 컴파일
qcc -o slog2_test main.c

# 실행
./slog2_test

# 로그 확인 (실시간)
slogger2 -e &
```

---

## 4. slog2 메시지 포맷

### 로그 레벨

```c
#include <sys/slog2.h>

// 로그 레벨 정의
#define _SLOG_SHUTDOWN  0   // 종료
#define _SLOG_CRITICAL  1   // 심각
#define _SLOG_ERROR     2   // 오류
#define _SLOG_WARNING   3   // 경고
#define _SLOG_NOTICE    4   // 공지
#define _SLOG_INFO      5   // 정보
#define _SLOG_DEBUG     6   // 디버그

// 사용
slog2_c(&info, 0, _SLOG_CRITICAL, "Critical error");
slog2_e(&info, 0, _SLOG_ERROR, "Error occurred");
slog2_w(&info, 0, _SLOG_WARNING, "Warning issued");
slog2_i(&info, 0, _SLOG_INFO, "Information");
slog2_d(&info, 0, _SLOG_DEBUG, "Debug data");
```

### 메시지 포맷팅

```c
#include <sys/slog2.h>
#include <string.h>

int main() {
    slog2_info_t info;
    slog2_register(&info, NULL, _SLOG_ALLOC_CODE);
    
    // 텍스트 메시지
    slog2_info(&info, 0, "Process started");
    
    // 포맷팅된 메시지
    int value = 42;
    slog2_info(&info, 0, "Value is %d", value);
    
    // 여러 필드
    slog2_info(&info, 0, "user=%s pid=%d", "root", 1234);
    
    // 긴 메시지
    char buffer[256];
    snprintf(buffer, sizeof(buffer), 
             "Long message: x=%d, y=%f", 10, 3.14);
    slog2_info(&info, 0, buffer);
    
    return 0;
}
```

---

## 5. slog2 버퍼 관리

### 단일 버퍼 사용

```c
#include <sys/slog2.h>

int main() {
    slog2_info_t info;
    
    // 단일 버퍼 등록
    // info.nentries = 8192 (기본값)
    slog2_register(&info, NULL, _SLOG_ALLOC_CODE);
    
    slog2_info(&info, 0, "Using default buffer");
    
    return 0;
}
```

### 다중 버퍼 사용 (고급)

```c
#include <sys/slog2.h>

int main() {
    slog2_info_t info;
    slog2_buffer_t buffers[3];
    
    // 버퍼 생성
    info.nentries = 2048;
    info.nbuffers = 3;
    info.buffers = buffers;
    
    slog2_register(&info, NULL, _SLOG_ALLOC_CODE);
    
    // 각 버퍼에 로그 (인덱스 0, 1, 2)
    slog2_info(&info, 0, "Buffer 0: info");
    slog2_info(&info, 1, "Buffer 1: debug");
    slog2_info(&info, 2, "Buffer 2: performance");
    
    return 0;
}
```

---

## 6. 구조화된 로깅 (Data Logging)

### 바이너리 데이터 로깅

```c
#include <sys/slog2.h>
#include <sys/slog2_code.h>

typedef struct {
    uint32_t sensor_id;
    int32_t temperature;
    uint16_t humidity;
} sensor_data_t;

int main() {
    slog2_info_t info;
    slog2_register(&info, NULL, _SLOG_ALLOC_CODE);
    
    sensor_data_t data = {
        .sensor_id = 1,
        .temperature = 25,
        .humidity = 60
    };
    
    // 구조화된 데이터 로깅
    slog2_b(&info, 0, _SLOG_INFO, 
            &data, sizeof(data), "sensor_data");
    
    return 0;
}
```

---

## 7. 실시간 로그 모니터링

### slogger2로 로그 읽기

```bash
# 모든 로그 실시간 모니터링
slogger2 -e &

# 특정 프로세스만
slogger2 -p <pid> &

# 파일로 저장
slogger2 -f /var/log/qnx.log &

# 버퍼 크기 지정
slogger2 -b 1024 &
```

### sloginfo로 로그 통계

```bash
# 로그 통계 확인
sloginfo

# 예시 출력:
# slog2 Info:
#   Total of 3 files
#   File1: app1
#   File2: app2
#   File3: app3
```

---

## 8. 성능 측정

### slog2 성능 벤치마크

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <sys/slog2.h>

int main() {
    slog2_info_t info;
    slog2_register(&info, NULL, _SLOG_ALLOC_CODE);
    
    struct timespec start, end;
    int iterations = 100000;
    
    clock_gettime(CLOCK_REALTIME, &start);
    
    for (int i = 0; i < iterations; i++) {
        slog2_info(&info, 0, "Iteration %d", i);
    }
    
    clock_gettime(CLOCK_REALTIME, &end);
    
    double elapsed = (end.tv_sec - start.tv_sec) +
                     (end.tv_nsec - start.tv_nsec) / 1e9;
    double per_log = (elapsed / iterations) * 1e6;
    
    printf("Time: %.3f seconds\n", elapsed);
    printf("Per log: %.2f microseconds\n", per_log);
    
    return 0;
}

// 예상 결과:
// Time: 0.100 seconds
// Per log: 1.00 microseconds
```

---

## 9. Daemon과 slog2 통합

### Daemon에서 slog2 사용

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/slog2.h>

int main() {
    slog2_info_t info;
    slog2_register(&info, NULL, _SLOG_ALLOC_CODE);
    
    // Daemonize (생략)
    pid_t pid = fork();
    if (pid > 0) exit(0);
    
    slog2_info(&info, 0, "Daemon started (PID=%d)", getpid());
    
    // 메인 루프
    while (1) {
        slog2_debug(&info, 0, "Working...");
        sleep(5);
    }
    
    slog2_info(&info, 0, "Daemon stopped");
    
    return 0;
}
```

---

## 다음 단계

`_03_resource_manager.md`에서 QNX의 리소스 관리자를 학습합니다.
