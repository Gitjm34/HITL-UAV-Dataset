# UAV MAVLink 다채널 데이터셋 (HITL + 실기체, 2026)

PX4/MAVLink 기반 드론의 사이버-물리 보안 연구를 위한 **다채널 공개 데이터셋**이다. Pixhawk 6C Mini 비행 제어기, Raspberry Pi 4 온보드 컴퓨터, Gazebo 시뮬레이터를 활용한 HITL 환경에서 수집되었으며, 도메인 간 비교를 위해 검증된 실기체 정상 비행 자료도 함께 제공한다.

## 핵심 Contribution

본 데이터셋은 저자가 조사한 범위 내에서 UAV MAVLink 통신을 **4가지 서로 다른 관측 채널에서 동시에 캡쳐**한 첫 공개 데이터셋이다.

1. **`drone_to_gcs/`** (드론 → GCS, .tlog 다운링크) — 드론이 QGC로 출력하는 풀 텔레메트리
2. **`gcs_to_drone/`** (GCS → 드론, .tlog 업링크) — QGC가 드론으로 보내는 명령
3. **`drone_to_rpi/`** (드론 → RPi, TELEM1 UART) — 온보드 IDS 입력 채널
4. **`attacker_to_drone/`** (공격자 → 드론, UDP 14550) — 공격 패킷 (공격 비행에만)

각 비행에서 가능한 모든 채널을 동시 캡쳐했으며, NTP 동기화된 시계를 바탕으로 시간 정렬된 통합 파일(`merged_all.jsonl`)을 함께 제공한다. 이를 통해 다른 단일 채널 데이터셋이 불가능했던 **채널별 공격 발현 차이 정량 분석**이 가능하다.

## 디렉토리 구조

```
HITL-UAV-Dataset/
├── hitl/                              # HITL 데이터 (Pixhawk 6C Mini + Gazebo)
│   ├── hover/                         # 정지 비행
│   │   ├── normal/                    # 17 runs (4/20: 10 + 4/24: 7)
│   │   └── attack/
│   │       ├── disarm/                # 3 runs
│   │       ├── flooding/              # 3 runs
│   │       ├── pid_tamper/            # 3 runs
│   │       ├── tampering/             # 3 runs
│   │       └── malformed/             # 3 runs
│   │
│   └── mission/                       # 사각형 경로 미션 비행
│       ├── normal/                    # 9 runs
│       └── attack/
│           ├── flooding/              # 3 runs (★ 4채널 모두)
│           ├── pid_tamper/            # 3 runs (★)
│           ├── tampering/             # 5 runs (★)
│           └── malformed/             # 3 runs (★)
│
├── real_flight/                       # 실기체 검증된 정상 비행
│   ├── 2025-03-06_164436/
│   └── 2025-03-12_163440/
│
├── README.md
└── LICENSE
```

총 **HITL 57 runs + 실기체 2 세션**

## 각 Run 디렉토리 내부 구조

```
<run_id>/
├── drone_to_gcs/downlink.jsonl         # 드론→GCS 다운링크 (.tlog 추출)
├── gcs_to_drone/uplink.jsonl           # GCS→드론 업링크 (.tlog 추출)
├── drone_to_rpi/telem1.jsonl           # 드론→RPi TELEM1 (★ IDS 입력)
├── attacker_to_drone/attacker_udp.jsonl # 공격자→드론 UDP (공격에만)
├── tlog_original/<datetime>.tlog       # 원본 .tlog 파일
├── merged.jsonl                         # 양방향 시간 정렬
├── merged_all.jsonl                     # 모든 채널 시간 정렬 (★ 다채널 분석)
├── features.csv                         # 18개 엔지니어링 피처 (4/20 hover만)
└── meta.json                            # 메타데이터
```

각 run에 어떤 채널이 있는지는 `meta.json`의 `channels_available` 필드에 명시된다.

## 데이터 채널 상세

### 1. `drone_to_gcs/downlink.jsonl`
- **물리 경로**: 픽스호크 USB-C → Ubuntu VM (QGC)
- **추출**: `.tlog`에서 `srcSystem=1` 메시지만 추출
- **메시지 비율**: 약 566 msg/s
- **메시지 타입**: 풀 PX4 텔레메트리 (HIL_ACTUATOR_CONTROLS, ATTITUDE_QUATERNION, HIGHRES_IMU, ODOMETRY 등 모든 메시지)

### 2. `gcs_to_drone/uplink.jsonl`
- **물리 경로**: Ubuntu VM (QGC) → 픽스호크 USB-C
- **추출**: `.tlog`에서 `srcSystem=255` 메시지만 추출
- **내용**: ARM, TAKEOFF, COMMAND_LONG, MISSION_UPLOAD 등 QGC 명령

### 3. `drone_to_rpi/telem1.jsonl` (★ 본 IDS 입력)
- **물리 경로**: 픽스호크 TELEM1 UART (4핀) → Raspberry Pi 4 GPIO UART
- **수집 도구**: `collector_v2.py` (Python, pymavlink)
- **메시지 비율**: 약 12 msg/s
- **메시지 타입**: 14가지 (HIL_ACTUATOR_CONTROLS, ATTITUDE, GLOBAL_POSITION_INT, VFR_HUD, BATTERY_STATUS, GPS_RAW_INT 등)
- **방향**: 단방향 (드론 → RPi)
- **특징**: 본 연구의 온보드 IDS 모델이 입력으로 사용하는 제한된 정보 채널

### 4. `attacker_to_drone/attacker_udp.jsonl`
- **물리 경로**: Ubuntu VM 공격자 스크립트 → 네트워크 → 픽스호크 (UDP 14550)
- **수집 도구**: `collect_attack_uplink.py` (Python, scapy/pymavlink)
- **존재 조건**: 공격 비행에만

### 5. `tlog_original/<datetime>.tlog`
- 픽스호크 USB-C 채널의 원본 `.tlog` 파일 보존
- pymavlink (`mavutil.mavlink_connection`)로 직접 파싱 가능

## .tlog 커버리지 유형

각 run의 `meta.json`에 `tlog_coverage` 필드로 명시된다.

| Coverage | 의미 | 해당 runs |
|---|---|---|
| `full` | 비행 전체 캡쳐 | mission_normal 8 + hover_normal 1 + real_flight 2 = **11** |
| `pre_flight` | 비행 준비(ARM, TAKEOFF) 단계만 | hover_normal 4/24의 4 runs = **4** |
| `pre_attack` | 비행 시작 ~ 공격 직전 | mission_attack 14 runs = **14** |
| `none` | `.tlog` 없음 | hover_normal 4/20 (10) + hover_normal 4/24 미커버(2) + hover_attack 15 + mission_normal 1 = **28** |

`pre_flight`는 본인의 hover 정상 비행 절차에서 QGC로 ARM, TAKEOFF 명령을 보낸 후 RPi 시작 시점에 QGC를 종료한 결과이다. 짧은(50–95s) `.tlog` 안에는 양방향 명령(ARM 2회, TAKEOFF 2회 등)과 텔레메트리가 모두 보존되어 있어 비행 준비 단계의 양방향 통신 분석이 가능하다.

`pre_attack`는 공격 비행에서 UDP 14550 포트 충돌을 방지하기 위해 공격 시작 직전 QGC를 종료한 결과이다. 비행 시작부터 약 5–25초까지 캡쳐되어 있다.

## 시간 정렬 정직 명시

세 가지 시계가 사용되었다:
- RPi 시계 (drone_to_rpi 캡쳐)
- Ubuntu VM 시계 (.tlog 캡쳐 + UDP 캡쳐)

`mission/attack/flooding/20260428_193909_8fe27a` run에서 검증한 결과, NTP 동기화로 시계 차이는 1초 미만이다 (`drone_to_rpi`의 첫 메시지가 `.tlog` 시작 후 64.5초인 것은 본인 절차상 RPi가 안정 비행 후 시작되기 때문이다).

따라서 `merged.jsonl`과 `merged_all.jsonl`은 **각 채널의 원본 timestamp를 보정 없이 단순 정렬**한다. 사용자는 절대 시각 차이가 1초 미만임을 가정할 수 있다.

## 하드웨어/소프트웨어 환경

| 구성 요소 | 사양 |
|---|---|
| 비행 제어기 | Pixhawk 6C Mini (PX4 SITL, Gazebo HITL) |
| 온보드 컴퓨터 | Raspberry Pi 4 (TELEM1 UART, baud 57600) |
| GCS | QGroundControl (Ubuntu VM, USB-C 연결) |
| 공격자 | Ubuntu VM, pymavlink 기반 공격 스크립트 |

## 공격 유형

| 공격 유형 | 설명 | 일반 업링크 메시지 수 |
|---|---|---|
| `flooding` | 고속 명령 플러딩 | 약 11,000 msgs |
| `pid_tamper` | PID 파라미터를 PARAM_SET으로 변조 | 12 msgs |
| `tampering` | 다양한 파라미터를 PARAM_SET으로 변조 | 12 msgs |
| `malformed` | 잘못된 형식의 MAVLink 패킷 | 약 2,370 msgs |
| `disarm` | DISARM 명령 공격 (HITL hover에만) | 다양함 |

mission 시나리오에는 `disarm`이 포함되지 않는다.

## 데이터 수집 절차

1. 드론 부팅 (Gazebo HITL + Pixhawk SITL)
2. QGC 시작 및 드론 연결 (`.tlog` 기록 시작)
3. QGC를 통해 ARM, TAKEOFF 명령 전송
4. 안정적인 비행 상태(hover 정지 또는 mission 진입)까지 대기
5. 안정화 후 약 15초 뒤 RPi에서 `collector_v2.py` 실행 (`telem1.jsonl` 기록 시작)
6. **정상 mission 비행**: QGC를 비행 종료까지 계속 켜둠
7. **공격 비행**: QGC를 종료(`.tlog` 종료) 후 Ubuntu VM에서 `collect_attack_uplink.py` 실행
8. 비행 진행, RPi가 비행 전체를 기록
9. 착륙 후 RPi 종료, 공격 스크립트 실행 중이면 종료

## meta.json 필드

```json
{
  "run_id": "20260428_193909_8fe27a",
  "domain": "hitl",
  "scenario": "mission_rect",
  "attack_type": "flooding",
  "label": "attack",
  "duration_sec": 220.9,
  "channels_available": ["drone_to_gcs", "gcs_to_drone", "drone_to_rpi", "attacker_to_drone"],
  "tlog_coverage": "pre_attack",
  "has_merged": true,
  "has_merged_all": true,
  "has_features": false
}
```

## 사용 사례

### 1. 단일 채널 IDS 학습/평가 (본 연구의 사용법)
```python
# drone_to_rpi/telem1.jsonl만 사용 (12 msg/s, 14 types)
import json
msgs = [json.loads(line) for line in open('drone_to_rpi/telem1.jsonl')]
```

### 2. 다채널 공격 발현 비교 분석 (★ 본 데이터셋의 핵심 활용)
```python
# merged_all.jsonl로 4채널 시간 정렬된 모든 메시지 로드
msgs = [json.loads(line) for line in open('merged_all.jsonl')]

# 채널별 분포
from collections import Counter
print(Counter(m['channel'] for m in msgs))
# Counter({'drone_to_gcs': 39929, 'attacker_to_drone': 11420, 
#          'drone_to_rpi': 8340, 'gcs_to_drone': 233})
```

### 3. 양방향 통신 분석
```python
# merged.jsonl로 양방향 정렬 메시지 로드
msgs = [json.loads(line) for line in open('merged.jsonl')]
```

### 4. 원본 .tlog 직접 파싱
```python
from pymavlink import mavutil
m = mavutil.mavlink_connection('tlog_original/<datetime>.tlog')
```

## 데이터 품질 검증 과정에서 제외된 Run

| Run ID | 제외 사유 |
|---|---|
| `20260424_173322_88a384` | `raw.jsonl` JSON 파싱 에러 |
| `20260424_172105_116f93` | 비행 시간 28.6초 (mission 미완수) |
| `20260424_183921_4cec90` | 비행 시간 73초 (mission 미완수) |
| `20260428_192557_ecea9a` | pid_tamper로 라벨되었으나 COMMAND_ACK 2,369개로 비정상 트래픽, 라벨 오류 가능성 |

## 인용

```
(인용 형식 추후 추가)
```

## 라이선스

MIT License (LICENSE 파일 참조)

## 변경 이력

- **v3 (현재)**: 4채널 완전 분리 (drone_to_gcs/, gcs_to_drone/, drone_to_rpi/, attacker_to_drone/) + tlog_original/ 직관적 명명 + merged_all.jsonl 다채널 시간 정렬 추가
- **v2**: hitl/{hover,mission}/{normal,attack} 계층 구조 + qgc_usb*/ 분류
- **v1**: 4-perspective 구조 (gcs_to_drone_uplink, drone_to_gcs_downlink, bidirectional_merged, features) + hitl_extended 분리