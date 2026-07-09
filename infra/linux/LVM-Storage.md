---
tags:
  - linux
  - lvm
  - storage
  - disk
created: 2026-06-16
---

# LVM — 논리 볼륨 관리

> [!summary] 한 줄 요약
> LVM(Logical Volume Manager)은 물리 디스크를 추상화해 **온라인 확장, 스냅샷, 스트라이핑**을 제공한다. 베어메탈/VM 서버에서 디스크 유연하게 관리하는 표준 방식.

---

## 1. LVM 계층 구조

```
[물리 디스크]
/dev/sda   /dev/sdb   /dev/sdc
    │           │           │
    └─────PV(Physical Volume)────┘
              │
         VG(Volume Group)  ← vg_data
              │
    ┌─────────┼─────────┐
   LV       LV         LV       ← LV(Logical Volume)
  /data    /backup    /logs
```

---

## 2. 초기 설정

```bash
# 1. 물리 볼륨(PV) 생성
pvcreate /dev/sdb /dev/sdc
pvs                           # PV 목록
pvdisplay /dev/sdb            # 상세 정보

# 2. 볼륨 그룹(VG) 생성
vgcreate vg_data /dev/sdb /dev/sdc   # 두 디스크를 하나의 풀로
vgs                                   # VG 목록
vgdisplay vg_data

# 3. 논리 볼륨(LV) 생성
lvcreate -L 50G  -n lv_data   vg_data   # 고정 크기
lvcreate -l 80%VG -n lv_data  vg_data   # VG의 80%
lvcreate -l 100%FREE -n lv_logs vg_data # 남은 전부
lvs                                      # LV 목록

# 4. 파일시스템 생성 & 마운트
mkfs.xfs  /dev/vg_data/lv_data          # XFS (대용량 파일, 병렬 I/O 강함)
mkfs.ext4 /dev/vg_data/lv_logs          # ext4 (범용)

mkdir /data /logs
mount /dev/vg_data/lv_data  /data
mount /dev/vg_data/lv_logs  /logs

# /etc/fstab 영구 마운트
echo "/dev/vg_data/lv_data /data xfs defaults 0 0" >> /etc/fstab
```

---

## 3. 온라인 확장 (무중단)

```bash
# 방법 1: 새 디스크 추가해서 VG 확장
pvcreate /dev/sdd
vgextend vg_data /dev/sdd      # VG에 새 PV 추가
vgs                             # Free 용량 늘어남 확인

# 방법 2: LV 확장 (VG에 여유 공간 필요)
lvextend -L +20G /dev/vg_data/lv_data          # 20GB 추가
lvextend -l +100%FREE /dev/vg_data/lv_data     # 여유 전부 사용
lvextend -L 100G /dev/vg_data/lv_data          # 절대 크기로 지정

# 파일시스템도 확장 (LV 확장만으론 fs가 안 커짐)
xfs_growfs /data               # XFS (마운트 상태에서 온라인 확장)
resize2fs /dev/vg_data/lv_data # ext4

# 확인
df -h /data
lvdisplay /dev/vg_data/lv_data
```

---

## 4. 스냅샷 — 백업 & 테스트

```bash
# 스냅샷 생성 (COW: Copy-On-Write, 변경분만 저장)
lvcreate -L 5G -s -n lv_data_snap /dev/vg_data/lv_data
#              ↑ snapshot 플래그
# ⚠ snap 크기는 스냅샷 기간 동안 예상 변경량보다 크게 설정

lvs    # Origin, Snap%, 사용률 확인

# 스냅샷 마운트 (읽기 전용 백업)
mkdir /mnt/snap
mount -o ro /dev/vg_data/lv_data_snap /mnt/snap
# → /mnt/snap은 스냅샷 시점의 /data 복사본

# 백업 후 스냅샷 제거
umount /mnt/snap
lvremove /dev/vg_data/lv_data_snap

# 스냅샷으로 원본 복구 (롤백)
lvconvert --merge /dev/vg_data/lv_data_snap
# 병합 후 재부팅 필요 (마운트된 LV인 경우)
```

---

## 5. 씬 프로비저닝 (Thin Provisioning)

```bash
# Thin Pool 생성
lvcreate -L 100G --thinpool vg_data/tp_pool vg_data

# Thin Volume — 실제 공간보다 더 크게 할당 가능 (초과 구독)
lvcreate -V 500G --thin -n lv_thin1 vg_data/tp_pool
lvcreate -V 500G --thin -n lv_thin2 vg_data/tp_pool
# 실제 100GB 풀에 500GB + 500GB = 초과 구독, 실 사용량으로만 공간 소비

lvs -a  # Thin Pool 사용률 모니터링 필수
```

---

## 6. 모니터링 & 문제 해결

```bash
# 전체 LVM 현황
pvs && vgs && lvs

# VG 여유 공간
vgdisplay vg_data | grep "Free  PE"

# 스냅샷 사용률 (100% 채우면 스냅샷 무효화)
lvs -o name,snap_percent /dev/vg_data

# 디스크 장애: PV 교체
pvmove /dev/sdb                    # sdb 데이터를 다른 PV로 이동
vgreduce vg_data /dev/sdb          # VG에서 제거
pvremove /dev/sdb                  # PV 해제

# RAID-like 미러링 (LVM stripping)
lvcreate -L 50G -m1 -n lv_mirror vg_data   # 미러 1개 (2중화)
```

---

## 7. ext4 vs XFS 선택

| | ext4 | XFS |
|--|------|-----|
| 최대 파일 크기 | 16TB | 8EB |
| 최대 볼륨 크기 | 1EB | 8EB |
| 온라인 축소 | 가능 | **불가** (확장만) |
| 온라인 확장 | 가능 | 가능 |
| 대용량 파일 I/O | 보통 | 강함 |
| 메타데이터 성능 | 보통 | 강함 |
| 권장 | 범용, 소용량 | DB, 로그, AI 데이터 |

---

## 8. 관련
- [[Systemd]] · [[Sysstat]] · [[Kernel-Parameters]]
- [[Rack-Mount-Server]] — RAID 컨트롤러와 병행 사용 고려
