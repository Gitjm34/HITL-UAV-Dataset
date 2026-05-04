# UAV MAVLink IDS Dataset (HITL, 2026)

A bidirectional MAVLink raw archive dataset for UAV intrusion detection 
research, collected in a PX4-based Hardware-In-The-Loop (HITL) environment 
with Pixhawk 6C Mini and Raspberry Pi 4 onboard companion computer.

## Highlights

- **25 runs**: 10 normal flights + 15 attack flights
- **5 attack types**: flooding, pid_tamper, tampering, disarm, malformed
- **All attacks at MAVLink protocol layer (L7)**, implemented via pymavlink
- **4 dataset perspectives** for different research needs:
  1. GCS-to-drone uplink (attacker side, raw)
  2. Drone-to-GCS downlink (RPi onboard side, raw)
  3. Bidirectional time-merged archive
  4. Pre-extracted feature vectors (18-dim, 1-second windows)

## Dataset Structure

public_dataset/
├── gcs_to_drone_uplink/         # GCS → drone direction (raw uplink)
│   └── attack/
│       ├── disarm/        (3 runs)
│       ├── flooding/      (3 runs)
│       ├── malformed/     (3 runs)
│       ├── pid_tamper/    (3 runs)
│       └── tampering/     (3 runs)
│
├── drone_to_gcs_downlink/       # drone → GCS direction (raw downlink)
│   ├── normal/            (10 runs)
│   └── attack/            (15 runs, 5 types × 3)
│
├── bidirectional_merged/        # time-synchronized bidirectional archive
│   ├── normal/            (10 runs, downlink-only)
│   └── attack/            (15 runs, uplink + downlink merged)
│
├── features/                    # pre-extracted 18-dim features (1-sec windows)
│   ├── normal/            (10 runs)
│   └── attack/            (15 runs, 5 types × 3)
│
└── runs.csv                     # metadata index for all 25 runs

Each run directory contains:
- `uplink.jsonl` (uplink-only) or `raw.jsonl` (downlink-only) or 
  `merged.jsonl` (bidirectional) or `features.csv` (pre-extracted features)
- `meta.json` (run metadata)

## Why Four Perspectives?

The same flight session is observed from different vantage points, each 
useful for different research questions:

- **Uplink-only**: study attacker-side traffic patterns and signature-based 
  detection
- **Downlink-only**: study onboard IDS scenarios where uplink is not directly 
  observable (e.g., RF-isolated attacker)
- **Bidirectional merged**: study command-response relationships, ACK 
  matching, and protocol-level dialogue analysis
- **Features**: ready-to-use input for machine learning experiments without 
  re-implementing feature extraction

Note that the same PX4 output reaches different observers at different 
stream rates due to channel bandwidth limits:
- USB to PC (used by GCS): ~69 msg/s
- TELEM1 UART to RPi: ~12 msg/s

This bandwidth-induced rate difference is preserved in the dataset and is 
relevant for cross-perspective generalization studies.

## Attack Types

All 5 attacks are MAVLink protocol-layer (L7) attacks executed against the 
Pixhawk over UDP 14550 from a separate Ubuntu VM running the attack script.

| Attack | Description | Avg Uplink | Avg Downlink |
|---|---|---|---|
| flooding | High-rate COMMAND_LONG flooding | ~11,080 | ~8,430 |
| disarm | MAV_CMD_COMPONENT_ARM_DISARM injection | ~597 | ~2,838 |
| pid_tamper | PARAM_SET on PID gains (MC_PITCH_P, MC_PITCH_I) | 12 | ~2,562 |
| tampering | PARAM_SET on attitude param (MC_YAW_P) | 12 | ~2,582 |
| malformed | Malformed MAVLink packet injection | 2,380 | ~4,910 |

## Phase Structure

Each attack run consists of three phases (210 seconds total):
- `normal` (30s): pre-attack baseline
- `attack` (60s): active attack period
- `post` (120s): post-attack recovery

Phase boundaries are explicitly marked in uplink.jsonl and merged.jsonl 
via `PHASE_START` and `PHASE_END` records, enabling sub-second ground truth 
labeling. The features.csv files include a `label_detail` column with values 
`pre_attack`, `attack`, or `post_attack`.

## Data Format

### Raw archives (jsonl)

Each line is a JSON object:

```json
{
  "t": 1776673013.30,
  "type": "PHASE_START",
  "data": {"phase": "attack", "duration_sec": 60},
  "direction": "uplink",
  "source": "ubuntu_attacker"
}
```

Fields:
- `t`: Unix timestamp (seconds, microsecond precision)
- `type`: MAVLink message type or pipeline event
- `data`: Message payload (raw fields)
- `direction`: `uplink` or `downlink` (in merged archives only)
- `source`: `ubuntu_attacker` or `rpi_telem1` (in merged archives only)

### Features (csv)

Each row is a 1-second window with 18 features plus metadata:

**Physical features (14)**: roll/pitch/yaw mean+std, rollspeed_std, 
pitchspeed_std, yawspeed_std, groundspeed, alt, climb, throttle, heading

**Communication features (4)**: msg_count_in_window, 
attitude_count_in_window, cmd_ack_count, param_value_count

The first window of each run may contain empty values for VFR_HUD-derived 
features (groundspeed, climb, throttle, heading) because these messages 
arrive at ~2-second intervals and the first 1-second window may not yet 
contain one.

## Hardware and Software

- **Flight controller**: Pixhawk 6C Mini (PX4 firmware)
- **Companion computer**: Raspberry Pi 4
- **Simulation**: Gazebo HITL via PX4 SITL bridge (Ubuntu VM)
- **GCS**: QGroundControl (terminated during attacks to avoid UDP 14550 
  conflict with the attack script)
- **Collector versions**: 
  - Ubuntu side: `collect_attack_uplink.py` (custom)
  - RPi side: `collector_v2.py` v2.1 (TELEM1 UART, 57600 bps)

## Reproducibility

This dataset corresponds to the experimental data used in the master's 
thesis "PX4/MAVLink-based Onboard AI Intrusion Detection System" 
(MES Lab, 2026). All quantitative results in the thesis can be reproduced 
from this archive.

Per-attack run counts match Section 5.9.8 of the thesis (coefficient of 
variation ≤ 5.4% across 3 repeated runs per attack type).

## Limitations

- All runs share a single hover scenario; mission flights are not included.
- Attacks are MAVLink protocol-layer (L7) only. Signal-layer (L1) attacks 
  such as GPS spoofing or GNSS jamming are out of scope and are not 
  included in this dataset.
- Normal flight runs do not include uplink data, as the GCS was used only 
  for monitoring during normal flights without active command transmission. 
  Only attack runs include uplink data, captured by the attack script on 
  the Ubuntu VM side.
- The first 1-second window of each run may contain empty values for some 
  features (see Data Format section).

## Citation

If you use this dataset in your research, please cite:

[Citation TBD - master's thesis publication forthcoming, MES Lab 2026]

## License

MIT License. See LICENSE file for details.

## Contact

For questions or issues, please open an issue in this repository.
