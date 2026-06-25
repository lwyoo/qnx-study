# QNX SDK 학습 노트

# 1. QNX SDK 구조

## 1.1 Host 와 Target

### Host

개발 PC(Ubuntu)에서 실행되는 도구들

예)

```text
qcc
q++
mkifs
mkqnximage
ntox86_64-gdb
```

Host는 "타겟용 프로그램을 만들기 위한 개발 도구들의 집합"이라고 생각하면 된다.

---

### Target

실제 QNX 시스템에 들어가는 파일들

예)

```text
startup-x86
procnto-smp-instr

libc.so

io-sock
pci-server
devb-*
```

Target은 실제 하드웨어 또는 가상머신에서 실행되는 바이너리와 라이브러리의 집합이다.

---

# 2. QNX 이미지 생성 과정

QNX가 부팅 가능한 이미지를 만드는 과정이다.

---

## 2.1 ifs.build

### 역할

IFS(Image File System)에 포함할 파일과 초기 실행 절차를 정의한다.

예)

```text
startup-x86
procnto-smp-instr

/bin/sh
/bin/pidin

[type=script] startup-script = {
    slogger2
    pci-server
    io-sock
}
```

### 포함 내용

* 어떤 파일을 IFS에 포함할지
* 어떤 라이브러리를 포함할지
* 어떤 서비스를 부팅 시 실행할지
* 어떤 순서로 실행할지

---

## 2.2 mkifs

### 역할

```text
ifs.build
    ↓
mkifs
    ↓
ifs.bin
```

Buildfile을 실제 부팅 이미지(IFS)로 변환하는 도구이다.

실제 위치:

```bash
~/qnx800/host/linux/x86_64/usr/bin/mkifs
```

---

## 2.3 IFS (Image File System)

### 역할

QNX가 부팅 직후 사용하는 최소 파일시스템이다.

부팅 직후에는 아직

```text
디스크 드라이버
파일시스템 드라이버
네트워크 드라이버
```

가 동작하지 않는다.

그래서 QNX는 부팅에 필요한 모든 파일을 하나의 이미지에 묶는다.

그것이 IFS이다.

---

### ifs.bin 내부 예시

```text
ifs.bin

├── startup-x86
├── procnto-smp-instr
├── libc.so
├── devc-ser8250
├── slogger2
├── sh
├── ls
├── pidin
└── startup script
```

---

### ISO 와의 차이

ISO

```text
CD/DVD 파일시스템 이미지
```

예)

```text
ubuntu.iso
```

IFS

```text
QNX 부팅 이미지
```

예)

```text
ifs.bin
```

---

## 2.4 mkqnximage

### 역할

QNX VM 또는 Target 이미지를 생성하는 상위 레벨 도구

내부적으로

```text
mkqnximage
 ├─ mkifs
 ├─ 파일시스템 생성
 ├─ 디스크 이미지 생성
 └─ QEMU 실행
```

을 수행한다.

---

### 실제 흐름

```text
ifs.build
   ↓
mkifs
   ↓
ifs.bin
   ↓
mkqnximage
   ↓
disk-qemu.vmdk
   ↓
QEMU 실행
```

---

### 예제

```bash
mkdir ~/qnx_vm
cd ~/qnx_vm

mkqnximage \
    --type=qemu \
    --arch=x86_64 \
    --ssh-ident=none
```

생성 결과

```text
local/
output/
```

---

# 3. QNX 부팅 과정

생성된 ifs.bin이 실제로 어떻게 실행되는지 설명한다.

---

## 3.1 전체 흐름

```text
BIOS / UEFI
        ↓
Bootloader
        ↓
startup-x86
        ↓
procnto
        ↓
IFS Startup Script
        ↓
io-sock
pci-server
devb
sshd
...
```

---

## 3.2 startup-x86

### 정의

QNX의 x86 플랫폼용 Startup Program

커널이 아니다.

부트로더 다음 단계에서 실행된다.

---

### 역할

```text
CPU 초기화
RAM 초기화
MMU 초기화
Interrupt 초기화
Timer 초기화
```

---

### 실제 수행 과정

```text
CPU 확인
 ↓
RAM 확인
 ↓
인터럽트 설정
 ↓
페이지 테이블 생성
 ↓
System Page 생성
 ↓
procnto 실행
```

---

### 왜 가장 먼저 실행되는가?

IFS 내부 구조가 다음과 같기 때문이다.

```text
ifs.bin

┌─────────────────────┐
│ startup-x86         │
├─────────────────────┤
│ procnto             │
├─────────────────────┤
│ libc                │
├─────────────────────┤
│ startup script      │
└─────────────────────┘
```

QEMU는

```bash
-kernel ifs.bin
```

으로 부팅한다.

부트로더는 IFS의 첫 엔트리인 startup-x86을 실행한다.

---

## 3.3 procnto

### 정의

QNX의 핵심 실행 파일

```text
Microkernel
+
Process Manager
```

가 하나로 합쳐져 있다.

---

### 역할

```text
Scheduler
IPC
Memory Manager
Process Manager
Thread Manager
Signal Manager
```

---

### Linux와 비교

Linux

```text
Kernel
 ↓
init(systemd)
```

QNX

```text
procnto
```

하나에 통합되어 있다.

---

## 3.4 Startup Script

### 역할

부팅 후 초기 서비스를 실행한다.

예)

```text
slogger2
pci-server
io-sock
devb
sshd
```

---

### Linux 대응 개념

```text
systemd service
```

와 비슷한 위치에 해당한다.

---

# 4. QNX 아키텍처

---

## 4.1 Linux vs QNX

### Linux

대부분의 기능이 Kernel Space에 존재

```text
Kernel
 ├─ Scheduler
 ├─ Memory Manager
 ├─ IPC
 ├─ Network Stack
 ├─ Filesystem
 └─ Drivers
```

---

### QNX

커널은 최소 기능만 유지

```text
procnto
 ├─ Scheduler
 ├─ IPC
 ├─ Memory Manager
 └─ Thread Manager
```

커널 외부

```text
io-sock
pci-server
devb
screen
```

---

### 핵심 차이

Linux

```text
기능 대부분이 Kernel 안
```

QNX

```text
기능 대부분이 User Space 프로세스
```

---

## 4.2 왜 드라이버를 커널 밖에 두는가?

### Linux

```text
Driver Crash
 ↓
Kernel Panic 가능
```

---

### QNX

```text
Driver Crash
 ↓
해당 프로세스 종료
 ↓
재시작 가능
```

---

### 장점

```text
Fault Isolation
높은 안정성
높은 유지보수성
안전 인증 용이
```

자동차, 철도, 항공 분야에서 QNX가 많이 사용되는 이유 중 하나이다.

---

# 5. VM 생성 실습

## VM 생성

```bash
mkdir ~/qnx_vm
cd ~/qnx_vm

mkqnximage \
    --type=qemu \
    --arch=x86_64 \
    --ssh-ident=none
```

---

## bridge-utils 설치

```bash
sudo apt update
sudo apt install bridge-utils
```

---

## VM 실행

```bash
mkqnximage --run
```

정상 부팅 시

```text
Startup complete
QNX noname 8.0.0
```

출력 확인

---

## VM IP 확인

QNX 내부

```bash
ifconfig -a
```

예)

```text
vtnet0
172.31.1.247
```

---

호스트 PC

```bash
mkqnximage --getip
```

예)

```text
172.31.1.247
```

---

## SSH 서버 확인

QNX 내부

```bash
pidin | grep sshd
```

예)

```text
system/bin/sshd
```

---

## SSH 접속

호스트 PC

```bash
ssh root@172.31.1.247
```

---

# 6. 이번 실습에서 확인한 핵심 사실

QNX Software Center에서

```text
QNX 8.0 SDK
```

만 설치했음에도

```text
mkqnximage
mkifs
startup-x86
procnto
IFS Build Templates
QEMU Integration
```

이 모두 포함되어 있었다.

즉

```text
QNX 8.0 SDK
 ├─ mkqnximage
 ├─ mkifs
 ├─ startup-x86
 ├─ procnto
 ├─ x86_64 runtime
 ├─ IFS templates
 └─ QEMU integration
```

구성이 기본 제공되므로 별도 BSP 없이 QEMU 기반 x86_64 QNX VM 생성이 가능했다.
