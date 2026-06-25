# QNX 8.0 SDK 기반 QEMU VM 구축 및 부팅 구조 분석

## 사전 확인

QNX Software Center에서 QNX 8.0 SDK를 설치한다.

QNX VM 생성에 필요한 x86_64 타겟 파일과 IFS 생성 도구가 설치되어 있는지 확인한다.

---

## 1. Startup 바이너리 확인

QNX는 일반적인 Linux 커널처럼 단일 커널 이미지를 부팅하지 않는다.

부팅 시 가장 먼저 실행되는 Startup 프로그램이 CPU와 메모리를 초기화한 후 Neutrino 마이크로커널(procnto)을 실행한다.

### 확인 명령어

```bash
find ~/qnx800/target -type f | grep startup
```

### 예시

```
/home/ldg/qnx800/target/qnx/x86_64/boot/sys/startup-x86
/home/ldg/qnx800/target/qnx/x86_64/boot/sys/startup-apic
```

### 파일 역할

| 파일 | 역할 |
|------|------|
| `startup-x86` | x86_64 PC/QEMU용 초기 부트 코드 |
| `startup-apic` | APIC 기반 SMP 환경용 Startup |

**특히 `startup-x86` 파일이 존재해야 x86_64 QNX VM 이미지를 생성할 수 있다.**

---

## 2. Neutrino 마이크로커널 확인

QNX의 실제 운영체제 커널은 `procnto`이다.

### 확인 명령어

```bash
find ~/qnx800 -name "procnto*"
```

### 예시

```
/home/ldg/qnx800/target/qnx/x86_64/boot/sys/procnto-smp-instr
```

### 파일 역할

| 파일 | 역할 |
|------|------|
| `procnto-smp-instr` | QNX Neutrino 마이크로커널 |
| `procnto-smp-instr-safety` | Safety 인증용 커널(설치 시 포함될 수 있음) |

### 부팅 과정

```
startup-x86
    ↓
procnto
    ↓
드라이버 및 서비스 실행
```

---

## 3. mkifs 확인

QNX는 IFS(Initial File System)를 생성하여 부팅한다.

IFS 생성기는 `mkifs`이다.

### 확인 명령어

```bash
find ~/qnx800 -name mkifs
```

### 예시

```
/home/ldg/qnx800/host/linux/x86_64/usr/bin/mkifs
```

### 역할

```
Buildfile
    ↓
mkifs
    ↓
ifs.bin 생성
```

**`ifs.bin`은 QNX 부팅 이미지이다.**

---

## 4. IFS(Buildfile) 템플릿 확인

### 확인 명령어

```bash
find ~/qnx800 -name "*.build"
```

### 예시

```
/home/ldg/qnx800/host/common/mkqnximage/inputs/ifs.build
/home/ldg/qnx800/host/common/mkqnximage/inputs/system.build
/home/ldg/qnx800/host/common/mkqnximage/inputs/data.build
/home/ldg/qnx800/host/common/mkqnximage/inputs/boot_part.build
/home/ldg/qnx800/host/common/mkqnximage/inputs/installer_part.build
```

### Buildfile의 역할

Buildfile은 IFS에 포함될 구성 요소를 정의한다.

**예:**

```
startup-x86
procnto-smp-instr
devc-ser8250
io-sock
sshd
sh
```

이 정보를 기반으로 `mkifs`가 부팅 이미지를 생성한다.

**특히 `ifs.build` 파일이 존재하면 `mkqnximage`가 QNX Initial File System(IFS)을 생성할 수 있다.**

---

## 5. QEMU 통합 기능 확인

### 확인 명령어

```bash
find ~/qnx800/host/common/mkqnximage/qemu
```

### 예시

```
runimage
stopimage
qemu_options
snippets/
```

### 파일 역할

| 파일 | 역할 |
|------|------|
| `runimage` | QEMU 실행 |
| `stopimage` | QEMU 종료 |
| `qemu_options` | QEMU 실행 옵션 생성 |
| `snippets` | VM 구성 템플릿 |

**QEMU 관련 파일이 존재하면 `mkqnximage`를 통해 VM 생성이 가능하다.**

---

## VM 생성에 필요한 핵심 구성 요소

실제로 VM 생성에 필요한 최소 구성 요소는 다음과 같다.

```
startup-x86
    ↓
procnto
    ↓
mkifs
    ↓
ifs.build
    ↓
mkqnximage
    ↓
ifs.bin 생성
    ↓
QEMU 부팅
```

### 각 구성 요소 역할

| 구성 요소 | 역할 |
|----------|------|
| `startup-x86` | CPU 초기화 |
| `procnto` | QNX 마이크로커널 |
| `mkifs` | 부팅 이미지 생성 |
| `ifs.build` | 이미지 구성 정의 |
| `mkqnximage` | VM 자동 생성 및 관리 |
| `QEMU` | 가상 머신 실행 |

---

## VM 생성 및 파일 구조

### VM 생성 명령어

```bash
mkdir ~/qnx_vm
cd ~/qnx_vm
mkqnximage \
    --type=qemu \
    --arch=x86_64 \
    --ssh-ident=none
```

### 생성 결과

```
local/
output/
```

### 디렉토리 구조

#### `local/` - 사용자 설정 저장

```
local/
 ├── options
 ├── misc_files
 └── ...
```

#### `output/` - QNX 부팅 이미지

```
output/
 └── ifs.bin
```

**`ifs.bin` 구성:**
- `startup-x86`
- `procnto`
- 드라이버
- 초기 스크립트

> QEMU는 다음과 같이 직접 부팅한다:
> ```bash
> qemu-system-x86_64 -kernel output/ifs.bin
> ```

### 가상 디스크 파일

**`disk-qemu.vmdk`**

- 가상 디스크 파일
- `/data` 디렉토리에 사용자 파일, 설정 파일, 추가 패키지 등의 영구 저장 공간 역할

### 빌드 결과물 저장

```
output/
 ├── ifs.bin
 ├── disk-qemu.vmdk
 ├── build/
 └── ...
```

#### 주요 파일

| 파일 | 설명 |
|------|------|
| `ifs.bin` | QNX Initial File System 이미지 |
| `disk-qemu.vmdk` | QEMU 가상 디스크 파일 |
| `build/` | 빌드 디렉토리 |
| `build/qemu_options` | QEMU 실행 옵션 |

### mkqnximage 옵션 설명

| 옵션 | 설명 |
|------|------|
| `--type=qemu` | QEMU VM 생성 |
| `--arch=x86_64` | 아키텍처 설정 |
| `--ssh-ident=none` | SSH 인증 설정 |

---

## VM 운영

### 네트워크 유틸리티 설치

```bash
sudo apt update
sudo apt install bridge-utils
```

### VM 실행

```bash
mkqnximage --run
```

**정상 부팅 시 출력:**

```
Startup complete
QNX noname 8.0.0 x86pc x86_64
```

### VM 종료

```bash
# 방법 1: QEMU 콘솔에서
Ctrl + A
X

# 방법 2: 명령어 사용
mkqnximage --stop
```

**정상 종료 시 출력:**

```
Startup complete
QNX noname 8.0.0 x86pc x86_64
```

---

## 네트워크 확인

### QNX VM 내부 - 네트워크 인터페이스 확인

```bash
ifconfig -a
```

**예시:**

```
vtnet0
inet 172.31.1.247
```

### SSH 서버 확인

```bash
pidin | grep sshd
```

**예시:**

```
system/bin/sshd
```

### 호스트에서 VM IP 확인

```bash
mkqnximage --getip
```

**예시:**

```
172.31.1.247
```

### SSH 접속

```bash
ssh root@172.31.1.247
```

---

## 확인된 사실

이번 테스트 결과 QNX Software Center에서 QNX 8.0 SDK만 설치해도 아래 구성 요소가 모두 포함되어 있었다.

```
QNX 8.0 SDK
 ├─ startup-x86
 ├─ procnto
 ├─ mkifs
 ├─ mkqnximage
 ├─ x86_64 target runtime
 ├─ IFS templates
 └─ QEMU integration
```

**따라서 별도의 BSP, Virtual Machine Package 또는 추가 VM SDK 설치 없이 QEMU 기반 x86_64 QNX 개발 환경을 구축할 수 있었다.**

이 환경은 C++ 애플리케이션 개발 및 Python 기반 개발 환경 구축의 시작점으로 사용할 수 있다.