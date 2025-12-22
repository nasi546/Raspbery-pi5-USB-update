
# Raspberry Pi USB Auto Updater (with rollback & health check)

라즈베리파이 5 + Raspberry Pi OS 환경에서,
`UPDATE_USB` 라벨이 붙은 USB/외장 SSD를 꽂으면

- 앱 디렉터리(`/opt/my-app`)를 USB에 있는 패키지로 자동 업데이트하고
- 업데이트 후 health check 실패 시 이전 버전으로 자동 롤백하고
- 수동 롤백, 업데이트 히스토리 로그까지 제공하는
미니 프로젝트용 USB 자동 업데이트 서브시스템입니다.

## 기능 요약

- **USB 기반 앱 업데이트**
  - `/media/<USER>/UPDATE_USB/update/app.tar.gz` 를 `/opt/my-app`으로 배포
  - `manifest.json`의 `version`/`target`/`service` 필드를 기반으로 동작

- **버전 비교 & 중복 업데이트 방지**
  - state.json에 저장된 `current_version`과 manifest의 `version`을 비교
  - 이미 같은 버전이면 실제 업데이트 작업은 스킵

- **헬스 체크(Health check)와 pending 상태**
  - 새 버전을 설치하면 `pending=true`로 표시
  - 실제 서비스(`my-app.service`)가 정상 기동되면
    `usb-update-mark-healthy`가 호출되어 `pending=false`, `last_status="ok"`로 전환

- **자동 롤백(Auto rollback)**
  - `pending=true` 상태에서 usb-updater가 다시 실행되면
  - 백업 디렉터리(`/opt/my-app-backups/...`)에서 이전 버전으로 자동 복구
  - state.json에 `last_status="rollback"` + `last_error`에 이유 기록

- **수동 롤백(Manual rollback)**
  - `usb-update-rollback` 스크립트로 특정 백업 디렉터리 혹은 “최근 버전”으로 복구 가능

- **이력 로그(History log)**
  - `/var/log/usb-updater-history.log` 에
    - 타임스탬프, 상태(ok/rollback/skip/manual-rollback 등), 이전/현재 버전, 메시지 기록

- **udev + systemd 연동**
  - `UPDATE_USB` 라벨이 붙은 스토리지를 꽂으면
  - udev 룰 → `usb-updater-on-usb@.service` → `usb-updater-udev-wrapper` → `usb-updater`
  - 완전히 자동으로 업데이트 수행

## 디렉터리 구조

```text
usb-auto-updater/
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
