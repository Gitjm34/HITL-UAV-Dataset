# HITL UAV 데이터셋

PX4/MAVLink 기반 드론의 사이버-물리 보안 연구를 위한 다채널 데이터셋이다. Pixhawk 6C Mini 비행 제어기, Raspberry Pi 4 온보드 컴퓨터, Gazebo 시뮬레이터를 활용한 HITL(Hardware-in-the-Loop) 환경에서 수집되었으며, 도메인 간 비교를 위해 소량의 실기체 정상 비행 기준선 데이터도 함께 제공한다.

## 구성 개요

HITL-UAV-Dataset/
├── hitl/                              # HITL 데이터 (Pixhawk 6C Mini + Gazebo)
│   ├── hover/                         # 정지 비행(hover) 시나리오
│   │   ├── normal/                    # 정상 17 runs (4/20 10개 + 4/24 7개)
│   │   └── attack/
│   │       ├── disarm/                # 디사밍 공격 3 runs
│   │       ├── flooding/              # 명령 플러딩 공격 3 runs
│   │       ├── pid_tamper/            # PID 파라미터 변조 공격 3 runs
│   │       ├── tampering/             # 일반 파라미터 변조 공격 3 runs
│   │       └── malformed/             # 잘못된 형식의 MAVLink 패킷 공격 3 runs
│   │
│   └── mission/                       # 사각형 경로 미션(mission_rect) 시나리오
│       ├── normal/                    # 정상 9 runs
│       └── attack/
│           ├── flooding/              # 3 runs
│           ├── pid_tamper/            # 3 runs
│           ├── tampering/             # 5 runs
│           └── malformed/             # 3 runs
│
├── real_flight/                       # 실기체 비행 데이터 (공격 없음)
│   ├── 2025-03-06_164436/             # 검증된 정상 비행 세션
│   └── 2025-03-12_163440/             # 검증된 정상 비행 세션
│
├── README.md
└── LICENSE

총 **HITL 57 runs + 실기체 2 세션**으로 구성되어 있다.

---

## 각 Run 디렉토리 구조

각 run 디렉토리는 최대 세 가지 관측 채널의 데이터를 포함한다.

### `drone_to_gcs/` — 드론에서 GCS 방향(downlink)

- **HITL 데이터**: `rpi_telem1.jsonl`
  - Pixhawk의 TELEM1 UART 포트에서 Raspberry Pi 4가 수집한 텔레메트리
  - 약 12 msg/s, 14가지 메시지 타입
  - 본 연구의 IDS 모델이 입력으로 사용하는 채널과 동일

- **실기체 데이터**: `downlink.jsonl`
  - QGC `.tlog` 파일에서 srcSystem=1(드론) 메시지만 추출한 다운링크
  - HITL의 RPi 캡쳐와는 다른 채널(USB-C 풀 텔레메트리)이므로 이름을 구분함

### `gcs_to_drone/` — GCS에서 드론 방향(uplink)

업링크가 캡쳐된 경우에만 존재한다.

- **HITL 공격 비행**: `ubuntu_attacker.jsonl`
  - Ubuntu VM의 공격자 스크립트가 UDP 14550 포트로 전송한 공격 패킷
  - 공격 phase(약 60초)만 캡쳐됨

- **실기체**: `uplink.jsonl`
  - `.tlog`에서 srcSystem=255(QGC) 메시지만 추출한 업링크

### `qgc_usb/`, `qgc_usb_pre_flight/`, `qgc_usb_pre_attack/` — QGC USB-C 텔레메트리

QGroundControl이 USB-C로 연결되어 기록한 `.tlog` 파일이다. 약 566 msg/s의 PX4 풀 텔레메트리(HIL_ACTUATOR_CONTROLS, ATTITUDE_QUATERNION, HIGHRES_IMU, ODOMETRY, LOCAL_POSITION_NED 등)를 양방향으로 포함한다. QGC의 작동 방식에 따라 세 가지 커버리지 유형이 있다.

- **`qgc_usb/`** — 비행 전체 커버리지
  - 비행 시작 전부터 종료 직후까지 QGC가 켜져 있던 경우
  - 정상 mission 비행 대부분과 hover normal 1 run에 해당

- **`qgc_usb_pre_flight/`** — 비행 준비 단계만 커버리지
  - QGC로 ARM, TAKEOFF 명령을 보낸 후 RPi 시작 시점에 QGC를 끈 경우
  - hover normal 4 runs에 해당
  - 비행 자체는 캡쳐되지 않았으나 비행 직전 양방향 명령 흐름이 보존됨

- **`qgc_usb_pre_attack/`** — 공격 시작 직전까지의 커버리지
  - 공격 비행에서 QGC를 켜놓다가 공격 시작 직전에 종료한 경우 (UDP 14550 충돌 방지)
  - 모든 mission attack 14 runs에 해당
  - 비행 시작부터 약 5~25초 후까지 캡쳐됨

### `merged.jsonl` — 양방향 시간 정렬 통합

다운링크와 업링크가 모두 있는 run에서 두 채널을 timestamp 기준으로 시간 정렬해 합친 파일이다.

### `features.csv` — 엔지니어링 피처 윈도우

본 연구에서 사용한 18개 물리/통신 피처를 윈도우 단위로 추출한 CSV 파일이다. 4/20 hover 데이터(원본 25 runs)에 대해서만 제공된다.

### `meta.json` — Run 메타데이터

run의 시작 시각, 종료 시각, 시나리오, 공격 유형, 비행 시간, 메시지 수 등의 메타 정보를 담고 있다.

---

## 하드웨어/소프트웨어 환경

| 구성 요소 | 사양 |
|---|---|
| 비행 제어기 | Pixhawk 6C Mini (PX4 SITL, Gazebo HITL) |
| 온보드 컴퓨터 | Raspberry Pi 4 (TELEM1 UART, baud 57600) |
| GCS | QGroundControl (Ubuntu VM, USB-C 연결) |
| 공격자 | Ubuntu VM, pymavlink 기반 공격 스크립트 |

---

## 공격 유형

| 공격 유형 | 설명 | 일반 업링크 메시지 수 |
|---|---|---|
| `flooding` | 고속 명령 플러딩 공격 | 약 11,000 msgs |
| `pid_tamper` | PID 파라미터를 PARAM_SET으로 변조 | 12 msgs |
| `tampering` | 다양한 파라미터를 PARAM_SET으로 변조 | 12 msgs |
| `malformed` | 잘못된 형식의 MAVLink 패킷 전송 | 약 2,370 msgs |
| `disarm` | DISARM 명령 공격 (HITL hover에만 포함) | 다양함 |

mission 시나리오에는 `disarm` 공격이 포함되지 않는다. mission 비행에서는 4가지 공격 유형(flooding, pid_tamper, tampering, malformed)에 집중하여 실험을 수행했다.

---

## 데이터 수집 절차

표준 수집 절차는 다음과 같다.

1. 드론 부팅 (Gazebo HITL + Pixhawk SITL)
2. QGC 시작 및 드론 연결 (`.tlog` 기록 시작)
3. QGC를 통해 ARM, TAKEOFF 명령 전송
4. 안정적인 비행 상태(hover 정지 또는 mission 진입)까지 대기
5. 안정화 후 약 15초 뒤 RPi에서 `collector_v2.py` 실행 (`raw.jsonl` 기록 시작)
6. **정상 mission 비행의 경우**: QGC를 비행 종료까지 계속 켜둠
7. **공격 비행의 경우**: QGC를 종료(`.tlog` 종료) 후 Ubuntu VM에서 `collect_attack_uplink.py` 실행
8. 비행 진행, RPi가 비행 전체를 기록
9. 착륙 후 RPi 종료, 공격 스크립트 실행 중이면 종료

이 절차의 차이가 위에서 설명한 `.tlog` 커버리지의 세 가지 유형(`qgc_usb`, `qgc_usb_pre_flight`, `qgc_usb_pre_attack`)을 만들어낸다.

---

## 데이터 채널 가용성

모든 run이 세 가지 채널을 모두 보유하지는 않는다. 정직한 가용성은 다음과 같다.

### HITL hover normal (17 runs)
- **4/20 hover normal (10 runs)**: RPi TELEM1만 보유 (당시 QGC 미연결, Ubuntu attacker 미실행)
- **4/24 hover normal (7 runs)**:
  - 1 run에 `qgc_usb/` (전체 커버리지)
  - 4 runs에 `qgc_usb_pre_flight/` (준비 단계만)
  - 2 runs는 `.tlog` 없음 (QGC 비활성)

### HITL hover attack (15 runs)
- 모든 run에 RPi TELEM1, Ubuntu attacker uplink, merged.jsonl, features.csv 보유
- `.tlog`는 없음 (당시 QGC 사용 정책 미정착)

### HITL mission normal (9 runs)
- 모든 run에 RPi TELEM1 보유
- 8 runs에 `qgc_usb/` (전체 커버리지)
- 1 run(`20260424_165837_d23baf`)은 `.tlog` 매칭 실패

### HITL mission attack (14 runs)
- 모든 run에 RPi TELEM1, Ubuntu attacker uplink, merged.jsonl 보유
- 모든 run에 `qgc_usb_pre_attack/` 보유

### 실기체 (2 세션)
- 두 세션 모두 `.tlog` 원본 + 추출된 downlink/uplink + merged.jsonl 보유

---

## 데이터셋에서 제외된 Run

다음 run들은 데이터 품질 검증 과정에서 제외되었다. 정직성을 위해 명시한다.

| Run ID | 제외 사유 |
|---|---|
| `20260424_173322_88a384` | `raw.jsonl` JSON 파싱 에러 1줄 발견 |
| `20260424_172105_116f93` | 비행 시간 28.6초로 매우 짧음 (mission 미완수, 실험 실패 추정) |
| `20260424_183921_4cec90` | 비행 시간 73초로 짧음 (mission 미완수 추정) |
| `20260428_192557_ecea9a` | pid_tamper로 라벨되었으나 COMMAND_ACK 2,369개로 비정상 트래픽 패턴, 다른 공격이 혼합되었거나 라벨 오류 가능성 |

---

## 사용 시 주의 사항

1. **본 연구의 IDS 모델 학습/평가용 데이터**: `drone_to_gcs/rpi_telem1.jsonl`(HITL) 또는 `drone_to_gcs/downlink.jsonl`(실기체)을 사용한다. 이 채널은 RPi의 TELEM1 stream으로, 본 연구의 온보드 IDS 입력과 동일하다.

2. **다채널 연구용 데이터**: `qgc_usb*/`의 `.tlog` 파일은 QGC USB-C 채널의 풀 텔레메트리(566 msg/s)를 제공한다. RPi TELEM1 채널(12 msg/s)과 함께 분석하면 채널 간 공격 발현 양상의 차이를 연구할 수 있다.

3. **시간 정렬**: 모든 timestamp는 Unix epoch 초(UTC) 단위이다. Run 단위 시간 정보는 각 `meta.json`에 포함되어 있다.

4. **`.tlog` 파일 처리**: pymavlink 라이브러리(`pymavlink.mavutil.mavlink_connection`)로 읽을 수 있다. 파일명의 시각은 QGC가 파일을 닫은 시각(즉 종료 시각)이며, 실제 데이터 시작 시각은 파일 내부의 첫 timestamp를 확인해야 한다.

---

## 인용

본 데이터셋을 연구에 활용한 경우 아래와 같이 인용한다.


## 라이선스

MIT License (LICENSE 파일 참조)

---

## 변경 이력

- **v2 (현재)**: 디렉토리 구조 재정비. `hitl/{hover,mission}/{normal,attack}` 계층 + 방향 기반 서브디렉토리(`drone_to_gcs/`, `gcs_to_drone/`) + `.tlog` 커버리지별 분류
- **v1**: 4-perspective 구조(gcs_to_drone_uplink, drone_to_gcs_downlink, bidirectional_merged, features) + `hitl_extended/`, `real_flight_baseline/` 분리