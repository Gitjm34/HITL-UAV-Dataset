# UAV MAVLink 다채널 데이터셋 (HITL + Real Flight, 2026)

PX4/MAVLink 기반 드론의 사이버-물리 보안 연구를 위한 **다채널 시간 정렬 공개 데이터셋**이다. Pixhawk 6C Mini 비행 제어기와 Raspberry Pi 4 온보드 컴퓨터, Gazebo HITL 시뮬레이터를 사용하여 동일 비행을 4가지 서로 다른 관측 채널에서 동시 캡쳐하였다. 도메인 간 비교를 위해 검증된 실기체 정상 비행 자료도 함께 제공한다.


## 본 데이터셋의 차별점

저자가 조사한 범위 내에서 본 데이터셋은 다음 세 가지를 동시에 만족하는 첫 공개 UAV MAVLink 데이터셋이다.

### 1. 4채널 동시 캡쳐 (★ 핵심)

기존 공개 데이터셋(RflyMAD, ALFA 등)은 단일 또는 두 채널만 제공한다. 본 데이터셋은 동일 비행에서 다음 4가지 채널을 동시 캡쳐했다.

| 채널 | 물리 경로 | 비율 | 용도 |
|---|---|---|---|
| `drone_to_gcs` | 픽스호크 USB-C → QGC (다운링크) | ~566 msg/s | 드론 풀 텔레메트리 |
| `gcs_to_drone` | QGC → 픽스호크 USB-C (업링크) | ~5–50 msg/s | GCS 명령 |
| `drone_to_rpi` | 픽스호크 TELEM1 UART → RPi | ~12 msg/s | **온보드 IDS 입력** |
| `attacker_to_drone` | Ubuntu 공격자 → 픽스호크 (UDP 14550) | 공격에 따라 | 공격 패킷 자체 |

이 구성으로 사용자는 **같은 비행에서 같은 공격이 채널별로 어떻게 다르게 나타나는가**를 정량적으로 분석할 수 있다. 이는 단일 채널 데이터셋이 불가능했던 분석이다.

### 2. NTP 검증된 시간 정렬

두 시스템(RPi 4, Ubuntu VM) 모두 `timedatectl status`로 NTP 동기화를 직접 검증했다. NTP 동기화된 timestamp를 바탕으로 4채널을 한 파일(`merged_all.jsonl`)에 시간 정렬하여 제공한다. 사용자는 별도 보정 없이 즉시 다채널 분석을 시작할 수 있다.

### 3. 풍부한 메타데이터 + 정직한 한계 명시

각 run의 `meta.json`에 가용 채널, .tlog 커버리지 유형, 추가 자료 보유 여부를 명시한다. 데이터 품질 문제로 제외된 4 runs를 정직히 명시하여 사용자가 학술 연구에 활용할 때 신뢰성을 보장한다.


## 데이터셋 구성 요약

```
HITL-UAV-Dataset/
├── hitl/                              # Pixhawk 6C Mini + Gazebo HITL 비행 자료
│   ├── hover/                         # 정지 비행
│   │   ├── normal/                    # 17 runs (4/20: 10, 4/24: 7)
│   │   └── attack/
│   │       ├── disarm/                # 3 runs
│   │       ├── flooding/              # 3 runs
│   │       ├── pid_tamper/            # 3 runs
│   │       ├── tampering/             # 3 runs
│   │       └── malformed/             # 3 runs
│   │
│   └── mission/                       # 사각형 경로 mission_rect 비행
│       ├── normal/                    # 9 runs
│       └── attack/                    # ★ 14 runs 모두 4채널
│           ├── flooding/              # 3 runs
│           ├── pid_tamper/            # 3 runs
│           ├── tampering/             # 5 runs
│           └── malformed/             # 3 runs
│
├── real_flight/                       # 실기체 검증 정상 비행 (RPi 캡쳐 없음)
│   ├── 2025-03-06_164436/             # 14분 비행
│   └── 2025-03-12_163440/             # 6분 비행
│
├── README.md
└── LICENSE
```

총 **HITL 57 runs + 실기체 2 sessions**.

### 시나리오별 자료 가용성 정직 명시

각 run의 자료 보유 상태가 다르다. 사용자는 `meta.json`의 `channels_available` 필드로 확인할 수 있다.

| 카테고리 | Total | RPi only | + .tlog (full/pre_*) | + attacker | 4채널 모두 |
|---|---|---|---|---|---|
| HITL hover normal | 17 | 12 | 5 | – | – |
| HITL hover attack | 15 | – | – | 15 | – |
| HITL mission normal | 9 | 1 | 8 | – | – |
| HITL mission attack | 14 | – | – | – | **14 (★ 핵심)** |
| Real flight | 2 | – | 2 | – | – |

본 데이터셋의 **다채널 분석 핵심 자료는 mission_attack 14 runs**이다. 모든 4채널이 동시 캡쳐되어 있어 채널별 공격 발현 차이를 직접 비교할 수 있다.


## 각 Run 디렉토리 구조

```
<run_id>/
├── drone_to_gcs/downlink.jsonl         # 드론→GCS 다운링크 (.tlog srcSystem=1 추출)
├── gcs_to_drone/uplink.jsonl           # GCS→드론 업링크 (.tlog srcSystem=255 추출)
├── drone_to_rpi/telem1.jsonl           # 드론→RPi TELEM1 (★ IDS 입력)
├── attacker_to_drone/attacker_udp.jsonl # 공격자→드론 UDP 14550 (공격 비행에만)
├── tlog_original/<datetime>.tlog       # 원본 .tlog (변환 안 한 보존본)
├── merged.jsonl                         # 양방향 통신 시간 정렬
├── merged_all.jsonl                     # 모든 채널 시간 정렬 (★ 다채널 분석 핵심)
├── features.csv                         # 18개 엔지니어링 피처 (4/20 hover만)
└── meta.json                            # 메타데이터
```

각 run의 가용 자료는 `meta.json`의 `channels_available`, `has_merged`, `has_merged_all`, `has_features`, `tlog_coverage` 필드로 명시된다.


## 데이터 채널 상세

### 1. drone_to_gcs/downlink.jsonl (드론 → GCS 다운링크)

- **물리 경로**: 픽스호크 USB-C → Ubuntu VM (QGC)
- **추출 방법**: `.tlog` 원본에서 `srcSystem=1` 메시지만 분리
- **메시지 비율**: 약 566 msg/s (USB-C 풀 대역폭)
- **메시지 타입**: 풀 PX4 텔레메트리 (`HIL_ACTUATOR_CONTROLS`, `ATTITUDE_QUATERNION`, `HIGHRES_IMU`, `ODOMETRY`, `LOCAL_POSITION_NED`, `BATTERY_STATUS`, `GPS_RAW_INT`, `SYS_STATUS`, `EXTENDED_SYS_STATE` 등 모든 메시지)

### 2. gcs_to_drone/uplink.jsonl (GCS → 드론 업링크)

- **물리 경로**: Ubuntu VM (QGC) → 픽스호크 USB-C
- **추출 방법**: `.tlog` 원본에서 `srcSystem=255` 메시지만 분리
- **메시지 타입**: GCS 명령 (`HEARTBEAT`, `COMMAND_LONG` (ARM, TAKEOFF 등), `COMMAND_INT`, `MISSION_ITEM`, `MISSION_ITEM_INT`, `PARAM_SET`, `PARAM_REQUEST_LIST`, `TIMESYNC`, `PING` 등)

### 3. drone_to_rpi/telem1.jsonl (★ 본 IDS 입력 채널)

- **물리 경로**: 픽스호크 TELEM1 (UART, 4핀) → Raspberry Pi 4 GPIO UART
- **수집 도구**: `collector_v2.py` (Python, pymavlink)
- **베이스 레이트**: 57600 baud
- **메시지 비율**: 약 12 msg/s
- **메시지 타입**: 14가지로 제한 (`HIL_ACTUATOR_CONTROLS`, `ATTITUDE`, `GLOBAL_POSITION_INT`, `VFR_HUD`, `BATTERY_STATUS`, `GPS_RAW_INT`, `HEARTBEAT`, `SYS_STATUS`, `ALTITUDE`, `HIGHRES_IMU`, `LOCAL_POSITION_NED`, `COMMAND_ACK`, `ESTIMATOR_STATUS` 등)
- **방향**: 단방향 (드론 → RPi)
- **특징**: 본 연구의 온보드 IDS 모델 입력 채널. 제한된 정보 환경에서의 IDS 성능 평가에 적합.

### 4. attacker_to_drone/attacker_udp.jsonl (공격자 → 드론)

- **물리 경로**: Ubuntu VM 공격자 스크립트 → 네트워크 → 픽스호크 (UDP 14550)
- **수집 도구**: `collect_attack_uplink.py` (Python)
- **존재 조건**: 공격 비행에만
- **메시지 비율** (공격 유형별): `flooding` ~11,000 msgs (60초 공격) / `pid_tamper` 12 msgs (단일 PARAM_SET) / `tampering` 12 msgs / `malformed` ~2,370 msgs / `disarm` 다양함

### 5. tlog_original/<datetime>.tlog

- 픽스호크 USB-C 채널의 원본 `.tlog` 파일
- pymavlink (`mavutil.mavlink_connection`)로 직접 파싱 가능
- 학술적 reproducibility를 위해 보존

### 6. merged.jsonl (양방향 통신 시간 정렬)

- 양방향 통신을 캡쳐한 채널을 시간 정렬한 통합 파일
- run 카테고리별 정의:
  - **HITL 공격 비행** (hover_attack, mission_attack): `drone_to_rpi` + `attacker_to_drone`
  - **HITL 정상 비행** (hover_normal 4/24, mission_normal): `drone_to_gcs` + `gcs_to_drone`
  - **Real flight**: `drone_to_gcs` + `gcs_to_drone`
- 각 메시지에 `direction` 필드 ("downlink" / "uplink") 추가

### 7. merged_all.jsonl (★ 다채널 시간 정렬 — 본 데이터셋의 핵심)

- 가용한 모든 채널의 메시지를 timestamp 기준으로 정렬한 통합 파일
- 각 메시지에 `channel` 필드로 출처 채널 명시
- **사용자가 한 파일에서 4채널을 동시에 분석 가능** — 본 데이터셋의 핵심 활용 자료

예시 (mission_attack flooding):

```json
{"ts": 1777372685.109, "msg_type": "TIMESYNC", "channel": "drone_to_gcs", ...}
{"ts": 1777372749.606, "msg_type": "ATTITUDE", "channel": "drone_to_rpi", ...}
{"ts": 1777372758.064, "msg_type": "COMMAND_LONG", "channel": "attacker_to_drone", ...}
{"ts": 1777372758.500, "msg_type": "HEARTBEAT", "channel": "gcs_to_drone", ...}
```

### 8. features.csv (엔지니어링 피처)

- 본 연구에서 사용한 18개 물리/통신 피처를 윈도우 단위로 추출
- **4/20 HITL hover 25 runs에만 제공** (본 IDS 모델 학습용)
- 사용자는 본인 연구에 맞는 피처를 직접 추출하여 사용 권장

### 9. meta.json

```json
{
  "run_id": "20260428_193909_8fe27a",
  "domain": "hitl",
  "scenario": "mission_rect",
  "attack_type": "flooding",
  "label": "attack",
  "start_time": "2026-04-28T19:39:09.466794",
  "end_time": "2026-04-28T19:42:50.424097",
  "duration_sec": 220.9,
  "total_messages": 8340,
  "channels_available": ["drone_to_gcs", "gcs_to_drone",
                         "drone_to_rpi", "attacker_to_drone"],
  "tlog_coverage": "pre_attack",
  "has_merged": true,
  "has_merged_all": true,
  "has_features": false
}
```


## .tlog 커버리지 유형

각 run의 `meta.json`에 `tlog_coverage` 필드로 명시된다.

| Coverage | 의미 | 해당 runs (총) |
|---|---|---|
| `full` | QGC가 비행 전체 동안 켜져 있어 비행 전체 캡쳐 | mission_normal 8 + hover_normal 1 + real_flight 2 = **11** |
| `pre_flight` | QGC로 ARM, TAKEOFF 명령 후 RPi 시작 시점에 QGC 종료 | hover_normal 4/24의 4 runs = **4** |
| `pre_attack` | 비행 시작 후 공격 직전 (QGC 종료) 시점까지 | mission_attack 14 runs = **14** |
| `none` | `.tlog` 없음 | hover_normal 4/20 (10) + hover_normal 4/24 미커버(2) + hover_attack 15 + mission_normal 1 = **28** |

`pre_flight`(50–95s)에는 ARM 2회 + TAKEOFF 2회 명령과 텔레메트리 응답이 모두 보존되어 비행 준비 단계의 양방향 통신 분석이 가능하다. `pre_attack`(5–25s)는 공격 비행에서 UDP 14550 포트 충돌 방지를 위해 공격 직전 QGC를 종료한 결과이다.


## 시간 정렬 — NTP 검증 명시

### 시계 환경

본 데이터셋의 timestamp는 두 가지 시계를 사용한다:

- **RPi 4 시계**: `drone_to_rpi/telem1.jsonl`
- **Ubuntu VM 시계**: `drone_to_gcs/`, `gcs_to_drone/`, `attacker_to_drone/`, `tlog_original/`

### NTP 동기화 직접 검증

두 시스템 모두 `timedatectl status`로 NTP 동기화를 직접 검증했다.

```
RPi:        System clock synchronized: yes, NTP service: active
Ubuntu VM:  System clock synchronized: yes, NTP service: active
```

두 시스템 모두 KST (Asia/Seoul, +0900) 시간대로 동기화되어 있다.

### 검증된 시간 차이 (한 샘플 run)

`hitl/mission/attack/flooding/20260428_193909_8fe27a` run에서 채널별 시작 시점:

```
.tlog 시작:       1777372685.109  (T+0)        ← QGC 켜기
RPi 시작:         1777372749.606  (T+64.5)     ← 안정 비행 후 RPi 시작
.tlog 종료:       1777372755.362  (T+70.3)     ← QGC 끄기
UDP 시작:         1777372758.064  (T+72.9)     ← 공격 시작
RPi 종료:         1777372970.413  (T+285.3)
```

`.tlog`와 RPi의 시작 시점 64.5초 차이는 **본 실험 절차상 차이** (.tlog 시작 후 ARM, TAKEOFF, 안정 비행 진입까지 약 64초)이며, **시계 차이가 아니다**. NTP 동기화된 시계의 차이는 1초 미만으로 보장된다.

### 정렬 방법

`merged.jsonl`과 `merged_all.jsonl`은 각 채널의 원본 timestamp를 보정 없이 단순 정렬한다. 각 메시지에 `channel`(또는 `direction`) 필드로 출처를 명시한다.

### 정직히 알려야 할 한계

- Timestamp는 "**캡쳐 도구가 메시지 받은 시점**"이다. "드론이 메시지 보낸 시점"이 아니다.
- UART (TELEM1, 57600 baud), USB-C, UDP의 전송 지연으로 메시지 시점에 약 ±10 ms 오차가 있을 수 있다.
- 정밀 ms 단위 분석이 필요한 경우 사용자가 같은 메시지를 두 채널에서 비교하여 추가 보정을 권장한다.


## 하드웨어/소프트웨어 환경

| 구성 요소 | 사양 |
|---|---|
| 비행 제어기 | Pixhawk 6C Mini (PX4 SITL, Gazebo HITL) |
| 온보드 컴퓨터 | Raspberry Pi 4 Model B (4GB) |
| TELEM1 UART | 57600 baud, 14 message types |
| GCS | QGroundControl (Ubuntu VM, USB-C 연결) |
| 공격자 | Ubuntu VM, pymavlink 기반 공격 스크립트 |
| 시뮬레이터 | Gazebo Classic + PX4 SITL HITL |


## 공격 유형 (5종)

| 공격 유형 | 설명 | 일반 업링크 메시지 수 | 시나리오 |
|---|---|---|---|
| `flooding` | 고속 명령 플러딩 (60초) | ~11,000 msgs | hover, mission |
| `pid_tamper` | PARAM_SET으로 PID 파라미터 변조 | 12 msgs | hover, mission |
| `tampering` | PARAM_SET으로 다양한 파라미터 변조 | 12 msgs | hover, mission |
| `malformed` | 잘못된 형식의 MAVLink 패킷 | ~2,370 msgs | hover, mission |
| `disarm` | DISARM 명령 공격 | 다양함 | hover만 |

`disarm` 공격은 mission 시나리오에서는 의도적으로 제외했다. mission 비행은 4가지 공격 유형(flooding, pid_tamper, tampering, malformed)에 집중하였다.


## 데이터 수집 절차

표준 절차는 다음과 같다.

1. 드론 부팅 (Gazebo HITL + Pixhawk 6C Mini SITL)
2. QGC 시작 및 드론 USB-C 연결 (`.tlog` 기록 시작)
3. QGC를 통해 ARM, TAKEOFF 명령 전송
4. 안정적인 비행 상태 (hover 정지 또는 mission 진입) 대기
5. 안정화 후 약 15초 뒤 RPi에서 `collector_v2.py` 실행 (`telem1.jsonl` 기록 시작)
6. **정상 mission 비행**: QGC를 비행 종료까지 계속 켜둠
7. **공격 비행**: QGC를 종료(`.tlog` 종료) 후 Ubuntu VM에서 `collect_attack_uplink.py` 실행
8. 비행 진행, RPi가 비행 전체를 기록
9. 착륙 후 RPi 종료, 공격 스크립트 실행 중이면 종료

위 절차의 차이가 `.tlog` 커버리지 세 가지 유형(`full`, `pre_flight`, `pre_attack`)을 만든다.


## 사용 사례

### 사례 1: 단일 채널 IDS 학습/평가 (본 연구의 사용법)

```python
import json
# 드론→RPi TELEM1 채널만 사용 (12 msg/s, 14 types)
msgs = [json.loads(line) for line
        in open('hitl/mission/attack/flooding/20260428_193909_8fe27a/'
                'drone_to_rpi/telem1.jsonl')]
print(f"메시지 수: {len(msgs)}")
```

### 사례 2: ★ 다채널 공격 발현 비교 (본 데이터셋의 핵심 활용)

```python
import json
from collections import Counter

# 4채널이 시간 정렬된 통합 파일 로드
msgs = [json.loads(line) for line
        in open('hitl/mission/attack/flooding/20260428_193909_8fe27a/'
                'merged_all.jsonl')]

# 채널별 분포 확인
channel_dist = Counter(m['channel'] for m in msgs)
print(channel_dist)
# Counter({'drone_to_gcs': 39929,    # 드론 풀 텔레메트리
#          'attacker_to_drone': 11420, # 공격 패킷
#          'drone_to_rpi': 8340,       # RPi 캡쳐 (제한된 정보)
#          'gcs_to_drone': 233})       # GCS 정상 명령

# 공격 시점 채널별 메시지 빈도 분석
attack_start = 1777372758.064  # UDP 시작 시점
for ch in channel_dist:
    ch_msgs_during_attack = [m for m in msgs
                              if m['channel'] == ch
                              and m['ts'] >= attack_start]
    print(f"  {ch}: 공격 동안 {len(ch_msgs_during_attack)} msgs")
```

### 사례 3: 양방향 통신 분석

```python
import json
msgs = [json.loads(line) for line
        in open('.../merged.jsonl')]
# direction 필드로 양방향 구분
downlink = [m for m in msgs if m.get('direction') == 'downlink']
uplink = [m for m in msgs if m.get('direction') == 'uplink']
```

### 사례 4: 원본 .tlog 직접 파싱

```python
from pymavlink import mavutil
m = mavutil.mavlink_connection(
    '.../tlog_original/2026-04-28 19-39-15.tlog')
while True:
    msg = m.recv_match()
    if msg is None: break
    # 사용자 정의 처리
```

### 사례 5: Cross-domain 비교 (HITL vs 실기체)

```python
import json
# HITL hover normal 자료
hitl_msgs = [json.loads(line) for line
             in open('hitl/hover/normal/.../drone_to_rpi/telem1.jsonl')]

# 실기체 정상 비행 자료
real_msgs = [json.loads(line) for line
             in open('real_flight/2025-03-06_164436/drone_to_gcs/downlink.jsonl')]

# Domain gap 분석
```


## 데이터 품질 검증 — 정직히 제외된 Run

데이터 품질 보장을 위해 다음 4 runs를 학술적 정직성 차원에서 제외하였다.

| Run ID | 제외 사유 |
|---|---|
| `20260424_173322_88a384` | `raw.jsonl` JSON 파싱 에러 (한 줄 이상 손상) |
| `20260424_172105_116f93` | 비행 시간 28.6초 (mission 미완수, 실험 실패 추정) |
| `20260424_183921_4cec90` | 비행 시간 73초 (mission 미완수 추정) |
| `20260428_192557_ecea9a` | `pid_tamper`로 라벨되었으나 COMMAND_ACK 2,369개 (정상 12 msgs와 큰 차이), 라벨 오류 또는 다른 공격 혼합 가능성 |

이러한 제외 결정은 본인 데이터를 학술적으로 신뢰성 있게 활용 가능하도록 하기 위함이다.


## 본 데이터셋이 가능하게 하는 학술 연구

1. **Single-channel IDS 평가**: `drone_to_rpi/telem1.jsonl`만 사용. 제한된 정보 환경에서의 온보드 IDS 성능 평가.
2. **Multi-channel IDS 학습**: `merged_all.jsonl` 사용. 여러 채널 결합 시 성능 향상 정량 분석.
3. **★ Per-channel Attack Manifestation 분석**: 같은 공격이 채널별로 다른 시그니처를 보이는지 분석. 본 데이터셋이 처음으로 가능하게 한 분석.
4. **Onboard vs Offboard IDS 비교**: 온보드 (TELEM1, 12 msg/s) vs 오프보드 (USB-C, 566 msg/s) IDS 비교. 동일 비행, 동일 공격 조건에서 직접 비교 가능.
5. **Cross-domain 분석**: HITL (시뮬레이션) vs Real flight (실기체). Domain gap 측정 및 보완 방법 연구.
6. **Pre-flight Phase 보안 연구**: `pre_flight` .tlog로 비행 준비 단계의 양방향 명령 흐름 분석. ARM, TAKEOFF 단계 공격 시그니처 연구.

## 약어 안내

- **HITL**: Hardware-in-the-Loop (시뮬레이션 환경에 실제 비행 제어기 + 
  실제 온보드 컴퓨터를 연결하는 시뮬레이션 방식)
- **RPi**: Raspberry Pi
- **GCS**: Ground Control Station (QGroundControl)
- **TELEM1**: Pixhawk의 텔레메트리 출력 UART 포트
- **MAVLink**: Micro Air Vehicle Link
- **NTP**: Network Time Protocol

## 인용

```
(인용 형식 추후 추가 — 학위 논문 또는 SCI 저널 게재 후)
```


## 라이선스

MIT License — `LICENSE` 파일 참조.


## 변경 이력

- **v3 (현재)**: 4채널 완전 분리 (`drone_to_gcs/`, `gcs_to_drone/`, `drone_to_rpi/`, `attacker_to_drone/`) + `tlog_original/` 직관적 명명 + `merged_all.jsonl` 다채널 시간 정렬 추가 + NTP 검증 직접 명시 + Git LFS
- **v2**: `hitl/{hover,mission}/{normal,attack}` 계층 구조 + `qgc_usb*/` .tlog 분류
- **v1**: 4-perspective 구조 (gcs_to_drone_uplink, drone_to_gcs_downlink, bidirectional_merged, features) + `hitl_extended/` 분리


## 기여 / 문의

데이터셋 사용 중 발견한 오류, 개선 제안, 협업 문의는 GitHub Issues로 부탁드립니다.
