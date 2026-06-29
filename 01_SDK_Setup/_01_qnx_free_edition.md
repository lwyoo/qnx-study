# QNX 8.0 Free Edition 다운로드 및 라이선스 활성화

## 1. QNX Free Edition이란?

QNX 8.0 Free Edition은 개발자 학습과 비상업적 용도로 제공되는 무료 버전입니다.

### 특징

| 항목 | 설명 |
|------|------|
| **라이선스** | 비상업적 용도 전용 (학습, 개인 프로젝트) |
| **기능** | QNX SDP(Software Development Platform) 전체 포함 |
| **유효기간** | 1년 (갱신 가능) |
| **활성화** | MyQNX 계정 등록 필요 |
| **지원** | 커뮤니티 포럼 |

---

## 2. 설치 전 요구사항

### 시스템 요구사항

```
OS        : Linux (Ubuntu 20.04+), Windows WSL2, macOS
Memory    : 최소 8GB (권장 16GB)
Disk      : 최소 20GB (모든 타겟 포함 시 50GB+)
Internet  : 설치 중 필수
```

### 필요한 패키지

**Ubuntu/Debian 기반:**

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential \
  git \
  python3 \
  libfuse-dev \
  libssl-dev \
  pkg-config
```

**다운로드 도구:**

- QNX Software Center (권장)
- 또는 웹 브라우저에서 직접 다운로드

---

## 3. MyQNX 계정 등록 (중요!)

### 단계 1: MyQNX 사이트 접속

```
https://www.qnx.com/account/register
```

### 단계 2: 계정 생성

- 이메일 주소 입력
- 비밀번호 설정
- 개인 정보 입력 (회사명, 용도 등)
- **"Non-Commercial Use"** 선택 필수

### 단계 3: 라이선스 활성화

1. MyQNX 대시보드 로그인
2. "Download" 섹션에서 QNX 8.0 선택
3. "Free Edition" 클릭
4. 라이선스 키 생성 (자동 발급)

### 라이선스 키 형식

```
License created successfully
Key ID: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
Email: your.email@example.com
```

---

## 4. 라이선스 설정

### 환경 변수 설정

라이선스 키를 환경에 등록합니다.

**bash 기반 (Linux/macOS):**

```bash
# ~/.bashrc 또는 ~/.zshrc에 추가
export QNX_HOST_VERSION=8.0.0
export QNX_LICENSE="<Your License Key>"

# 적용
source ~/.bashrc
```

**Windows PowerShell:**

```powershell
$env:QNX_LICENSE = "<Your License Key>"
```

### 라이선스 확인

```bash
# QNX 설치 후 확인 가능
$QNX_HOST/usr/bin/license_validate
```

---

## 5. 설치 옵션 선택

### 최소 설치 (권장: 학습용)

```
- QNX 8.0 Runtime
- x86_64 타겟 지원
- 기본 개발 도구
- 총 크기: ~15GB
```

### 전체 설치 (프로덕션 개발)

```
- 모든 타겟 아키텍처 (ARM, x86, PowerPC 등)
- 전체 라이브러리 및 드라이버
- 문서 및 샘플 코드
- 총 크기: ~50GB+
```

### 추천 학습용 설치

```
QNX 8.0 Free Edition
  ├── Runtime (필수)
  ├── x86_64 target (QNX 8.0용 QEMU 학습)
  ├── Documentation
  ├── Examples
  └── Development Tools
```

---

## 6. 설치 이후 검증

### 설치 디렉토리 확인

```bash
# QNX_HOST 환경 변수 확인
echo $QNX_HOST
# 출력 예: /opt/qnx800/host/linux/x86_64

# QNX_TARGET 환경 변수 확인
echo $QNX_TARGET
# 출력 예: /opt/qnx800/target/qnx

# 설치 구조 확인
ls -la $QNX_HOST/
```

### 라이선스 유효성 검증

```bash
# 라이선스 정보 확인
$QNX_HOST/usr/bin/printenv | grep QNX_LICENSE

# 개발 도구 테스트
$QNX_HOST/usr/bin/qcc --version
# 출력 예: QCC 8.3.0 (GCC 8.3.0)
```

---

## 7. 라이선스 갱신 및 연장

### Free Edition 라이선스 갱신

- **갱신 주기**: 1년
- **갱신 방법**: MyQNX 계정에서 자동 갱신
- **갱신 방식**: 이메일 알림 수신 후 클릭으로 갱신

### 상업용 라이선스로 전환 필요 시

```
https://www.qnx.com/contact/licensing
```

---

## 참고: 라이선스 정책 확인

### Non-Commercial 용도 정의

```
✅ 허용되는 용도
- 개인 학습 및 연구
- 오픈소스 기여
- 비영리 프로젝트
- 회사 내부 연구 (상용화 X)

❌ 금지되는 용도
- 상용 제품에 포함
- 수익 창출 프로젝트
- 클라이언트 제공
```

---

## 다음 단계

라이선스 활성화 후 `_02_qnx_software_center.md`에서 QNX Software Center를 통한 설치 과정을 진행합니다.
