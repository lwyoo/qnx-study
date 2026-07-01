# The QNX OS Microkernel

* 마이크로커널의 역할
    *  임베디드 실시간 시스템용 핵심 POSIX 기능 구현
    *  메시지 패싱 서비스 제공
    *  파일/디바이스 I/O 같은 나머지는 user space의 프로세스가 담당

* 메시지 패싱 기능
    * QNX의 기본 통신 메커니즘

* 기본 객체 (Thread, Channel, Dispatch 등)
    * 마이크로커널 저수준에 기본 객체들이 있다
    * 그 객체들을 조작하는 최적화된 루틴들이 있다
    * 이 위에 OS 전체가 구축된다

---

# Design goals of the QNX OS
> 설계 목표: 
> 메모리 제한이 많은 임베디드 시스템  ←→  고성능 SMP 머신 (기가바이트 메모리)
> → 한 OS로 둘 다 효율적으로 지원하는 게 설계 목표

* POSIX 실시간 및 스레드 확장
  * 마이크로커널에 직접 구현
    * 실시간 서비스 (타이머, 우선순위 스케줄링 등)
    * 스레드 서비스
      → 마이크로커널에 내장되어 있음
    장점: 추가 OS 모듈 없어도 바로 사용 가능

* 프로세스 모델 확장
  * Level 1: 스레드만 필요한 경우
         → 마이크로커널이 직접 처리

  * Level 2: 다중 스레드를 가진 프로세스
         → 프로세스 매니저가 확장

* 메모리 보호와 POSIX 완전성
  * 다른 실시간 커널과의 차이
  
  |다른 RTOS | QNX|
  |--|--|
  스레드만 지원 (메모리 보호 없음) | 스레드 + 프로세스 + 메모리 보호
  프로세스 모델 없음 | 완전한 프로세스 모델
  POSIX 미준수 | POSIX 완전 준수 가능

  → 메모리 보호 없는 순수 스레드 모델로는 POSIX 완전성 불가능

* 정리
  * 임베디드부터 고성능까지 모든 범위 지원
  * 스레드 + 프로세스 + 메모리 보호 세 가지 모두 제공
  * 덕분에 진정한 POSIX 준수 가능

---

# System Services

* 마이크로커널이 제공하는 10가지 Kernel Calls
  1. threads           - 스레드 관리
  2. message passing   - 프로세스 간 통신
  3. signals          - 신호 처리
  4. clocks           - 시간 정보
  5. timers           - 타이머
  6. interrupt handling - 인터럽트
  7. semaphores       - 세마포어
  8. mutexes          - 뮤텍스 (상호배제)
  9. condition variables - 조건 변수
  10. barriers        - 배리어 (동기화)

    → 이 10가지가 **모든 OS 서비스의 기초**가 된다.

* 완전한 선점성 (Full Preemptibility)
  * 메시지 패싱 중간에도 언제든 선점 가능
  * 선점 후 복귀하면 정확한 지점에서 재개
  * 실시간 응답성 보장

* 마이크로커널 설계의 핵심 원칙
  * 짧은 Non-Preemptible Code Path
    * 커널의 단순성 → 인터럽트 비활성화 구간 최소화
    * 일반적으로 수백 나노초 수준
  * 작은 코드 크기
    * 코드가 작으면 멀티프로세서 문제 분석 용이
    * 버그 가능성 감소

* 기능 분배 원칙: 커널 vs 외부 프로세스
  * 선택 기준
    ```
    ✅ 커널에 포함:
    - 짧은 실행 경로의 작업들
    - 예: 컨텍스트 스위칭, 메시지 큐

    ❌ 외부 프로세스에 위임:
    - 복잡한 작업들
    - 예: 프로세스 로딩, 파일시스템 접근
    ```
    → 컨텍스트 스위칭 오버헤드 < 실제 작업 시간

* 마이크로커널 vs 모놀리식 커널 성능 비교 (중요)
  * 흔한 오해
    ```
    ❌ "마이크로커널은 모놀리식보다 느리다"
    (컨텍스트 스위칭 때문에)
    ```
  * 왜 빠를 수 있나:
    ```
    총 실행 시간 = 컨텍스트 스위칭 시간 + 실제 작업 시간

    마이크로커널:
    컨텍스트 스위칭: 매우 빠름 (간단한 구조)
    실제 작업: 충분한 시간
    → 스위칭 시간이 작업 시간에 묻힘

    모놀리식:
    스위칭 없음 (같은 커널 공간)
    하지만 복잡한 커널로직 + 메모리 보호 오버헤드
    ```
  * 핵심 인사이트
    **→ 컨텍스트 스위칭 비용이 실제 작업량에 비해 무시할 수 있는 수준**

* 인터럽트 비활성화 시간
  * 인터럽트 또는 선점이 꺼지는 시간: ~수백 ns
  * 마이크로초 단위가 아닌 나노초 단위
  **→ 매우 빠른 실시간 응답성**

---

#  Threads and processes
> 이 문서는 QNX에서 스레드와 프로세스의 개념과 사용법을 설명한다.

* Thread vs Process
  * Thread
    ```
    정의: 실행의 최소 단위

    특징:
    ├─ 마이크로커널이 스케줄링하는 단위
    ├─ 실행 흐름 (execution context)
    ├─ CPU 사이클 할당받는 대상
    └─ 가벼운 실행 단위
    ```
  * Process
    ```
    정의: 스레드의 컨테이너

    특징:
    ├─ 주소 공간 (메모리 영역) 정의
    ├─ 하나 이상의 스레드 포함
    ├─ MMU로 보호됨 (다른 프로세스와 격리)
    └─ 프로세스는 최소 1개의 스레드 포함 필수
    ```
  * 관계
    ```
    프로세스
        ├─ 스레드 1 ✓
        ├─ 스레드 2 ✓
        ├─ 스레드 3 ✓
        └─ 스레드 N ✓
    
            ↓
    
    같은 프로세스 내 스레드들:
    • 같은 주소 공간 공유
    • 메모리 직접 접근 가능
    • 빠른 통신
    
    다른 프로세스의 스레드들:
    • 독립적 주소 공간
    • 메모리 접근 불가
    • 메시지 패싱으로 통신
    ```
* 동시성 (Concurrency)
  * 왜 여러 스레드를 쓰는가?
    ```
    시나리오: 음악 스트리밍 앱

    스레드 1: UI 처리 (버튼 클릭)
    스레드 2: 네트워크에서 음악 데이터 받기
    스레드 3: 음성 디코딩
    스레드 4: 스피커로 음성 출력

    → 모두 동시에 실행 (동시성)
    → 사용자 경험 향상
    ```
  * 통신 방식
    ```
    같은 프로세스 내 스레드:
        ├─ 메모리 공유 (가장 빠름)
        ├─ 뮤텍스, 세마포어로 동기화
        └─ 높은 대역폭 통신

    다른 프로세스 스레드:
        ├─ 메시지 패싱 (IPC)
        ├─ 마이크로커널 중계
        └─ 안전하지만 느림
    ```

* pthread_* 호출 분류
  * Microkernel 호출 없는 것들 (라이브러리만)
      ```
      이 호출들은 커널을 거치지 않음:

      pthread_attr_*
      • 속성 설정 (스택 크기, 스케줄 정책 등)
      • 라이브러리에서만 처리
      • SYSCALL 아님

      pthread_key_*
      • Thread-local storage 관리
      • C 라이브러리 기능

      pthread_self(), pthread_equal()
      • 정보 조회만
      ```
  * Microkernel 호출이 있는 것들 (주요)

    |POSIX 호출|Microkernel 호출|역할|
    |-|-|-|
    |pthread_create()|ThreadCreate()|새 스레드 생성|
    |pthread_exit()|ThreadDestroy()|스레드 종료|
    |pthread_join()|ThreadJoin()|스레드 종료 대기|
    |pthread_cancel()|ThreadCancel()|스레드 취소|
    ||ThreadCtl()|QNX 특화 설정|

* 뮤텍스 (Mutex)
  * POSIX vs Microkernel
    ```
    pthread_mutex_init()       → SyncTypeCreate()
        생성

    pthread_mutex_lock()       → SyncMutexLock()
        잠금 (다른 스레드 대기)

    pthread_mutex_trylock()    → SyncMutexLock()
        조건부 잠금

    pthread_mutex_unlock()     → SyncMutexUnlock()
        잠금 해제
      ```
  * 뮤텍스의 역할
    ```
    상황: 두 스레드가 같은 데이터 접근

    스레드 A: count++ (count = 1)
    스레드 B: count++ (count = 1)

    ❌ 뮤텍스 없이:
    결과: count = 1 (잘못됨, 2여야 함)

    ✅ 뮤텍스 사용:
    스레드 A: lock(mutex) → count++ → unlock(mutex)
    스레드 B: lock(mutex) [대기] → count++ → unlock(mutex)
    결과: count = 2 (정확함)
    ```

* 조건 변수 (Condition Variable)
  * POSIX vs Microkernel
    ```
    pthread_cond_init()          → SyncTypeCreate()
        생성

    pthread_cond_wait()          → SyncCondvarWait()
        대기 (신호 받을 때까지)

    pthread_cond_signal()        → SyncCondvarSignal()
        신호 (한 스레드만 깨우기)

    pthread_cond_broadcast()     → SyncCondvarSignal()
        신호 (모든 스레드 깨우기)
    ```
  * 조건 변수 사용 예시
    ```
    시나리오: 버퍼에 데이터가 들어올 때까지 기다리기

    생산자 스레드:
        데이터 생성
        버퍼에 삽입
        cond_signal() 호출 → 소비자 깨우기

    소비자 스레드:
        cond_wait() → 대기 (버퍼 비어있으면)
        신호 받으면 → 버퍼에서 데이터 가져오기
    ```

* 스케줄링
  * POSIX vs Microkernel
  ```
    pthread_sigmask()            → SignalProcmask()
        스레드의 신호 마스크 관리

    pthread_kill()               → SignalKill()
        특정 스레드에 신호 전송
  ```

* 주요 개념
  * MMU 보호
    ```
    프로세스 A의 메모리
    ├─ 스레드 A1
    └─ 스레드 A2
    
    프로세스 B의 메모리
    ├─ 스레드 B1
    └─ 스레드 B2

    = MMU가 프로세스 경계 보호
    = 각 프로세스는 자신의 메모리만 접근 가능
    ```
  * IPC (Inter-Process Communication)
    ```
    "IPC" = 프로세스 간 통신
    (실제로는 스레드 간 통신도 포함)

    같은 프로세스:
    • 메모리 공유 통신 (빠름)

    다른 프로세스:
    • 메시지 패싱 (안전함)
    ```

* 선택 가이드
  * pthread_* vs Microkernel 호출
    ```
    일반적인 경우:
    → pthread_* 사용 권장
    
    특수한 경우:
    → Microkernel 호출 직접 사용
      (예: ThreadCtl() - QNX 특화 기능)
    ```
  * 왜 pthread_*가 낫나?
    ```
    pthread_* 호출:
    • POSIX 표준 (이식성)
    • C 라이브러리에서 추가 기능 제공
    • 더 사용하기 쉬움

    Microkernel 호출:
    • 낮은 수준 (direct access)
    • QNX 특화 기능만
    ```

* 정리
    ```
    Thread
    = 실행의 최소 단위
    = 마이크로커널이 스케줄링하는 대상
    = CPU 사이클 할당받는 단위

    Process
    = 스레드의 컨테이너
    = 메모리 주소 공간 정의
    = MMU로 보호됨

    POSIX pthread_*
    = 스레드/뮤텍스/조건변수 관리
    = 마이크로커널 호출 감싸는 라이브러리
    → 일반적으로 이것을 사용
    ```

---

## Thread personas

* 정의
    ```
    Persona = 스레드의 "모드" 또는 "상태"

    스레드는 2가지 모드로 실행된다:
    1. User Persona (사용자 모드)
    2. Kernel Persona (커널 모드)
    ```

*  User Persona (사용자 모드)
   *  특징
        ```
        상태:
            • 사용자 코드 실행 중
            • 일반적인 애플리케이션 코드
        
        권한:
            • 제한된 권한 수준
        
        메모리:
            • 프로세스의 주소 공간만 접근 가능
            • 커널 메모리 비활성화
            
        스택:
            • 사용자 스택 사용
        
        예시:
            int x = 10;
            printf("%d\n", x);
            // ← User Persona에서 실행
        ```
*  Kernel Persona (커널 모드)
   *  특징
        ```
        상태:
            • SYSCALL 호출했을 때 전환
            • 커널 코드 실행 중
        
        권한:
            • 상승된 권한 수준
        
        메모리:
            • 커널 주소 공간 접근 가능
            • 시스템 리소스 접근 가능
        
        스택:
            • 커널 스택 사용
            • 사용자 스택과 분리됨
        
        예시:
            fd = open("file.txt", O_RDONLY);
            // ← SYSCALL → Kernel Persona로 전환
        ```
   *  변환
        ```
        User Persona에서 Kernel Persona로 전환 시:

        ① 권한 상승
        User ← → Kernel (elevated)

        ② 주소 공간 변경
        User address space only ← → Kernel + User both visible

        ③ 스택 변경
        User stack ← → Kernel stack
        ```
*  Persona 전환
   *  스레드의 특징 유지
        ```
        Kernel Persona에서도 변하지 않는 것:

        ✓ 우선도 (Priority) - 그대로 유지
        ✓ 스케줄링 알고리즘 - 그대로 적용
        ✓ 선점 가능 - 여전히 선점 가능
        ✓ 인터럽트 처리 - 여전히 가능

        = 스레드의 정체성은 유지
        ```
   *  시간 다이어그램
        ```
        스레드 실행

        [User Persona]
            printf("Hello");
            ↓ (open() SYSCALL 호출)
        [Kernel Persona]
            fd = open(...)
            (커널 권한으로 파일 오픈)
            ↓ (open() 완료)
        [User Persona]
            if (fd < 0) { ... }
            (다시 사용자 코드)
        ```

*  IST (Interrupt Service Thread)
   *  정의
        ```
        IST = Kernel Persona만 가진 특수한 스레드

        생성:
            InterruptAttachEvent() 호출로 생성
        ```
   *  특징
        ```
        ✗ User Persona 없음
        ✗ User Stack 없음
        ✗ 사용자 코드 절대 실행 안 함

        ✓ Kernel Persona만 존재
        ✓ Kernel Stack만 사용
        ✓ 인터럽트 처리만 수행
        ```
   *  구조
        ```
        일반 스레드:
            ├─ User Persona (사용자 코드)
            └─ Kernel Persona (커널 코드)

        IST:
            └─ Kernel Persona만 (인터럽트 처리)
        ```
   *  사용 예시
        ```
        시나리오: 네트워크 인터럽트 처리

        하드웨어:
            💥 네트워크 카드에서 데이터 도착!
            ↓
        IST 실행:
            InterruptAttachEvent()로 생성된 IST가
            Kernel Persona에서만 실행
            ↓
            데이터 처리
        ```
*  싱글 스레드 판정
   *  중요한 개념
        ```
        프로세스에 스레드가 2개 있어도:

        스레드 1: User + Kernel Persona (일반 스레드)
        스레드 2: Kernel Persona만 (IST)

        → "싱글 스레드"로 간주됨
            (동기화 목적으로)
        ```
    * 왜 이렇게 간주하는가?
        ```
        일반 스레드 1개:
        • User Persona에서 사용자 코드 실행
        • 동기화 필요

        IST:
        • User Persona 없음
        • 사용자 코드와 상호작용 없음
        • 동기화 불필요

        → 따라서 "싱글 스레드 안전성" 보장
        ```
*  정리
   *  Persona 비교
        |구분|User Persona|Kernel Persona|
        |---|---|---|
        |코드 타입|사용자 코드|커널 코드|
        |권한|제한됨|상승됨|
        |메모리|User 공간만|Kernel + User|
        |스택|User Stack|Kernel Stack|
        |트리거|일반 실행|SYSCALL|
        |선점|가능|가능|

   *  IST 특수성
        |구분|일반 스레드|IST|
        |---|---|---|
        |Persona 개수|2개|1개|
        |User Persona|✓ 있음|✗ 없음|
        |Kernel Persona|✓ 있음|✓ 있음|
        |User Stack|✓ 있음|✗ 없음|
        |용도|일반 처리|인터럽트만|
        |Thread Count|카운트됨|별도 분류|
*  핵심 이해
    ```
    Persona
        = 스레드의 실행 모드 상태
        = 권한/메모리/스택이 달라지는 단계
    
    전환:
        User → SYSCALL → Kernel → 완료 → User
    
    특수:
        IST는 Kernel Persona만 (User Persona 없음)
    ```

---
## Thread attributes
> 스레드들은 프로세스의 주소 공간을 공유하지만,
> 각 스레드는 고유한 개인 데이터를 가진다
> 공유: 주소 공간, 메모리, 파일 디스크립터
> 개인: tid, priority, stack, register set 등
> 
## Thread Local Storage (TLS)
## Thread lifecycle

---

# Thread scheduling
## Scheduling priority
## Scheduling policies
### FIFO scheduling
### Round-robin scheduling
### Sporadic scheduling
### Manipulating priority and scheduling policies
## IPC issues
## Thread complexity issues
## External code invoked by the scheduler