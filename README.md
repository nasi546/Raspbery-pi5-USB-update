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
## 전체 동작 플로우 (Flowchart)
