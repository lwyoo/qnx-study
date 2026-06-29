# QNX Software Center를 통한 SDK 설치 및 관리

## 1. QNX Software Center 개요

QNX Software Center는 QNX 개발 환경 설치, 업데이트, 라이선스 관리를 위한 통합 도구입니다.

### 특징

- **GUI 기반 설치** - 명령어 없이 클릭으로 설치
- **네트워크 기반** - 인터넷 연결 필수
- **버전 관리** - 여러 QNX 버전 동시 설치 가능
- **자동 업데이트** - 보안 패치 및 기능 업데이트
- **라이선스 검증** - 설치 시 라이선스 확인

---

## 2. QNX Software Center 다운로드

### 다운로드 경로

```
https://www.qnx.com/download/qnx_software_center/
```

또는 MyQNX 대시보드에서:

```
MyQNX Dashboard → Downloads → QNX Software Center
```

### 플랫폼별 다운로드

| OS | 파일 | 크기 |
|----|------|------|
| Linux (x86_64) | `qnx-software-center-x.x.x-linux.run` | ~100MB |
| Windows | `qnx-software-center-x.x.x-windows.exe` | ~100MB |
| macOS | `qnx-software-center-x.x.x-macos.dmg` | ~100MB |

---

## 3. QNX Software Center 설치

### Linux에서 설치

```bash
# 1. 다운로드 폴더로 이동
cd ~/Downloads

# 2. 파일에 실행 권한 부여
chmod +x qnx-software-center-*.run

# 3. 설치 시작
./qnx-software-center-*.run
```

### GUI 설치 마법사

```
┌─ Welcome ─────────────┐
│                       │
│ "Next" 클릭           │
└───────────────────────┘
        ↓
┌─ Installation Path ───┐
│                       │
│ 기본값: $HOME/qnx800  │
│ (또는 /opt/qnx800)    │
└───────────────────────┘
        ↓
┌─ License ────────────┐
│                       │
│ MyQNX 계정으로 로그인 │
│ (또는 라이선스 키)    │
└───────────────────────┘
        ↓
┌─ Component Selection ─┐
│                       │
│ 설치할 타겟 선택      │
│ (x86_64 권장)         │
└───────────────────────┘
        ↓
┌─ Installing... ──────┐
│                       │
│ (10-30분 소요)        │
└───────────────────────┘
```

---

## 4. QNX Software Center 실행 및 초기 설정

### Software Center 실행

```bash
# 설치 완료 후 자동 실행되거나 수동 실행:
~/qnx800/qnx-software-center
# 또는
/opt/qnx800/qnx-software-center
```

### 로그인

1. Software Center 시작
2. "Sign In" 클릭
3. MyQNX 계정 정보 입력:
   - 이메일 주소
   - 비밀번호

### 라이선스 확인

```
Settings → License Information
→ "Active Licenses" 확인
→ "Non-Commercial" 표시 확인
```

---

## 5. SDK 컴포넌트 설치

### 기본 설치 구조

```
QNX 8.0
├── Runtime (필수)
├── Target Architectures
│   ├── x86_64 (권장)
│   ├── ARM (추가 가능)
│   ├── PowerPC (추가 가능)
│   └── MIPS (추가 가능)
├── Documentation
├── Examples
└── Developer Tools
```

### x86_64 타겟 설치 (학습용 권장)

1. Software Center에서 "Products" 탭 클릭
2. "QNX 8.0" 선택
3. "Select Components" 클릭
4. 다음 항목 선택:
   - ✓ QNX Neutrino 8.0 Runtime
   - ✓ x86_64 Architecture
   - ✓ Documentation
   - ✓ Examples
5. "Install" 버튼 클릭

### 설치 시간 및 크기

```
x86_64만 설치:
├── 설치 크기: ~12-15GB
├── 다운로드: 약 5-10분 (인터넷 속도에 따라)
├── 설치: 약 10-20분
└── 전체: 약 20-30분
```

---

## 6. 환경 변수 자동 설정

### 자동 설정 스크립트

QNX Software Center는 설치 완료 후 환경 설정 스크립트를 제공합니다.

```bash
# Bash 사용자
cat >> ~/.bashrc << 'BASHRC'
# QNX Environment Setup
if [ -f ~/qnx800/qnxenv-8.0.0.sh ]; then
    source ~/qnx800/qnxenv-8.0.0.sh
fi
BASHRC

# 적용
source ~/.bashrc
```

### 환경 변수 검증

```bash
# 설치 확인
echo "QNX_HOST: $QNX_HOST"
echo "QNX_TARGET: $QNX_TARGET"

# 예상 출력:
# QNX_HOST: /home/user/qnx800/host/linux/x86_64
# QNX_TARGET: /home/user/qnx800/target/qnx
```

---

## 7. 업데이트 및 추가 컴포넌트 설치

### Software Center에서 업데이트

```
Settings → Check for Updates
```

또는 자동 업데이트 설정:

```
Settings → Auto Update → Enable
```

### 추가 타겟 아키텍처 설치

1. Software Center "Products" 탭
2. "QNX 8.0" 아래 "Manage" 클릭
3. 추가 아키텍처 선택:
   - ARM
   - PowerPC
   - MIPS 등
4. "Install" 클릭

### 설치된 컴포넌트 확인

```bash
# Software Center에서 또는 명령어로:
ls -la $QNX_HOST/../
```

---

## 8. SDK 구조 확인

### 설치 완료 후 디렉토리 구조

```
~/qnx800/
├── host/
│   └── linux/x86_64/
│       ├── usr/bin/        # 개발 도구 (qcc, qde, mkifs 등)
│       ├── usr/lib/        # 호스트 라이브러리
│       └── etc/            # 설정 파일
├── target/
│   └── qnx/
│       ├── x86_64/         # x86_64 타겟 라이브러리
│       │   ├── lib/
│       │   ├── usr/lib/
│       │   └── boot/
│       └── common/         # 공통 헤더 파일
├── qnx-software-center    # GUI 도구
└── qnxenv-8.0.0.sh        # 환경 설정 스크립트
```

### 설치 검증

```bash
# 컴파일러 확인
$QNX_HOST/usr/bin/qcc --version

# 부팅 이미지 생성 도구 확인
$QNX_HOST/usr/bin/mkifs --version

# 라이브러리 확인
ls $QNX_TARGET/x86_64/lib/ | head
```

---

## 9. 문제 해결

### 라이선스 오류

```
Error: License validation failed
```

해결 방법:
1. MyQNX 계정 로그인 확인
2. Software Center에서 "Sign Out" → "Sign In" 다시 시도
3. 인터넷 연결 확인
4. 라이선스 만료 확인 (Settings → License Information)

### 설치 실패

```
Error: Download failed
```

해결 방법:
1. 인터넷 연결 확인
2. 디스크 여유 공간 확인 (최소 20GB)
3. Software Center 재시작
4. 설치 경로 권한 확인

### 환경 변수 미설정

```bash
# 수동 설정
export QNX_HOST=$HOME/qnx800/host/linux/x86_64
export QNX_TARGET=$HOME/qnx800/target/qnx
export PATH=$QNX_HOST/usr/bin:$PATH
```

---

## 다음 단계

SDK 설치 완료 후 `03_sdk_validation.md`에서 설치 검증을 진행합니다.
