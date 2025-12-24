# Raspberry Pi 5 – USB Auto Update System
(자동 롤백 + 헬스 체크 + udev 트리거)

라즈베리파이 5 + Raspberry Pi OS 환경에서  
**USB(또는 외장 SSD)를 꽂기만 하면 애플리케이션이 자동 업데이트**되고,  
문제가 생기면 **이전 버전으로 자동 롤백**되는 미니 프로젝트입니다.

> 현업에서 자주 쓰는 “안전한 업데이트 흐름(backup + pending + health + rollback)”을  
> 작은 예제(`my-app`)로 축소해서 구현/학습 가능한 형태로 정리했습니다.

---

## ✅ Features

- **USB 기반 앱 배포**
  - 파일시스템 라벨이 `UPDATE_USB`인 스토리지에
    `update/manifest.json` + `update/app.tar.gz`만 넣으면 자동 업데이트
  - 기본 타겟 디렉터리: `/opt/my-app`

- **버전 관리 & 중복 업데이트 방지**
  - `/var/lib/usb-updater/state.json`에 `current_version` 저장
  - `manifest.json`의 `version`과 비교해 **같은 버전이면 skip**

- **헬스 체크(Health Check) + pending 상태**
  - 새 버전 설치 직후 `pending=true`
  - 서비스 정상 확인 후 `usb-update-mark-healthy` 호출 → `pending=false`, `last_status="ok"`

- **자동 롤백(Auto Rollback)**
  - `pending=true` 상태에서 updater가 다시 실행되면(예: timer, 재부팅 후 실행 등)
    → 최신 백업에서 자동 복구(롤백)

- **수동 롤백(Manual Rollback)**
  - `usb-update-rollback last`로 최근 백업으로 복원
  - 특정 백업 디렉터리 이름으로도 롤백 가능

- **History Log 기록**
  - `/var/log/usb-updater-history.log`에 한 줄 요약 기록
    (timestamp / status / 버전 / 메시지)

- **udev + systemd 통합**
  - `UPDATE_USB` 라벨 스토리지를 꽂으면
    udev → `usb-updater-on-usb@.service` → `usb-updater-udev-wrapper` → `usb-updater`
  - 별도 명령 없이 자동 업데이트 수행

---

## 🧪 Tested Environment

- Raspberry Pi 5
- Raspberry Pi OS (Debian 기반, bookworm 계열)
- `/opt/my-app` 경로 사용
- USB 또는 외장 SSD (파일시스템 라벨: `UPDATE_USB`)

---

## 📁 Repository Structure

```text
Raspbery-pi5-USB-update/
├── README.md
├── scripts/
│   ├── usb-updater               # 메인 업데이트 스크립트
│   ├── usb-update-mark-healthy   # 헬스 승인(healthy mark)
│   ├── usb-update-rollback       # 수동 롤백
│   └── usb-updater-udev-wrapper  # udev에서 호출되는 wrapper
├── systemd/
│   ├── my-app.service
│   ├── usb-updater.service
│   ├── usb-updater.timer
│   └── usb-updater-on-usb@.service
├── udev/
│   └── 99-usb-updater.rules
└── examples/
    ├── example-my-app.sh
    └── example-manifest.json
```
💾 USB Update Package Format

라벨이 UPDATE_USB인 스토리지 안에 아래 구조로 준비합니다.

UPDATE_USB/
└── update/
    ├── manifest.json
    └── app.tar.gz

manifest.json 예시
{
  "version": "1.0.0",
  "description": "example update package",
  "sha256": "",
  "target": "/opt/my-app",
  "service": "my-app.service"
}


version : 버전 문자열 (중복 업데이트 방지에 사용)

description : 업데이트 설명(로그용)

sha256 : (옵션) 무결성 확인용 해시

target : 전개 대상 디렉터리 (예: /opt/my-app)

service : 업데이트 후 재시작할 systemd 서비스 (예: my-app.service)

app.tar.gz 만들기 예시
mkdir -p /tmp/app_v1.0.0
cp examples/example-my-app.sh /tmp/app_v1.0.0/my-app.sh
chmod +x /tmp/app_v1.0.0/my-app.sh

tar -C /tmp/app_v1.0.0 -czf app.tar.gz .
# 만든 app.tar.gz 를 UPDATE_USB/update/ 아래로 복사

🛠 Installation

Raspberry Pi OS 환경 + /opt/my-app + UPDATE_USB 라벨 기준으로 설명합니다.

1) 저장소 클론
cd ~
git clone https://github.com/nasi546/Raspbery-pi5-USB-update.git
cd Raspbery-pi5-USB-update

2) 스크립트 설치
sudo cp scripts/usb-updater \
        scripts/usb-update-mark-healthy \
        scripts/usb-update-rollback \
        scripts/usb-updater-udev-wrapper \
        /usr/local/sbin/

sudo chmod +x /usr/local/sbin/usb-*

3) systemd 유닛 설치
sudo cp systemd/*.service systemd/*.timer /etc/systemd/system/
sudo systemctl daemon-reload


샘플 유닛 가정:

앱 서비스: my-app.service

업데이트 서비스: usb-updater.service

주기 실행: usb-updater.timer

udev 템플릿: usb-updater-on-usb@.service

4) udev 룰 설치
sudo cp udev/99-usb-updater.rules /etc/udev/rules.d/
sudo udevadm control --reload


99-usb-updater.rules 핵심(요약):

ACTION=="add", SUBSYSTEM=="block", ENV{ID_FS_LABEL}=="UPDATE_USB", ENV{SYSTEMD_WANTS}="usb-updater-on-usb@%k.service"

5) 예제 앱 설치(선택)
sudo mkdir -p /opt/my-app
sudo cp examples/example-my-app.sh /opt/my-app/my-app.sh
sudo chmod +x /opt/my-app/my-app.sh

▶️ Enable & Start
sudo systemctl enable my-app.service
sudo systemctl enable usb-updater.service
sudo systemctl enable usb-updater.timer

sudo systemctl start my-app.service
sudo systemctl start usb-updater.timer

USB 라벨 설정(한 번만)
lsblk -f
# 예: /dev/sda1 이 UPDATE_USB용 파티션이라면(ext4)
sudo e2label /dev/sda1 UPDATE_USB

🚀 Usage
1) 자동 업데이트(추천)

USB/SSD에 UPDATE_USB 라벨 설정

update/manifest.json, update/app.tar.gz 배치

라즈베리파이에 꽂기

로그 확인:

sudo journalctl -u usb-updater.service -u usb-updater-on-usb@sda1.service --no-pager
tail -n 30 /var/log/usb-updater.log
tail -n 30 /var/log/usb-updater-history.log


상태 확인:

cat /var/lib/usb-updater/state.json
/opt/my-app/my-app.sh

2) 수동 실행
sudo usb-updater
# 또는 특정 마운트 경로 강제 지정
sudo usb-updater --mount-path /media/pi07/UPDATE_USB

3) 헬스 체크 승인(Healthy Mark)

앱이 정상 동작하는 것이 확인되면 아래를 실행합니다.

sudo /usr/local/sbin/usb-update-mark-healthy
cat /var/lib/usb-updater/state.json


샘플 my-app.service는 재기동 시 ExecStartPost로 위 스크립트를 호출하도록 구성할 수 있습니다.

4) 수동 롤백
# 백업 목록 보기
ls /opt/my-app-backups

# 가장 최근 백업으로 롤백
sudo usb-update-rollback last

# 특정 백업 디렉터리로 롤백
sudo usb-update-rollback my-app-20251218-105544

🧠 State & Rollback Logic
state.json 예시

/var/lib/usb-updater/state.json

{
  "current_version": "1.3.0",
  "previous_version": "1.2.0",
  "backup_dir": "/opt/my-app-backups/my-app-20251222-101503",
  "target_dir": "/opt/my-app",
  "pending": "false",
  "last_update": "2025-12-22 10:17:02",
  "last_status": "ok",
  "last_error": ""
}


current_version : 현재 설치된 버전

previous_version : 직전 버전

backup_dir : 롤백용 백업 경로

pending : "true"면 아직 헬스 체크 승인 전

last_status : "ok" | "pending" | "rollback" | "manual-rollback" | "skip" 등

last_error : 오류/롤백 사유

자동 롤백 조건(핵심)

pending == "true"

backup_dir가 존재

updater가 다시 실행됨(timer/재부팅/수동 실행 등)

→ backup_dir 내용으로 target_dir 복구
→ 버전 정보 복원 + pending="false", last_status="rollback"

🗂 Logs

메인 로그: /var/log/usb-updater.log

히스토리 로그: /var/log/usb-updater-history.log

상태 파일: /var/lib/usb-updater/state.json

백업 디렉터리: /opt/my-app-backups/

작업 디렉터리: /opt/my-app-updates/work/

🧯 Troubleshooting (Issues & Fixes)

문제 분석은 보통 아래 순서가 가장 빠릅니다.

라벨 확인: lsblk -f → 대상 파티션 라벨이 UPDATE_USB인지

udev 트리거 확인: usb-updater-on-usb@...service가 떴는지

파일 확인: /media/<USER>/UPDATE_USB/update/{manifest.json,app.tar.gz} 존재 여부

상태 확인: state.json의 pending/current_version/last_error

이력 확인: usb-updater-history.log의 skip/rollback 사유

A) “USB 꽂았는데 업데이트가 시작 자체를 안 함”

Symptom

usb-updater-on-usb@...service가 뜨지 않음

로그가 비어있음

Cause

라벨이 UPDATE_USB가 아님

udev 룰이 설치/리로드되지 않음

Fix

lsblk -f
sudo e2label /dev/sda1 UPDATE_USB

sudo cp udev/99-usb-updater.rules /etc/udev/rules.d/
sudo udevadm control --reload


Verify

sudo journalctl -u usb-updater-on-usb@sda1.service --no-pager

B) “서비스는 떴는데 manifest/app.tar.gz를 못 찾음”

Symptom

실행은 되는데 업데이트가 진행되지 않음

로그에 경로 관련 오류

Cause

USB 구조가 정확히 아래가 아님

UPDATE_USB/update/manifest.json

UPDATE_USB/update/app.tar.gz

Fix

UPDATE_USB/
└── update/
    ├── manifest.json
    └── app.tar.gz


Verify

ls -R /media/*/UPDATE_USB/update 2>/dev/null || true

C) “업데이트 직후 다시 롤백됨 (자동 롤백 오동작처럼 보임)”

Symptom

업데이트가 된 것 같지만 다음 실행/타이머 이후 이전 버전으로 되돌아감

history 로그에 rollback 기록

Cause (설계된 동작)

설치 직후 pending=true

헬스 승인(healthy mark) 전에 updater가 다시 실행되면 롤백 조건이 될 수 있음

Fix

sudo /usr/local/sbin/usb-update-mark-healthy
cat /var/lib/usb-updater/state.json


Verify

state.json에서 pending이 "false"인지 확인

이후 timer가 돌아도 rollback이 발생하지 않는지 확인

D) “업데이트가 계속 skip 처리됨”

Symptom

새 버전을 넣었다고 생각했는데 설치가 안 되고 skip 로그만 남음

Cause

manifest.json의 version이 state.json의 current_version과 동일함

Fix

manifest.json의 version을 실제 새 버전으로 올려서 배포

Verify

cat /var/lib/usb-updater/state.json
cat /media/*/UPDATE_USB/update/manifest.json 2>/dev/null || true
tail -n 30 /var/log/usb-updater-history.log

E) “수동 롤백이 안 됨 / 백업이 없다”

Symptom

usb-update-rollback last 실패

/opt/my-app-backups/가 비어있음

Cause

정상 업데이트가 한 번도 수행되지 않았거나

백업 생성 단계에서 권한/디스크 문제로 실패

Fix

ls /opt/my-app-backups
sudo usb-update-rollback last


Verify

롤백 후 앱 실행 및 state.json 갱신 확인

F) “sha256 필드가 있는데 무결성 검증이 약한 것 같음”

Symptom

sha256이 비어있어도 업데이트가 진행되는 것처럼 보임

Cause

현재 구조상 sha256는 확장 포인트(필수 강제는 TODO로 남겨둘 수 있음)

Fix (운영 팁)

sha256sum app.tar.gz
# manifest.json에 반영 (또는 스크립트에서 필수 검증으로 강화)

✅ Quick Check Commands
# 서비스 상태
systemctl status usb-updater.service
systemctl status usb-updater.timer
systemctl status my-app.service

# udev 트리거 확인
sudo journalctl -fu systemd-udevd -fu usb-updater-on-usb@sda1.service

# 업데이트 로그
tail -n 50 /var/log/usb-updater.log
tail -n 50 /var/log/usb-updater-history.log

# 상태 확인
cat /var/lib/usb-updater/state.json

📌 TODO / 확장 아이디어

SHA256 검증 “필수화”

여러 앱(target) 지원 (manifest 다중 엔트리)

A/B 루트 파티션(전체 OS 롤백) 구조로 확장

네트워크 업데이트 서버 연동(USB는 fallback 용)
