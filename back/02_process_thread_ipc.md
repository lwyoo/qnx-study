# QNX 8.0 학습 로드맵 (실습 중심)

## 현재 완료한 단계

* QNX 8.0 SDK 설치
* QEMU 기반 QNX VM 생성
* 네트워크 확인 및 SSH 접속
* qcc를 이용한 C/C++ 프로그램 빌드
* VM에서 실행 확인

현재는 **개발 환경 구축 단계가 완료된 상태**이며, 이제부터는 QNX의 핵심 개념을 실제 코드로 학습하는 단계로 진입하면 된다.

---

# 권장 학습 순서

## Phase 0. 개발 환경 숙달 (완료)

### 목표

* QNX SDK 구조 이해
* VM 생성 및 관리
* SSH 접속
* qcc 빌드
* 실행 파일 배포

### 실습

* Hello World
* SSH 접속
* qcc 빌드
* VM 실행

---

## Phase 1. Process / Thread / IPC

### 목표

QNX의 핵심인 Message Passing 구조 이해

### 학습 내용

* Process
* Thread
* Channel
* Connection
* Message Passing
* Client / Server 구조

### 실습

#### Step 1 - Hello Server

```c
name_attach()
MsgReceive()
MsgReply()
```

#### Step 2 - Hello Client

```c
name_open()
MsgSend()
```

#### Step 3 - 구조체 전달

```c
struct SensorData
{
    int id;
    float value;
};
```

#### Step 4 - 다중 Client

```text
Client A
Client B
Client C
    ↓
  Server
```

### 결과

* QNX IPC 구조 이해
* Client / Server 모델 이해
* Message Passing 흐름 이해

---

## Phase 2. Pulse

### 목표

경량 이벤트 통신 이해

### 학습 내용

* Pulse
* Timer Event
* Signal Event

### 실습

```c
MsgSendPulse()
```

### 결과

* Message 와 Pulse 차이 이해
* Event 기반 설계 이해

---

## Phase 3. Service Development

### 목표

실제 서비스 프로세스 작성

### 학습 내용

* Background Service
* Signal 처리
* SIGHUP
* SIGTERM
* slog2
* Startup Script
* SLM

### 실습

#### sensor_service

```text
Sensor Service
    ↓
Message Passing
    ↓
Client
```

### 로그 출력

```c
slog2_register()
slog2f()
```

### 결과

* 서비스 작성 방법 이해
* 시스템 로그 활용 방법 습득

---

## Phase 4. Shared Memory

### 목표

고성능 프로세스 간 데이터 공유

### 학습 내용

* shm_open()
* mmap()
* Shared Memory
* Typed Memory

### 실습

```text
Process A
    ↓
Shared Memory
    ↓
Process B
```

### 결과

* 대용량 데이터 공유 방법 이해
* IPC와 Shared Memory 차이 이해

---

## Phase 5. Network Programming

### 목표

실제 차량 SW와 유사한 구조 경험

### TCP Server

```c
socket()
bind()
listen()
accept()
```

### TCP Client

```c
connect()
send()
recv()
```

### 추가 실습

* 멀티스레드 서버
* IPC + TCP Gateway

### 결과

* 네트워크 프로그래밍 이해
* Gateway 구조 경험

---

## Phase 6. Resource Manager

### 목표

QNX의 핵심 설계 철학 이해

### 학습 내용

QNX에서는 장치뿐 아니라 사용자 서비스도 파일처럼 노출할 수 있다.

예:

```text
/dev/mydevice
```

### 실습

```text
cat /dev/mydevice
echo 123 > /dev/mydevice
```

### 주요 API

```c
dispatch_create()
resmgr_attach()
io_read()
io_write()
```

### 결과

* QNX Driver Architecture 이해
* Resource Manager 패턴 이해

---

## Phase 7. C++ Application Architecture

### 목표

실무 수준의 서비스 구조 작성

### 예제

```text
Sensor Service
    ↓
IPC
    ↓
Gateway Service
    ↓
TCP Client
```

### 적용 기술

* RAII
* Smart Pointer
* STL
* Thread
* Modern C++17

### 결과

* 유지보수 가능한 서비스 구조 설계
* C++ 기반 QNX 서비스 개발

---

## Phase 8. Real-Time & Scheduling

### 목표

실시간 시스템 이해

### Scheduling

* FIFO
* RR
* Sporadic

### Real-Time

* Priority Inheritance
* Priority Ceiling

### Memory

* mmap()
* Shared Memory
* Typed Memory

### 결과

* 실시간 우선순위 설계 이해
* QNX 실시간 특성 이해

---

## Phase 9. Driver Architecture

### 목표

QNX 드라이버 구조 이해

### 학습 내용

* PCI
* Interrupt
* DMA
* Resource Manager Driver
* Driver Framework

### 결과

* BSP 및 Driver 개발 기초 확보

---

## Phase 10. Security

### 학습 내용

* Security Policy
* Pathtrust
* Abilities
* Secure ProcFS
* Trusted Boot

### 결과

* 차량 SW 보안 구조 이해

---

# 추천 문서 구조

```text
qnx_learning/

01_environment.md

02_process_thread_ipc.md
03_pulse.md

04_service_development.md

05_shared_memory.md

06_network_programming.md

07_resource_manager.md

08_cpp_application_architecture.md

09_realtime_and_scheduling.md

10_driver_architecture.md

11_security.md

notes/
```

---

# 우선순위

## 가장 중요

### 1. Message Passing

설명 가능해야 함

```text
Client
    ↓
 Channel
    ↓
 Server
```

### 2. Microkernel

Linux와 차이 설명 가능해야 함

### 3. Resource Manager

QNX 설계 철학의 핵심

### 4. Scheduling

실시간 우선순위 및 응답성

---

# 추천 다음 실습

## 이번 주

### Day 1

Hello IPC

* Server
* Client

### Day 2

구조체 전송

### Day 3

Pulse

### Day 4

멀티스레드 IPC

### Day 5

정리 문서 작성

---

# 현재 상태 평가

```text
QNX SDK 설치         완료
VM 생성              완료
SSH 접속             완료
qcc 빌드             완료
실행파일 배포        완료

Process/Thread       다음 단계
IPC                  다음 단계
Pulse                이후 단계
Service              이후 단계
Resource Manager     심화 단계
Driver 개발          고급 단계
```

현재 시점에서는 GUI나 드라이버보다 먼저 다음 순서를 추천한다.

```text
Process / Thread
        ↓
IPC
        ↓
Pulse
        ↓
Service
        ↓
Shared Memory
        ↓
Network
        ↓
Resource Manager
```

이 순서가 QNX의 핵심 설계를 가장 자연스럽게 이해할 수 있는 학습 경로이다.
