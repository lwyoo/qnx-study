# QNX 8.0 Learning Roadmap

이 문서는 QNX 학습의 전체 진행 상황을 관리하기 위한 로드맵이다.

각 단계의 체크박스를 완료하면서 해당 문서를 읽고 실습을 진행한다.

---

# Phase 0. Development Environment

개발 환경 구축 및 SDK 이해

- [x] SDK 설치
- [x] SDK 구성 확인
  - 📄 `01_Environment/01_sdk_validation.md`

- [x] SDK 구조 및 부팅 과정 이해
  - 📄 `01_Environment/02_sdk_structure_and_boot.md`

- [x] QEMU VM 구축
  - 📄 `01_Environment/03_qemu_vm_setup.md`

---

# Phase 1. QNX Architecture

QNX의 설계 철학과 운영 방식 이해

- [x] QNX OS Philosophy
  - 📄 `02_Architecture/01_qnx_os_philosophy.md`

- [ ] Linux vs QNX Architecture
  - 📄 `02_Architecture/02_linux_vs_qnx.md`

---

# Phase 2. Process & IPC

QNX의 핵심 기능인 Message Passing 학습

- [ ] Process & Thread
  - 📄 `02_Architecture/03_process_thread.md`

- [ ] Message Passing
  - 📄 `02_Architecture/04_message_passing.md`

- [ ] Pulse
  - 📄 `02_Architecture/05_pulse.md`

---

# Phase 3. System Programming

사용자 공간 서비스 개발

- [ ] Daemon
  - 📄 `03_System/01_daemon.md`

- [ ] slog2 Logging
  - 📄 `03_System/02_slog2.md`

- [ ] Resource Manager
  - 📄 `03_System/03_resource_manager.md`

- [ ] TCP/IP Programming
  - 📄 `03_System/04_network.md`

---

# Phase 4. Real-Time

QNX 실시간 기능 학습

- [ ] Scheduling
  - 📄 `04_Realtime/01_scheduler.md`

- [ ] Memory Management
  - 📄 `04_Realtime/02_memory.md`

---

# Final Goal

최종적으로 다음 구조를 직접 구현할 수 있는 수준을 목표로 한다.

```
Sensor Service
        │
        ▼
Message Passing
        │
        ▼
Gateway Service
        │
        ▼
TCP/IP
```

구현 항목

- IPC
- Multi-thread
- Logging
- Resource Manager
- Real-time Scheduling
- Modern C++

---

# Progress

| Phase | Status |
|--------|--------|
| Environment | ✅ |
| Architecture | 🟡 |
| Process & IPC | ⬜ |
| System Programming | ⬜ |
| Real-Time | ⬜ |