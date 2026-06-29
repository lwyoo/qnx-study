# 첫 번째 QNX 빌드 및 실행 (Hello World)

## 1. 간단한 Hello World 프로그램 작성

### 프로젝트 구조

```
qnx_hello_world/
├── main.c
├── Makefile
└── README.md
```

### main.c 작성

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    printf("Hello, QNX Neutrino 8.0!\n");
    printf("Process ID: %d\n", getpid());
    printf("User ID: %d\n", getuid());
    
    return EXIT_SUCCESS;
}
```

---

## 2. Makefile 작성

### QNX 표준 Makefile 구조

```makefile
# Makefile for QNX Neutrino

# 대상 실행 파일 이름
TARGET = qnx_hello

# 소스 파일
SRCS = main.c

# 컴파일 플래그
CFLAGS = -Wall -g -O0

# QNX 컴파일러 사용 (환경 변수에 의존)
CC = qcc
CFLAGS += -fPIC

# 대상 파일
OBJS = $(SRCS:.c=.o)

# 기본 타겟
all: $(TARGET)

# 실행 파일 생성
$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^

# 목적 파일 생성
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

# 정리
clean:
	rm -f $(OBJS) $(TARGET)

# PHONY 타겟 선언
.PHONY: all clean

# 빌드 상태 출력
info:
	@echo "Target: $(TARGET)"
	@echo "Sources: $(SRCS)"
	@echo "Compiler: $(CC)"
	@echo "CFLAGS: $(CFLAGS)"
```

---

## 3. 빌드 과정

### 환경 확인

```bash
# QNX 환경 변수 확인
echo $QNX_HOST
echo $QNX_TARGET

# 컴파일러 경로 확인
which qcc
# 출력: /home/user/qnx800/host/linux/x86_64/usr/bin/qcc
```

### 컴파일 명령어

```bash
# 단일 파일 컴파일
qcc -Wall -g -O0 -o qnx_hello main.c

# Makefile 사용
make clean
make

# 빌드 결과 확인
file qnx_hello
# 출력: qnx_hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), ...
```

### 세부 컴파일 과정

```bash
# 상세 컴파일 로그
qcc -v main.c

# 전처리만 수행
qcc -E main.c

# 어셈블리 코드 생성
qcc -S main.c
cat main.s | head -30
```

---

## 4. 호스트 시스템에서 실행

### 직접 실행 (x86_64 호스트)

```bash
# 실행 파일 권한 확인
ls -la qnx_hello
# 출력: -rwxr-xr-x 1 user user 7912 Jun 29 10:30 qnx_hello

# 실행
./qnx_hello
# 출력:
# Hello, QNX Neutrino 8.0!
# Process ID: 12345
# User ID: 1000
```

### 시스템 콜 추적

```bash
# strace를 사용하여 시스템 콜 모니터링
strace -e trace=open,read,write,exit_group ./qnx_hello

# 또는 QNX 환경에서:
truss ./qnx_hello
```

---

## 5. QEMU VM에서 실행

### IFS 생성

먼저 QEMU 이미지를 만들어야 합니다:

```bash
# 빌드한 바이너리를 임베디드 시스템에 포함시킬 수 있음
# 이는 다음 섹션 (_03_debugging.md)에서 자세히 다룸

# 간단한 예: 기본 IFS에서 QNX 시스템 부팅
$QNX_HOST/usr/bin/mkifs \
  -b $QNX_TARGET/x86_64/boot/sys/startup-x86 \
  -r $QNX_TARGET/x86_64 \
  qnx.ifs

# 또는 IFS 스크립트 사용 (다음 섹션 참조)
```

### QEMU로 부팅 및 실행

```bash
# QEMU 이미지 실행 (자세히는 01_qemu_vm_setup.md 참조)
qemu-system-x86_64 -m 1024 -kernel qnx.ifs

# 부팅 후 호스트에서:
ssh root@192.168.1.100  # 또는 직렬 포트로 접속

# QEMU 내부에서:
./qnx_hello
```

---

## 6. 멀티 프로세스 확장

### Producer-Consumer 예제

```c
#include <stdio.h>
#include <stdlib.h>
#include <spawn.h>
#include <wait.h>

int main() {
    pid_t pid;
    int status;
    
    // 자식 프로세스 생성
    pid = fork();
    
    if (pid == 0) {
        // 자식 프로세스
        printf("Child process: PID=%d, PPID=%d\n", 
               getpid(), getppid());
        exit(EXIT_SUCCESS);
    } else if (pid > 0) {
        // 부모 프로세스
        printf("Parent process: PID=%d, Child=%d\n", 
               getpid(), pid);
        
        // 자식 프로세스 종료 대기
        waitpid(pid, &status, 0);
        printf("Child exited with status: %d\n", status);
    } else {
        perror("fork failed");
        exit(EXIT_FAILURE);
    }
    
    return EXIT_SUCCESS;
}
```

### Makefile 업데이트

```makefile
TARGETS = qnx_hello fork_example
SRCS_hello = main.c
SRCS_fork = fork_example.c

all: $(TARGETS)

qnx_hello: $(SRCS_hello)
	$(CC) $(CFLAGS) -o $@ $^

fork_example: $(SRCS_fork)
	$(CC) $(CFLAGS) -o $@ $^

clean:
	rm -f $(TARGETS)

.PHONY: all clean
```

---

## 7. 성능 측정

### 실행 시간 측정

```bash
# 시간 측정
time ./qnx_hello

# 출력:
# Hello, QNX Neutrino 8.0!
# Process ID: 12345
# User ID: 1000
# real    0m0.001s
# user    0m0.000s
# sys     0m0.001s
```

### 메모리 사용량 확인

```bash
# 프로세스 메모리 추적
/proc/self/status | grep VmRSS

# 또는 다음 프로그램으로:
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/resource.h>

int main() {
    struct rusage usage;
    getrusage(RUSAGE_SELF, &usage);
    
    printf("Memory (KB): %ld\n", usage.ru_maxrss);
    printf("User time: %ld.%06lds\n", 
           usage.ru_utime.tv_sec, usage.ru_utime.tv_usec);
    
    return 0;
}
```

---

## 8. 일반적인 오류 및 해결

### 오류 1: qcc: command not found

```bash
# 해결: 환경 변수 설정
source $HOME/qnx800/qnxenv-8.0.0.sh
# 또는
export PATH=$QNX_HOST/usr/bin:$PATH
```

### 오류 2: undefined reference to 'main'

```
# 원인: 링킹 단계 실패
# 해결: Makefile의 SRCS 확인
```

### 오류 3: permission denied

```bash
# 원인: 실행 권한 없음
# 해결:
chmod +x qnx_hello
```

---

## 9. 최적화 및 디버그 정보

### 디버그 정보 포함

```bash
# -g 옵션으로 디버그 심볼 포함
qcc -g -O0 -o qnx_hello_debug main.c

# 바이너리 크기 비교
ls -la qnx_hello*
```

### 최적화 수준

```c
// 여러 최적화 수준 비교
// -O0: 최적화 없음 (디버깅용)
// -O1: 기본 최적화
// -O2: 중간 최적화
// -O3: 최대 최적화 (크기 증가 가능)
// -Os: 크기 최적화
```

---

## 다음 단계

`_03_debugging.md`에서 디버깅 도구와 기법을 학습합니다.
