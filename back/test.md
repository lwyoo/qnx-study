# QNX SDK 구조와 부팅 아키텍처 정리

## 개요

본 문서는 QNX 8.0 SDK를 이용하여 QEMU 기반 QNX VM을 생성하고 학습하는 과정에서 이해해야 하는 핵심 개념을 정리한 문서이다.

주요 내용은 다음과 같다.

* Host 와 Target 구조
* IFS(Image File System)
* mkifs
* mkqnximage
* startup-x86
* procnto
* Linux 와 QNX 아키텍처 차이
* QNX 부팅 과정

---

# 1. Host 와 Target 의 구분

QNX SDK는 크게 Host 와 Target 영역으로 구분된다.

## Host

개발 PC(Ubuntu 등)에서 실행되는 프로그램들이다.

즉,

> 타겟용 프로그램을 만들기 위한 개발 도구들의 집합

이라고 볼 수 있다.

예시:

```text
Host

qcc
mkifs
mkqnximage
gdb
Momentics IDE
```

대표 위치:

```bash
~/qnx800/host/
```

예:

```bash
~/qnx800/host/linux/x86_64/usr/bin/qcc
~/qnx800/host/linux/x86_64/usr/bin/mkifs
```

---

## Target

실제 QNX 시스템에서 실행되는 파일들이다.

실제 보드 또는 VM 내부에 들어가는 런타임 구성 요소라고 생각하면 된다.

예시:

```text
Target

startup-x86
procnto
libc.so
devb-*
io-sock
사용자 프로그램
```

대표 위치:

```bash
~/qnx800/target/
```

예:

```bash
~/qnx800/target/qnx/x86_64/boot/sys/startup-x86
~/qnx800/target/qnx/x86_64/boot/sys/procnto-smp-instr
```

---

# 2. IFS (Image File System)

IFS는 QNX가 부팅 직후 사용하는 최소 실행 환경이다.

정확히는

```text
Initial File System
```

의 약자이다.

하지만 실제 개발 현장에서는

```text
Boot Image
```

라고 부르는 경우가 많다.

---

## 왜 필요한가?

컴퓨터가 전원을 켜면 처음에는

```text
/
├── bin
├── lib
├── usr
└── data
```

같은 파일시스템을 읽을 수 없다.

왜냐하면 아직

```text
디스크 드라이버
파일시스템 드라이버
USB 드라이버
```

등이 실행되지 않았기 때문이다.

즉,

```text
부팅 직후

디스크 접근 불가
USB 접근 불가
네트워크 접근 불가
```

상태이다.

---

## 해결 방법

QNX는 부팅에 필요한 최소 구성 요소를 하나의 이미지로 묶는다.

```text
startup
+
kernel
+
필수 라이브러리
+
필수 서비스
```

이 묶음이 IFS이다.

---

## ifs.bin 안에는 무엇이 들어있는가?

예시:

```text
ifs.bin

├── startup-x86
├── procnto-smp-instr
├── libc.so
├── slogger2
├── devc-ser8250
├── sh
├── ls
├── pidin
└── startup script
```

실제 시스템에서는 더 많은 파일이 포함된다.

---

## IFS와 ISO의 차이

ISO:

```text
CD/DVD 파일시스템 이미지
```

예:

```text
ubuntu.iso
```

IFS:

```text
QNX 부팅 이미지
```

예:

```text
ifs.bin
```

---

# 3. mkifs

## 역할

mkifs는 Buildfile을 이용하여 IFS 이미지를 생성하는 도구이다.

위치:

```bash
~/qnx800/host/linux/x86_64/usr/bin/mkifs
```

---

## 동작 과정

```text
ifs.build
    ↓
mkifs
    ↓
startup-x86
procnto
libraries
startup script
수집
    ↓
ifs.bin 생성
```

---

## Buildfile 예시

```text
startup-x86
procnto-smp-instr

/bin/sh
/bin/ls
/bin/pidin
```

위와 같이 정의되어 있으면

```text
startup-x86
+
procnto
+
sh
+
ls
+
pidin
```

을 포함한 IFS가 생성된다.

---

## mkifs가 수행하는 일

단순 압축기가 아니다.

```text
파일 수집
파일 배치
권한 설정
링크 생성
부팅 스크립트 생성
IFS 생성
```

을 수행한다.

---

# 4. mkqnximage

## 역할

QNX VM 또는 타겟 이미지를 생성하는 상위 도구이다.

---

## 내부 동작

```text
mkqnximage

├─ ifs.build 생성/수정
├─ mkifs 호출
├─ ifs.bin 생성
├─ disk image 생성
├─ filesystem 생성
├─ network 설정
├─ qemu 옵션 생성
└─ VM 실행
```

---

## 관계

```text
mkifs
=
IFS 생성기
```

```text
mkqnximage
=
전체 VM 생성기
```

---

## VM 생성 예제

```bash
mkdir ~/qnx_vm
cd ~/qnx_vm

mkqnximage \
    --type=qemu \
    --arch=x86_64 \
    --ssh-ident=none
```

생성 결과:

```text
local/
output/
```

---

## 실행

```bash
mkqnximage --run
```

---

# 5. QNX 부팅 과정

## Linux

```text
Bootloader
 ↓
Kernel
 ↓
init
 ↓
systemd
 ↓
서비스 실행
```

---

## QNX

```text
BIOS / UEFI
 ↓
Bootloader
 ↓
IFS 로드
 ↓
startup-x86
 ↓
procnto
 ↓
IFS Startup Script
 ↓
slogger2
 ↓
pci-server
 ↓
devb
 ↓
io-sock
 ↓
sshd
 ↓
사용자 프로그램
```

---

# 6. ifs.build

## 정의

IFS에 포함할 파일과 부팅 순서를 정의하는 Buildfile이다.

즉,

```text
파일 목록
+
초기 실행 스크립트
```

라고 생각하면 된다.

---

## 파일 포함

```text
startup-x86
procnto-smp-instr

libc.so
libm.so

/bin/sh
/bin/ls
/bin/pidin
```

---

## 부팅 스크립트

```text
[type=script] startup-script = {
    slogger2
    pci-server
    io-sock
    sshd
}
```

부팅 후 실행 순서를 정의한다.

---

# 7. startup-x86

## 정의

startup-x86은 커널이 아니다.

Bootloader와 procnto 사이를 연결하는 초기화 프로그램이다.

---

## 역할

```text
CPU 초기화
메모리 초기화
인터럽트 초기화
타이머 초기화
페이지 테이블 구성
```

---

## 부팅 흐름

```text
BIOS
 ↓
startup-x86
 ↓
procnto
```

---

## 왜 가장 먼저 실행되는가?

mkifs는 IFS를 아래와 같이 구성한다.

```text
ifs.bin

┌─────────────────┐
│ startup-x86     │
├─────────────────┤
│ procnto         │
├─────────────────┤
│ libraries       │
├─────────────────┤
│ startup script  │
└─────────────────┘
```

QEMU는

```bash
-kernel ifs.bin
```

형태로 실행한다.

BIOS가 IFS를 메모리에 적재한 뒤 첫 엔트리인 startup-x86을 실행한다.

---

# 8. procnto

## 정의

procnto는

```text
Microkernel
+
Process Manager
```

를 하나의 실행 파일로 통합한 QNX의 핵심 시스템 프로세스이다.

---

## procnto가 담당하는 기능

```text
Scheduler
Memory Manager
Thread Manager
Signal Manager
Process Manager
Message Passing IPC
Interrupt Dispatch
```

---

## procnto가 담당하지 않는 기능

```text
Network
 → io-sock

Storage
 → devb-*

PCI
 → pci-server

USB
 → io-usb

Graphics
 → screen
```

---

# 9. Linux 와 QNX 아키텍처 차이

## Linux

Kernel Space

```text
Linux Kernel

├─ Scheduler
├─ Memory Manager
├─ VFS
├─ Network Stack
├─ USB Driver
├─ Disk Driver
└─ IPC
```

User Space

```text
systemd
sshd
bash
application
```

---

## QNX

Microkernel

```text
procnto

├─ Scheduler
├─ IPC
├─ Thread Manager
├─ Process Manager
└─ Memory Manager
```

User Space

```text
pci-server
devb
io-sock
screen
sshd
application
```

---

# 10. 왜 드라이버를 커널 밖에 두는가?

Linux

```text
Network Driver Bug
 ↓
Kernel Panic
 ↓
전체 시스템 다운
```

QNX

```text
io-sock Bug
 ↓
io-sock 종료
 ↓
커널은 계속 동작
```

---

## 목적

```text
Fault Isolation
```

즉,

```text
장애를 시스템 전체로 전파하지 않는다.
```

는 것이 핵심 목표이다.

---

# 11. 실제 확인한 파일들

startup-x86

```bash
find ~/qnx800 -name startup-x86
```

결과:

```text
~/qnx800/target/qnx/x86_64/boot/sys/startup-x86
```

---

procnto

```bash
find ~/qnx800 -name "procnto*"
```

결과:

```text
~/qnx800/target/qnx/x86_64/boot/sys/procnto-smp-instr
```

---

mkifs

```bash
find ~/qnx800 -name mkifs
```

결과:

```text
~/qnx800/host/linux/x86_64/usr/bin/mkifs
```

---

ifs.build

```bash
find ~/qnx800 -name ifs.build
```

결과:

```text
~/qnx800/host/common/mkqnximage/inputs/ifs.build
```

---

# 요약

```text
Host
 └─ qcc, mkifs, mkqnximage

Target
 └─ startup-x86, procnto, libc

ifs.build
 └─ IFS 구성 정의

mkifs
 └─ ifs.build → ifs.bin

startup-x86
 └─ 하드웨어 초기화

procnto
 └─ QNX Microkernel

mkqnximage
 └─ VM 생성 및 실행

QNX 특징
 └─ 대부분의 드라이버와 서비스가 User Space 프로세스
```
