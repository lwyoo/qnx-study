# QNX 디버깅 (GDB, IDE, Momentics)

## 1. 디버깅 환경 구축

### 필요한 도구

| 도구 | 목적 | 설명 |
|------|------|------|
| **gdb** | 소스 레벨 디버깅 | 브레이크포인트, 변수 검사 |
| **Momentics IDE** | 통합 개발 환경 | GUI 기반 디버깅 (권장) |
| **qde** | 원격 디버깅 서버 | QEMU/타겟 시스템 디버깅 |
| **pdebug** | QNX 프로세스 디버거 | 타겟 시스템 프로세스 모니터 |

### 디버그 심볼 포함한 컴파일

```bash
# 디버그 정보 포함하여 컴파일
qcc -g -O0 -o qnx_hello_debug main.c

# 바이너리 정보 확인
file qnx_hello_debug
readelf -S qnx_hello_debug | grep debug
```

---

## 2. 명령줄 GDB를 사용한 디버깅

### GDB 시작

```bash
# 바이너리로 GDB 시작
gdb ./qnx_hello_debug

# 또는 실행 후 attach
gdb attach <pid>
```

### 기본 GDB 명령어

```bash
# 브레이크포인트 설정
(gdb) break main
(gdb) break main.c:10

# 조건부 브레이크포인트
(gdb) break main.c:15 if x > 10

# 프로그램 실행
(gdb) run [args]
(gdb) continue
(gdb) step        # 라인 단위 실행 (함수 내부 진입)
(gdb) next        # 라인 단위 실행 (함수 건너뜀)
(gdb) finish      # 현재 함수 종료까지 실행

# 변수 검사
(gdb) print variable_name
(gdb) print &variable_name        # 주소 출력
(gdb) print sizeof(type)
(gdb) watch variable_name         # 변수 값 변화 감시

# 백트레이스 (콜 스택)
(gdb) backtrace
(gdb) frame 0
(gdb) frame 1

# 소스 코드 보기
(gdb) list
(gdb) list main.c:10,20

# 메모리 검사
(gdb) x/16x $rsp                  # RSP부터 16개 8바이트 16진수 출력
(gdb) x/s 0x7fffffffde70          # 문자열 출력

# 레지스터 확인
(gdb) info registers
(gdb) info registers rax rbx rcx

# 종료
(gdb) quit
```

---

## 3. 어드밴스 GDB 디버깅

### 멀티 스레드 디버깅

```bash
(gdb) info threads              # 모든 스레드 정보
(gdb) thread 1                  # 스레드 1로 전환
(gdb) thread apply all bt       # 모든 스레드의 콜스택 출력
(gdb) set scheduler-locking on  # 한 스레드만 실행
```

### 세그멘테이션 폴트 디버깅

```bash
# 코어 덤프 생성 가능하도록 설정
ulimit -c unlimited

# 프로그램 실행 후 segfault 발생
./buggy_program

# 코어 덤프 분석
gdb ./buggy_program core

(gdb) backtrace  # 폴트 발생 위치 확인
(gdb) frame 0
(gdb) print memory_location
```

### 성능 프로파일링

```bash
# perf로 CPU 사용률 측정
perf record ./qnx_hello_debug
perf report

# valgrind로 메모리 누수 검사
valgrind --leak-check=full ./qnx_hello_debug
```

---

## 4. Momentics IDE 설정 및 디버깅

### Momentics IDE 시작

```bash
# Momentics 실행
$QNX_HOST/bin/qde &

# 또는 설치된 경로
~/qnx800/momentics/qde &
```

### 프로젝트 생성

1. File → New → QNX C Project
2. 프로젝트 이름 입력: `HelloQNX`
3. 프로젝트 타입 선택: `Executable`
4. 다음 단계에서 기본 설정 유지
5. Finish

### 소스 코드 작성

1. Project Explorer에서 `src/` 폴더 우클릭
2. New → File → `main.c`
3. 다음 코드 작성:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/neutrino.h>

int main(void) {
    printf("Starting QNX Debug Session\n");
    printf("Process ID: %d\n", getpid());
    
    for (int i = 0; i < 3; i++) {
        printf("Loop iteration: %d\n", i);
        sleep(1);
    }
    
    printf("Program finished\n");
    return 0;
}
```

### IDE에서 빌드

```
Project → Build Project
또는 Ctrl+B
```

### IDE에서 디버그 실행

1. Run → Debug As → QNX C/C++ Application
2. 디버그 뷰 자동 오픈

### IDE 디버그 기능

| 기능 | 단축키 | 설명 |
|------|--------|------|
| 실행 | F11 | Step Into |
| 다음 라인 | F10 | Step Over |
| 계속 실행 | F8 | Resume |
| 종료 | - | Terminate |

### 변수 감시 (Watch)

1. Variables 탭 → Add Watch Expression
2. 변수명 입력: `i`, `argv[]` 등
3. 프로그램 실행 중 값 변화 실시간 확인

---

## 5. 원격 디버깅 (QEMU 타겟)

### qemu에서 디버그 서버 활성화

```bash
# QEMU 실행 시 gdbserver 포트 지정
qemu-system-x86_64 \
  -m 1024 \
  -kernel qnx.ifs \
  -s                    # -s: gdbserver 활성화 (port 1234)
```

### 호스트에서 원격 디버깅

```bash
# 호스트 GDB 실행
gdb ./qnx_hello_debug

# QEMU 타겟에 연결
(gdb) target remote localhost:1234

# 디버깅 시작
(gdb) break main
(gdb) continue
```

### Momentics에서 원격 디버깅

1. Run → Debug Configurations
2. QNX Neutrino 선택 → New
3. Main 탭:
   - Project: 프로젝트명
   - C/C++ Application: 실행 파일 경로
4. Debugger 탭:
   - Connection: TCP
   - Host: 192.168.1.100 (타겟 IP)
   - Port: 8000 (또는 지정된 포트)
5. Debug 클릭

---

## 6. 시스템 콜 추적 (truss)

### truss 기본 사용법

```bash
# 모든 시스템 콜 추적
truss ./qnx_hello

# 특정 시스템 콜만 추적
truss -e open,read,write ./qnx_hello

# 타이밍 정보 포함
truss -t ./qnx_hello

# 파일로 저장
truss -o trace.txt ./qnx_hello
```

### 출력 예시

```
execve("./qnx_hello", 0x7fffde70, 0x7fffde88)   = OK
mmap(0, 0x0, 0, 0x1000, -1, 0) = 0x7fd7b6a000
open("/lib/ld.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF...", 832) = 832
...
write(1, "Hello, QNX Neutrino 8.0!\n", 25) = 25
exit(0)
```

---

## 7. 프로세스 모니터링 (pdebug)

### pdebug로 프로세스 모니터

```bash
# 프로세스 정보 확인
ps aux | grep qnx_hello

# pdebug로 모니터
pdebug ./qnx_hello

# 또는 실행 중인 프로세스에 attach
pdebug -p <pid>
```

### 명령어

```
pdebug> go          # 프로세스 실행 재개
pdebug> stop        # 프로세스 중지
pdebug> signals     # 신호 정보
pdebug> quit        # 종료
```

---

## 8. Valgrind를 사용한 메모리 디버깅

### 메모리 누수 검사

```bash
# 전체 메모리 누수 검사
valgrind --leak-check=full \
         --show-leak-kinds=all \
         --log-file=valgrind-out.txt \
         ./qnx_hello

# 결과 확인
cat valgrind-out.txt | tail -20
```

### 출력 분석

```
==12345== HEAP SUMMARY:
==12345==     in use at exit: 0 bytes in 0 blocks
==12345==   total heap alloc: 1,024 bytes in 5 blocks
==12345==   total heap free: 1,024 bytes in 5 blocks
==12345== ERROR SUMMARY: 0 errors from 0 contexts
```

---

## 9. 커스텀 디버그 매크로

### printf 기반 디버깅

```c
#ifdef DEBUG
#define DBG(fmt, args...) \
    fprintf(stderr, "[%s:%d] " fmt "\n", __FILE__, __LINE__, ##args)
#else
#define DBG(fmt, args...)
#endif

// 사용
DBG("Variable value: %d", value);

// 컴파일: qcc -DDEBUG -o prog main.c
```

### assert 사용

```c
#include <assert.h>

void process_array(int *arr, int size) {
    assert(arr != NULL);
    assert(size > 0);
    assert(size < 10000);
    
    for (int i = 0; i < size; i++) {
        assert(arr[i] >= 0);  // 각 원소 검증
    }
}
```

---

## 10. 일반적인 디버깅 전략

### 분할 정복 (Divide & Conquer)

1. 함수 경계에 print 문 삽입
2. 범위를 좁혀나가며 원인 찾기
3. gdb로 정확한 위치 특정

### 상태 검사 (State Inspection)

```c
// 핵심 지점에서 상태 출력
printf("[DEBUG] State at %s:%d\n", __func__, __LINE__);
printf("  var_x = %d\n", x);
printf("  var_y = %p\n", y);
```

### 회귀 테스트

```bash
# 동일한 조건으로 반복 실행
for i in {1..10}; do
    ./qnx_hello_debug
done
```

---

## 다음 단계

아키텍처 섹션의 `03_linux_vs_qnx.md`에서 Linux와 QNX의 설계 차이를 학습합니다.
