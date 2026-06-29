# QNX Daemon 프로세스 구현

## 1. Daemon이란?

### Daemon의 특징

```
특징                설명
────────────────────────────────────
백그라운드 실행    터미널과 무관하게 실행
부모 프로세스 없음  init (PID 1) 또는 systemd의 자식
독립적인 세션     새로운 세션 생성
표준 스트림 리다이렉트  /dev/null로 연결
신호 처리         특정 신호에만 반응
```

### Daemon vs 일반 프로세스

```
일반 프로세스:
Terminal
    │
    └─→ Process (터미널 종료 시 함께 종료)

Daemon:
System (init/systemd)
    │
    └─→ Daemon (터미널과 무관하게 실행)
```

---

## 2. Daemon 생성 방법

### Step 1: Fork (부모 프로세스 종료)

```c
#include <unistd.h>
#include <stdlib.h>

pid_t pid = fork();

if (pid < 0) {
    perror("fork");
    exit(EXIT_FAILURE);
}

if (pid > 0) {
    // 부모 프로세스 종료
    // (셸이 새로운 프롬프트 표시)
    exit(EXIT_SUCCESS);
}

// 자식 프로세스 계속 실행
```

### Step 2: Session Leader 되기 (setsid)

```c
#include <unistd.h>

// 새로운 세션 생성
// → 터미널 신호로부터 독립
if (setsid() < 0) {
    perror("setsid");
    exit(EXIT_FAILURE);
}

// 이제 SIGHUP, SIGINT 등으로부터 독립됨
```

### Step 3: 다시 Fork (추가 안정성)

```c
pid = fork();

if (pid > 0) {
    exit(EXIT_SUCCESS);  // 세션 리더 종료
}

// 이제 진정한 daemon (세션 리더가 아님)
```

### Step 4: 표준 스트림 리다이렉트

```c
#include <fcntl.h>
#include <unistd.h>

// 표준 입출력을 /dev/null로 리다이렉트
int fd = open("/dev/null", O_RDWR);

dup2(fd, STDIN_FILENO);   // stdin
dup2(fd, STDOUT_FILENO);  // stdout
dup2(fd, STDERR_FILENO);  // stderr

if (fd > 2) {
    close(fd);
}
```

### Step 5: 작업 디렉토리 변경

```c
#include <unistd.h>

// 루트 디렉토리로 변경
// (마운트 해제 시 문제 방지)
if (chdir("/") < 0) {
    perror("chdir");
    exit(EXIT_FAILURE);
}
```

---

## 3. 완전한 Daemon 함수

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <syslog.h>
#include <signal.h>

#define DAEMON_NAME "my_daemon"
#define PID_FILE "/var/run/%s.pid"

void daemonize(void) {
    pid_t pid;
    
    // Step 1: First fork
    pid = fork();
    if (pid < 0) {
        exit(EXIT_FAILURE);
    }
    if (pid > 0) {
        exit(EXIT_SUCCESS);  // 부모 프로세스 종료
    }
    
    // Step 2: 새로운 세션 생성
    if (setsid() < 0) {
        exit(EXIT_FAILURE);
    }
    
    // Step 3: 신호 처리 (SIGHUP 무시)
    signal(SIGHUP, SIG_IGN);
    
    // Step 4: Second fork (추가 안전성)
    pid = fork();
    if (pid < 0) {
        exit(EXIT_FAILURE);
    }
    if (pid > 0) {
        exit(EXIT_SUCCESS);
    }
    
    // Step 5: 작업 디렉토리 변경
    if (chdir("/") < 0) {
        exit(EXIT_FAILURE);
    }
    
    // Step 6: 파일 마스크 설정
    umask(0);
    
    // Step 7: 표준 스트림 리다이렉트
    int fd = open("/dev/null", O_RDWR);
    if (fd >= 0) {
        dup2(fd, STDIN_FILENO);
        dup2(fd, STDOUT_FILENO);
        dup2(fd, STDERR_FILENO);
        if (fd > 2) {
            close(fd);
        }
    }
}

int main() {
    // Daemon화
    daemonize();
    
    // syslog 초기화
    openlog(DAEMON_NAME, LOG_PID, LOG_DAEMON);
    syslog(LOG_INFO, "Daemon started");
    
    // ... daemon 작업 ...
    
    syslog(LOG_INFO, "Daemon stopped");
    closelog();
    
    return EXIT_SUCCESS;
}
```

---

## 4. QNX 특화: procmgr와 Daemon

### QNX procmgr 사용

QNX는 프로세스 관리자(procmgr)를 통해 daemon 자동 재시작을 지원합니다.

```c
// 프로세스 정보 조회
#include <sys/procmgr.h>

struct procmgr_info info;
procmgr_info(&info);

printf("Max processes: %d\n", info.max_processes);
```

### Watchdog (자동 재시작)

```c
#include <sys/neutrino.h>

int main() {
    // 이 프로세스가 종료되면 procmgr가 자동 재시작
    // (spawn 시 특정 플래그 필요)
    
    // 주기적으로 "살아있음"을 신호
    while (1) {
        // 작업 수행
        sleep(5);
    }
}

// spawn 시:
// spawn -f ...
// (-f: 재시작 가능하게 설정)
```

---

## 5. PID 파일 관리

### PID 파일 작성

```c
#include <stdio.h>
#include <unistd.h>

void write_pidfile(const char *pidfile) {
    FILE *fp;
    
    fp = fopen(pidfile, "w");
    if (fp == NULL) {
        perror("fopen");
        exit(EXIT_FAILURE);
    }
    
    fprintf(fp, "%d\n", getpid());
    fclose(fp);
}

// 사용
write_pidfile("/var/run/my_daemon.pid");
```

### PID 파일 읽기

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

pid_t read_pidfile(const char *pidfile) {
    FILE *fp;
    pid_t pid;
    
    fp = fopen(pidfile, "r");
    if (fp == NULL) {
        perror("fopen");
        return -1;
    }
    
    if (fscanf(fp, "%d", &pid) != 1) {
        fprintf(stderr, "Invalid PID file\n");
        fclose(fp);
        return -1;
    }
    
    fclose(fp);
    return pid;
}

// 사용
pid_t daemon_pid = read_pidfile("/var/run/my_daemon.pid");
kill(daemon_pid, SIGTERM);  // daemon 종료 신호
```

---

## 6. 신호 처리

### Daemon에서의 신호 처리

```c
#include <signal.h>
#include <syslog.h>
#include <stdlib.h>

volatile sig_atomic_t keep_running = 1;

void signal_handler(int sig) {
    switch (sig) {
    case SIGTERM:
    case SIGINT:
        syslog(LOG_INFO, "Received termination signal");
        keep_running = 0;
        break;
    case SIGHUP:
        syslog(LOG_INFO, "Reloading configuration");
        // 설정 파일 재로드
        break;
    }
}

int main() {
    daemonize();
    
    openlog("daemon", LOG_PID, LOG_DAEMON);
    
    // 신호 핸들러 등록
    signal(SIGTERM, signal_handler);
    signal(SIGINT, signal_handler);
    signal(SIGHUP, signal_handler);
    
    // 메인 루프
    while (keep_running) {
        // 작업 수행
        sleep(1);
    }
    
    syslog(LOG_INFO, "Daemon stopped");
    closelog();
    
    return EXIT_SUCCESS;
}
```

---

## 7. Syslog를 통한 로깅

### Syslog 기본 사용

```c
#include <syslog.h>

int main() {
    openlog("my_daemon", LOG_PID | LOG_CONS, LOG_DAEMON);
    
    syslog(LOG_DEBUG, "Debug message");
    syslog(LOG_INFO, "Info message");
    syslog(LOG_WARNING, "Warning: something happened");
    syslog(LOG_ERR, "Error: %s", strerror(errno));
    syslog(LOG_CRIT, "Critical error!");
    
    closelog();
}

// 로그 확인 (Linux):
// cat /var/log/syslog | grep my_daemon
// tail -f /var/log/syslog
```

### Syslog 우선순위

```
LOG_EMERG   - 시스템 사용 불가
LOG_ALERT   - 즉시 조치 필요
LOG_CRIT    - 심각한 오류
LOG_ERR     - 오류
LOG_WARNING - 경고
LOG_NOTICE  - 공지
LOG_INFO    - 정보
LOG_DEBUG   - 디버그
```

---

## 8. Daemon 테스트

### 기본 테스트

```bash
# 1. Daemon 컴파일
qcc -o my_daemon daemon.c

# 2. Daemon 실행
./my_daemon &

# 3. 프로세스 확인
ps aux | grep my_daemon

# 4. 로그 확인
tail -f /var/log/syslog | grep my_daemon

# 5. Daemon 종료
kill <pid>
# 또는
pkill my_daemon
```

### 고급 테스트

```bash
# 표준 입출력 확인
lsof -p <daemon_pid> | grep "REG\|CHR"

# 프로세스 상태 확인
ps -p <daemon_pid> -o pid,ppid,sess,pgrp,state

# 신호 처리 확인
kill -HUP <daemon_pid>  # SIGHUP 전송
```

---

## 다음 단계

`_02_slog2.md`에서 QNX의 고급 로깅 시스템인 slog2를 학습합니다.
