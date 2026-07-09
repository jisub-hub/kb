---
tags:
  - infra
  - linux
  - moc
  - index
created: 2026-06-15
---

# Linux MOC

> 서버 운영에 필요한 Linux 시스템 설정과 튜닝.

## 파일 구성
- [[Ulimit]] — 프로세스 자원 제한 (nofile·nproc·stack), 왜 건드는지, 설정 방법
- [[Kernel-Parameters]] — sysctl 네트워크·파일시스템·메모리 파라미터 튜닝
- [[Ramdisk]] — tmpfs/ramfs 개념, 마운트, 활용 사례
- [[Sysstat]] — sar/iostat/mpstat/pidstat 사용법, 장애 사후 분석 패턴
- [[Systemd]] — unit 파일 작성, EnvironmentFile, Timer, journalctl 로그 조회
- [[LVM-Storage]] — PV/VG/LV 초기 설정, 온라인 확장, 스냅샷, ext4 vs XFS
- [[TCPDump-Wireshark]] — BPF 필터, 시나리오별 트러블슈팅, Wireshark 디스플레이 필터
- [[Process-Deployment]] — nohup·systemd Unit·Docker restart 비교, PID 관리, Compose를 systemd 서비스로

## 관련
- [[../network/Security-ACG-NSG]] · [[../ssh-cicd/SSH-SCP]] · [[../../programming/spring/java/JVM-Tuning]]
