# Raspberry Pi 5 – USB Auto Update System  
(자동 롤백 + 헬스 체크 + udev 트리거)

라즈베리파이 5 + Raspberry Pi OS 환경에서,  
**USB(또는 외장 SSD)를 꽂기만 하면 애플리케이션이 자동으로 업데이트**되고,  
문제가 생기면 **자동으로 이전 버전으로 롤백**되는 미니 프로젝트입니다.

> 이 레포는 “실제 현업에서 써도 될 수준”의 업데이트 흐름을  
> 작은 예제(my-app)로 축소해서 보여주는 데 목적이 있습니다.

---

## 기능 요약 (Features)

- **USB 기반 앱 배포**
  - `UPDATE_USB` 라벨을 가진 스토리지에  
    `update/manifest.json` + `update/app.tar.gz` 만 넣으면 동작
  - 타겟 디렉터리(기본: `/opt/my-app`)로 자동 전개

- **버전 관리 & 중복 업데이트 방지**
  - `/var/lib/usb-updater/state.json` 에 `current_version` 저장
  - `manifest.json` 의 `version` 과 비교해서  
    **이미 같은 버전이면 스킵(skip)**

- **헬스 체크(Health Check) & pending 상태**
  - 새 버전 설치 → `pending=true` 로 표시
  - 실제 서비스가 잘 뜨면 `usb-update-mark-healthy` 호출 → `pending=false`, `last_status="ok"`

- **자동 롤백(Auto Rollback)**
  - `pending=true` 상태에서 usb-updater가 다시 실행되면  
    → 최신 백업 디렉터리에서 이전 버전으로 자동 복구
  - 롤백 사유는 `state.json` + `history.log` 에 기록

- **수동 롤백(Manual Rollback)**
  - `usb-update-rollback last` 로 가장 최근 백업으로 복원
  - 특정 백업 디렉터리 이름으로도 롤백 가능

- **이력 로그(History Log)**
  - `/var/log/usb-updater-history.log` 에
    - 타임스탬프
    - 상태(ok / rollback / skip / manual-rollback / ok-pending 등)
    - 이전 버전, 새 버전
    - 메시지
    를 한 줄씩 남김

- **udev + systemd 통합**
  - `UPDATE_USB` 라벨 스토리지를 꽂으면
    - udev → `usb-updater-on-usb@.service` → `usb-updater-udev-wrapper` → `usb-updater`
  - 별도 명령 없이 자동 업데이트 수행

---

## 테스트 환경 (Tested Environment)

- Raspberry Pi 5
- Raspberry Pi OS (Debian 기반, bookworm 계열)
- 루트 파일시스템에 `/opt/my-app` 디렉터리 사용
- USB 또는 외장 SSD, **파일시스템 라벨: `UPDATE_USB`**

---

## 디렉터리 구조

```text
Raspbery-pi5-USB-update/
├── README.md
├── scripts/
│   ├── usb-updater               # 메인 업데이트 스크립트
│   ├── usb-update-mark-healthy   # 헬스 체크 승인용
│   ├── usb-update-rollback       # 수동 롤백용
│   └── usb-updater-udev-wrapper  # udev에서 호출하는 래퍼
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
USB 업데이트 패키지 구조
라벨이 UPDATE_USB 인 스토리지 안에 다음과 같이 준비합니다.

text
코드 복사
UPDATE_USB/
└── update/
    ├── manifest.json
    └── app.tar.gz
manifest.json 예시
json
코드 복사
{
  "version": "1.0.0",
  "description": "example update package",
  "sha256": "",
  "target": "/opt/my-app",
  "service": "my-app.service"
}
version : 애플리케이션 버전 문자열 (버전 비교에 사용)

description : 업데이트 설명 (로그용)

sha256 : (옵션) 패키지 무결성 확인용 해시

target : 전개 대상 디렉터리 (예: /opt/my-app)

service : 업데이트 이후 재시작할 systemd 서비스 이름

app.tar.gz 예시 만들기
bash
코드 복사
mkdir -p /tmp/app_v1.0.0
cp examples/example-my-app.sh /tmp/app_v1.0.0/my-app.sh
chmod +x /tmp/app_v1.0.0/my-app.sh

tar -C /tmp/app_v1.0.0 -czf app.tar.gz .
# 만들어진 app.tar.gz 를 UPDATE_USB/update/ 아래에 복사
설치 (Installation)
Raspberry Pi OS 환경, /opt/my-app 경로, UPDATE_USB 라벨을 기준으로 설명합니다.

1) 저장소 클론
bash
코드 복사
cd ~
git clone https://github.com/nasi546/Raspbery-pi5-USB-update.git
cd Raspbery-pi5-USB-update
2) 스크립트 설치
bash
코드 복사
sudo cp scripts/usb-updater \
        scripts/usb-update-mark-healthy \
        scripts/usb-update-rollback \
        scripts/usb-updater-udev-wrapper \
        /usr/local/sbin/

sudo chmod +x /usr/local/sbin/usb-*
3) systemd 유닛 설치
bash
코드 복사
sudo cp systemd/*.service systemd/*.timer /etc/systemd/system/
sudo systemctl daemon-reload
샘플 유닛들은 다음을 가정합니다:

앱 서비스: my-app.service

메인 업데이트 서비스: usb-updater.service

주기 실행 타이머: usb-updater.timer

udev 트리거용 템플릿 서비스: usb-updater-on-usb@.service

4) udev 룰 설치
bash
코드 복사
sudo cp udev/99-usb-updater.rules /etc/udev/rules.d/
sudo udevadm control --reload
99-usb-updater.rules 내용:

udev
코드 복사
ACTION=="add", SUBSYSTEM=="block", ENV{ID_FS_LABEL}=="UPDATE_USB", ENV{SYSTEMD_WANTS}="usb-updater-on-usb@%k.service"
5) 예제 앱 설치(선택)
bash
코드 복사
sudo mkdir -p /opt/my-app
sudo cp examples/example-my-app.sh /opt/my-app/my-app.sh
sudo chmod +x /opt/my-app/my-app.sh
서비스 활성화 및 실행
1) 서비스/타이머 enable + start
bash
코드 복사
sudo systemctl enable my-app.service
sudo systemctl enable usb-updater.service
sudo systemctl enable usb-updater.timer

sudo systemctl start my-app.service
sudo systemctl start usb-updater.timer
2) udev 트리거 확인
USB/외장 SSD에 라벨을 설정합니다 (한 번만 하면 됨):

bash
코드 복사
# 예: /dev/sda1 이 UPDATE_USB 용 파티션이라면 (ext4 경우)
sudo e2label /dev/sda1 UPDATE_USB
그 후, 스토리지를 꽂으면:

udev → usb-updater-on-usb@sda1.service 실행

usb-updater-udev-wrapper 가 /media/<USER>/UPDATE_USB 마운트 확인

usb-updater 실행 → 업데이트 진행

사용 방법 (Usage)
1) 자동 업데이트 (추천)
USB/SSD 에 UPDATE_USB 라벨 설정

update/manifest.json + update/app.tar.gz 배치

라즈베리파이 5 에 꽂기

로그 확인:

bash
코드 복사
sudo journalctl -u usb-updater.service -u usb-updater-on-usb@sda1.service --no-pager
tail -n 20 /var/log/usb-updater.log
앱 확인:

bash
코드 복사
/opt/my-app/my-app.sh
cat /var/lib/usb-updater/state.json
2) 수동 실행
bash
코드 복사
sudo usb-updater
# 또는 특정 마운트 경로를 강제로 지정
sudo usb-updater --mount-path /media/pi07/UPDATE_USB
3) 헬스 체크 승인
앱이 정상 동작하는 것이 확인되면:

my-app.service 가 재시작될 때
ExecStartPost=/usr/local/sbin/usb-update-mark-healthy 가 호출됩니다.

직접 호출해도 됩니다:

bash
코드 복사
sudo /usr/local/sbin/usb-update-mark-healthy
cat /var/lib/usb-updater/state.json
4) 수동 롤백
bash
코드 복사
# 백업 목록 보기
ls /opt/my-app-backups

# 가장 최근 버전으로 롤백
sudo usb-update-rollback last

# 특정 백업 디렉터리 이름으로 롤백
sudo usb-update-rollback my-app-20251218-105544
내부 동작 (State & Rollback Logic)
state.json 구조
/var/lib/usb-updater/state.json 예시:

json
코드 복사
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

previous_version: 직전 버전

backup_dir : 롤백용 백업 경로(대부분 최신 업데이트 이전 버전)

pending : true면 아직 헬스 체크 완료 전

last_status : "ok", "pending", "rollback", "manual-rollback", "skip" 등

last_error : 오류/롤백 사유 문자열

롤백 조건
pending == "true" 이고

backup_dir 가 비어있지 않으며

usb-updater 가 다시 실행되면:

→ 자동으로 backup_dir 내용으로 target_dir 복구
→ current_version 를 previous_version 또는 백업 기준 버전으로 되돌림
→ pending="false", last_status="rollback"

로그 위치 & 트러블슈팅
메인 로그: /var/log/usb-updater.log

히스토리 로그: /var/log/usb-updater-history.log

상태 파일: /var/lib/usb-updater/state.json

백업 디렉터리: /opt/my-app-backups/

작업 디렉터리: /opt/my-app-updates/work/

자주 보는 명령어
bash
코드 복사
# 서비스 상태
systemctl status usb-updater.service
systemctl status usb-updater.timer
systemctl status my-app.service

# udev 트리거 확인
sudo journalctl -fu systemd-udevd -fu usb-updater-on-usb@sda1.service

# 업데이트 로그
tail -n 20 /var/log/usb-updater.log
tail -n 20 /var/log/usb-updater-history.log
TODO / 확장 아이디어
SHA256 검증 필수화 (현재는 필드만 존재)

여러 앱(target) 지원 (manifest에 여러 엔트리)

A/B 루트 파티션(전체 OS 롤백) 구조로 확장

네트워크 기반 업데이트 서버와 연동 (USB는 fallback 용)
🧯 Troubleshooting (Issues & Fixes)

이 프로젝트는 USB(또는 외장 SSD) 삽입만으로 업데이트를 자동 수행하고, 업데이트 후 앱이 정상 기동하지 않으면 pending 기반 자동 롤백이 동작하도록 설계되어 있습니다. 

아래는 실제 운영/테스트에서 자주 만나는 문제를 Symptom → Cause → Fix → Verify로 정리했습니다.

0) 먼저 보는 로그/상태 파일 (원인 추적 1순위)

메인 로그: /var/log/usb-updater.log 
GitHub

히스토리 로그: /var/log/usb-updater-history.log 
GitHub

상태 파일: /var/lib/usb-updater/state.json 

백업 디렉터리: /opt/my-app-backups/ 

작업 디렉터리: /opt/my-app-updates/work/ 

자주 쓰는 확인 명령: 
# 서비스 상태
systemctl status usb-updater.service
systemctl status usb-updater.timer
systemctl status my-app.service

# udev 트리거 확인
sudo journalctl -fu systemd-udevd -fu usb-updater-on-usb@sda1.service

# 업데이트 로그
tail -n 50 /var/log/usb-updater.log
tail -n 50 /var/log/usb-updater-history.log

# 상태
cat /var/lib/usb-updater/state.json

1) “USB 꽂았는데 업데이트가 시작 자체를 안 함”

Symptom

USB를 꽂아도 usb-updater-on-usb@...service가 뜨지 않음

로그에 아무것도 안 남음

Cause

udev 트리거 조건은 파일시스템 라벨이 UPDATE_USB 일 때만 동작함 

udev 룰이 /etc/udev/rules.d/에 설치/리로드가 안 됐거나, 라벨이 다른 파티션에 설정됨 
GitHub

Fix

라벨 확인 및 설정 

lsblk -f
# 예: /dev/sda1 이 UPDATE_USB 용이면
sudo e2label /dev/sda1 UPDATE_USB


udev 룰 설치/리로드 

sudo cp udev/99-usb-updater.rules /etc/udev/rules.d/
sudo udevadm control --reload


Verify

USB 재삽입 후 아래 로그에 usb-updater-on-usb@...가 뜨는지 확인 

sudo journalctl -u usb-updater-on-usb@sda1.service --no-pager

2) “서비스는 떴는데 manifest/app.tar.gz를 못 찾는 것 같음”

Symptom

usb-updater가 실행은 되는데 업데이트가 진행되지 않음

로그에 manifest/app 경로 관련 실패가 보임

Cause

USB 내부 구조가 아래처럼 정확히 되어 있어야 함
UPDATE_USB/update/manifest.json + UPDATE_USB/update/app.tar.gz 

udev 흐름에서는 wrapper가 /media/<USER>/UPDATE_USB 마운트 경로를 기준으로 확인함(README 흐름 설명) 

Fix

USB에 아래 구조로 배치 

UPDATE_USB/
└── update/
    ├── manifest.json
    └── app.tar.gz


마운트 경로가 환경에 따라 다르면 수동 실행로 경로 고정 가능 

sudo usb-updater --mount-path /media/pi07/UPDATE_USB


Verify

ls -R /media/*/UPDATE_USB/update 2>/dev/null || true
cat /media/*/UPDATE_USB/update/manifest.json 2>/dev/null || true

3) “업데이트가 됐는데 바로 다시 롤백돼 버림 (자동 롤백 오동작처럼 보임)”

Symptom

업데이트 직후엔 새 버전이 설치된 것 같은데, 다음 실행/재부팅/타이머 동작 후 이전 버전으로 되돌아감

history.log에 rollback이 찍힘

Cause (설계된 동작)

이 프로젝트는 설치 직후 pending=true로 표시하고,
앱이 “정상 기동 확인”되면 usb-update-mark-healthy가 호출되어 pending=false가 되도록 설계됨 

pending=true 상태에서 usb-updater가 다시 실행되면(예: usb-updater.timer) 자동 롤백 조건을 만족할 수 있음 

Fix

앱이 정상 동작하는 걸 확인한 뒤, 헬스 승인 처리 
GitHub

sudo /usr/local/sbin/usb-update-mark-healthy
cat /var/lib/usb-updater/state.json


또는 my-app.service가 재시작될 때 ExecStartPost로 자동 호출되도록 구성되어 있으니, 서비스 유닛/동작을 함께 점검

Verify

state.json의 pending이 "false"인지 확인 
GitHub
+1

이후 타이머가 돌아도 rollback이 발생하지 않는지 확인

4) “업데이트가 계속 ‘skip’ 처리됨 (새 버전 넣었는데도…)”

Symptom

USB에 업데이트를 넣어도 로그에 skip이 찍히고 설치가 안 됨

Cause

state.json의 current_version과 manifest.json의 version이 같으면 중복 업데이트 방지로 스킵함 

Fix

manifest.json의 version을 실제 새 버전으로 올려서 배포 


(테스트 목적) state.json을 직접 수정하는 건 추천하지 않음 — 이력/롤백 판단이 꼬일 수 있음 


Verify

cat /var/lib/usb-updater/state.json
cat /media/*/UPDATE_USB/update/manifest.json 2>/dev/null || true
tail -n 30 /var/log/usb-updater-history.log

5) “수동 롤백이 안 됨 / 백업이 없다”

Symptom

usb-update-rollback last를 했는데 실패

/opt/my-app-backups/에 백업 디렉터리가 없음

Cause

백업은 업데이트 과정에서 생성/갱신되며, 상태 파일의 backup_dir도 함께 관리됨 


아직 “정상 업데이트가 한 번도 성공적으로 수행되지 않았거나”, 백업 경로 권한/디렉터리 생성이 실패했을 수 있음

Fix

백업 목록 확인 후 롤백 

ls /opt/my-app-backups
sudo usb-update-rollback last
# 또는 특정 백업 디렉터리로 지정
sudo usb-update-rollback my-app-20251218-105544


Verify

롤백 후 my-app.sh 실행 및 state.json 갱신 여부 확인 

6) “무결성(SHA256) 검증이 기대처럼 동작하지 않음”

Symptom

manifest.json에 sha256 필드가 있는데, 빈 값이어도 업데이트가 진행되는 것처럼 보임

Cause

현재 README에서도 확장 아이디어로 “SHA256 검증 필수화”가 TODO로 남아있는 상태 

(즉, 현 버전은 sha256이 ‘필수 강제’가 아닐 수 있음)

Fix (운영 팁)

배포 전에 수동 검증 습관화

sha256sum app.tar.gz
# manifest.json에 값 반영(또는 검증 로직을 필수화하도록 개선)


Verify

파일 손상/오배포를 일부러 만들어 검증 실패가 잡히는지(또는 개선 후) 테스트

7) “어디서부터 문제인지 모르겠을 때” (진단 순서)

라벨 확인: lsblk -f 에서 대상 파티션 라벨이 UPDATE_USB인지 

udev 트리거 확인: usb-updater-on-usb@...service가 실제로 떴는지 

마운트/파일 확인: /media/<USER>/UPDATE_USB/update/{manifest.json,app.tar.gz} 존재 여부 

상태 확인: state.json에서 current_version/pending/last_error 확인 

롤백 여부 확인: history.log에 rollback/skip 사유가 남는지 확인 
