# QNX 8.0 SDK 개발 환경 검증 가이드

## 📌 개요

QNX 8.0 SDK 설치 후 개발 환경이 제대로 구축되었는지 확인하는 단계별 가이드입니다.

| 항목 | 정보 |
|------|------|
| SDK 버전 | QNX SDP 8.0.4 |
| 호스트 OS | Ubuntu 24.04 (Linux x86_64) |
| 라이선스 | Non-commercial |

---

## 1️⃣ 환경 변수 설정

### 설정 명령어

```bash
source ~/qnx800/qnxsetup.sh
```

### 환경 변수 확인

```bash
echo $QNX_HOST
echo $QNX_TARGET
echo $MAKEFLAGS
```

### 예상 결과

```
QNX_HOST=/home/ldg/qnx800/host/linux/x86_64
QNX_TARGET=/home/ldg/qnx800/target/qnx
MAKEFLAGS=-I/home/ldg/qnx800/target/qnx/usr/include
```

### ⚠️ 주의사항

- **bash에서만 동작합니다**
  - zsh, fish 등 다른 쉘을 사용 중이면 bash로 전환 필요

#### 영구 설정 (권장)

`~/.bashrc` 끝에 다음을 추가하여 매번 수동 로드를 피할 수 있습니다:

```bash
# ~/.bashrc 끝에 추가
if [ -f ~/qnx800/qnxsetup.sh ]; then
    source ~/qnx800/qnxsetup.sh
fi
```

그 후 재시작:

```bash
source ~/.bashrc
```

---

## 2️⃣ SDK 버전 확인

### QCC (QNX 크로스 컴파일러) 버전

#### 명령어

```bash
qcc --version
```

#### 예상 결과

```
Toolchain:  gcc	Version:  12.2.0
```

### 경로 확인

#### 명령어

```bash
which qcc
```

#### 예상 결과

```
/home/ldg/qnx800/host/linux/x86_64/usr/bin/qcc
```

### ✅ 검증 포인트

- [ ] qcc가 PATH에 등록되어 있는가?
- [ ] Toolchain 버전이 gcc 12.2.0 이상인가?
- [ ] 경로가 SDK 디렉토리 내에 있는가?

---

## 3️⃣ Make 버전 확인

### Make 버전

#### 명령어

```bash
make --version | head -1
```

#### 예상 결과

```
GNU Make 4.4.1
```

### 경로 확인

#### 명령어

```bash
which make
```

#### 예상 결과

```
/home/ldg/qnx800/host/linux/x86_64/usr/bin/make
```

### ⚠️ 중요

QNX SDK의 make를 사용해야 합니다. 호스트 시스템의 make와 구별되는지 확인하세요:

```bash
# 호스트 시스템의 make 확인
ls -la /usr/bin/make

# QNX SDK의 make 확인
ls -la /home/ldg/qnx800/host/linux/x86_64/usr/bin/make
```

두 파일이 다른 위치에 있으면 정상입니다.

---

## 4️⃣ 환경 검증 체크리스트

### 필수 확인 항목

- [ ] qnxsetup.sh 실행 완료
- [ ] QNX_HOST 환경 변수 설정됨
- [ ] QNX_TARGET 환경 변수 설정됨
- [ ] MAKEFLAGS 환경 변수 설정됨
- [ ] qcc --version 실행 가능
- [ ] which qcc 경로 표시됨
- [ ] qcc 버전 12.2.0 이상
- [ ] make 버전 4.3 이상
- [ ] 라이선스 파일 위치: `~/.qnx/license.dat`

---

## 5️⃣ 첫 컴파일 테스트

### 프로젝트 디렉토리 생성

```bash
mkdir -p ~/qnx_projects/hello_test
cd ~/qnx_projects/hello_test
```

### Hello World 코드 작성

```c
// hello.c
#include <stdio.h>

int main() {
    printf("Hello QNX 8.0!\n");
    return 0;
}
```

### 컴파일

```bash
qcc -o hello hello.c
```

### 바이너리 검증

```bash
file ./hello
```

#### 예상 결과:

```
hello: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), 
dynamically linked, interpreter /usr/lib/ldqnx-64.so.2, 
BuildID[md5/uuid]=bc6d252113e245b39454b8052330d177, 
with debug_info, not stripped
```

### ⚠️ 주의

QNX 바이너리이므로 현재 Linux 호스트에서는 직접 실행 불가능합니다. QNX VM이나 에뮬레이터에서만 실행할 수 있습니다.

---

## 6️⃣ 문제 해결

### Q1: "qcc: command not found"

#### 원인
환경 변수가 설정되지 않음

#### 해결 방법

```bash
# 다시 환경 변수 로드
source ~/qnx800/qnxsetup.sh

# 또는 절대 경로로 직접 실행
~/qnx800/host/linux/x86_64/usr/bin/qcc --version
```

---

### Q2: "cannot find <sys/dispatch.h>"

#### 원인
QNX 헤더 파일을 찾을 수 없음

#### 해결 방법

```bash
# QNX_TARGET 환경 변수 확인
echo $QNX_TARGET

# 헤더 파일 존재 확인
ls -la $QNX_TARGET/usr/include/sys/dispatch.h
```

---

### Q3: bash 외 쉘에서 환경 변수가 작동하지 않음

#### 해결 방법

**zsh 사용 시:**
`~/.zshrc` 파일에 다음 추가:
```bash
source ~/qnx800/qnxsetup.sh
```

**fish 사용 시:**
`~/.config/fish/config.fish` 파일에 추가 (bash 호환성 필요)

#### 권장사항
개발 중에는 **bash 사용**을 권장합니다:

```bash
# bash로 전환
bash

# 현재 쉘 확인
echo $SHELL
```

---

## 📝 참고사항

- 모든 명령어는 QNX 8.0 SDK가 `~/qnx800/` 디렉토리에 설치되어 있다고 가정합니다.
- SDK 설치 경로가 다른 경우 경로를 적절히 수정하세요.
- Non-commercial 라이선스는 상업적 목적으로 사용할 수 없습니다.
- 문제 발생 시 QNX 공식 문서 및 커뮤니티 포럼을 참고하세요.
