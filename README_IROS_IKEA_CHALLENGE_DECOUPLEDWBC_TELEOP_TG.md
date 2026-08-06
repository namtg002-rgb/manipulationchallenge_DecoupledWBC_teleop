# IROS IKEA Challenge DecoupledWBC Teleop TG

이 문서는 `robofinals-sonic-planner` 컨테이너에서 IROS IKEA manipulation challenge를
Decoupled WBC + PICO VR + Isaac Sim 6 + WebRTC + LeRobot v3 구성으로 실행하는 최종 순서다.

## 1. 현재 구성

| 구성 | 목표 주기 | CPU | 비고 |
|---|---:|---:|---|
| PhysX authority | 200 Hz | 8-15 | headless, 카메라 없음 |
| Decoupled WBC | 50 Hz | 8-15 | action 하나당 PhysX 4회 |
| PICO latest reader | 90 Hz | 6 | SDK 최신값만 보관 |
| Upper-body IK worker | 50 Hz | 16-19 | ZMQ HWM=1, latest-only |
| 4-view RTX viewer | 실측 29.7~30.2 Hz | 0-5 | 중앙 3인칭 + head + 양 wrist |
| LeRobot writer/NVENC feeder | 실측 29.7~30.2 Hz | 20-21 | bounded queue 32 |
| XRoboToolkit service | PICO 입력 주기 | 22 | 호스트에서 실행 |

G1 시작 상태는 다음과 같다.

```text
root_xyz=(-2.1, 2.4, 0.78)
root_quat_xyzw=(0, 0, 0, 1)
fix_root_link=False
```

G1은 upright 상태로 바닥에 서며 world `+X` 방향의 테이블을 본다. Authority와 viewer는
모두 Isaac Sim 6의 `xyz+xyzw` pose를 사용하고 viewer에서 quaternion을 다시 재배열하지 않는다.

## 2. 완전 초기화

다음과 같은 경우 호스트 터미널에서 실행한다.

- 새로운 작업을 완전히 깨끗한 상태에서 시작할 때
- WebRTC client가 검은 화면에서 연결되지 않을 때
- `NVST_R_BUSY`, `NVST_R_INVALID_OPERATION` 또는 streamserver 시작 실패가 반복될 때
- 이전 viewer/Authority/PICO 프로세스나 `49200`, `48000`, `5580` 포트가 남았을 때
- viewer frame 또는 로봇 상태가 이전 실행 상태에 붙어 있다고 의심될 때

이 명령은 WebRTC client를 종료하고, planner 컨테이너 안의 Decoupled/PICO/Isaac 프로세스와
관련 포트를 정리한 다음 컨테이너를 재시작한다. 정상 실행 중에는 사용할 필요가 없으며,
실행하면 현재 simulation과 저장하지 않은 episode는 종료된다.

> **데이터 수집 후 주의:** LeRobot recorder를 한 번이라도 사용했다면 이 명령을 바로
> 실행하지 않는다. 먼저 viewer를 실행한 **터미널 2에서 `Ctrl+C`를 한 번만 누르고** 아래
> 로그가 출력될 때까지 기다린다.
>
> ```text
> [LeRobot] dataset finalize()
> ```
>
> 이 로그를 확인한 뒤에만 아래 완전 초기화를 실행한다. 강제 종료하면 MP4만 남고
> `data/*.parquet` footer와 `meta/episodes/*.parquet`가 기록되지 않아 데이터셋이 손상될 수
> 있다. viewer가 완전히 멈춰 정상 종료가 불가능할 때만 강제 초기화를 최후 수단으로 쓴다.

```bash
pkill -KILL -f '[i]saacsim-webrtc-streaming-client' 2>/dev/null || true
bash /home/user/Desktop/stop_decoupled_all_host.sh
docker restart robofinals-sonic-planner
```

### X11 인증 쿠키 동기화 (컨테이너 재시작 직후 필수)

호스트에 다시 로그인하거나 Xorg가 재시작되면 X11의 MIT-MAGIC-COOKIE가 바뀐다. 컨테이너의
`/root/.Xauthority`는 자동 갱신되지 않으므로, 위 `docker restart` 직후 또는 다음 오류가
발생했을 때 현재 호스트 쿠키를 복사한다.

```text
Invalid MIT-MAGIC-COOKIE-1 key
failed to acquire X connection
Can't connect to display ":1"
```

호스트 터미널에서 실행한다.

```bash
XAUTH_SRC="${XAUTHORITY:-/run/user/$(id -u)/gdm/Xauthority}"

docker cp "$XAUTH_SRC" \
  robofinals-sonic-planner:/root/.Xauthority

docker exec robofinals-sonic-planner \
  chmod 600 /root/.Xauthority
```

인증만 검증하려면 다음 명령을 사용한다.

```bash
docker exec robofinals-sonic-planner bash -lc '
  DISPLAY=:1 \
  XAUTHORITY=/root/.Xauthority \
  /opt/conda/envs/robofinals/bin/python -c "
from pynput.keyboard import Key
print(\"[OK] planner container X11 authentication\")
"
'
```

`[OK] planner container X11 authentication`이 출력된 다음 XRoboToolkit와 PICO preflight로
진행한다. `xhost +`처럼 모든 로컬 사용자의 X 접근을 여는 방식은 사용하지 않는다.

컨테이너가 올라왔는지 확인한다.

```bash
docker ps --filter name=robofinals-sonic-planner
```

기존 프로세스와 포트를 확인한다.

```bash
docker exec robofinals-sonic-planner bash -lc '
  pgrep -af "teleop_main.py|decoupled_visual_viewer|decoupled_upper_ik_worker" \
    || echo "[OK] no old Decoupled process"
'

ss -lntup | grep -E ':(49200|48000|5567|5568|5580|60061)\b' \
  || echo '[OK] Decoupled/WebRTC ports clear'

nvidia-smi \
  --query-gpu=encoder.stats.sessionCount,encoder.stats.averageFps \
  --format=csv,noheader
```

Isaac/WebRTC 프로세스를 실행하기 전 encoder session은 보통 `0, 0`이어야 한다. NoMachine이
encoder 하나를 사용하는 환경에서는 `nvidia-smi pmon -c 1`로 owner를 따로 확인한다.

## 3. XRoboToolkit와 PICO preflight

호스트에서 XRoboToolkit 서비스를 먼저 실행한다.

```bash
bash /home/user/Desktop/start_decoupled_xrt_host.sh
```

정상 출력:

```text
[OK] XRoboToolkit listening; pid=... cpu=22 log=...
```

그다음 PICO XRoboToolkit 앱에서 다음 순서로 연결한다.

1. PICO와 양쪽 controller를 켠다.
2.  PICO에서 IP를 `192.168.0.4`으로 선택한다.
3. Head와 Left/Right Controller 전송을 켠다.
4. 상태가 `WORKING`이 되면 `Send`를 누른다.
5. 실행 중 `.4`와 `.6`을 번갈아 사용하지 않는다.

PICO를 착용한 상태에서 호스트 터미널에 입력한다.

```bash
docker exec -it robofinals-sonic-planner bash -lc '
  cd /workspace/robofinals-sonic
  exec ./scripts/decoupled_wbc/preflight_decoupled_wbc.sh
'
```

다음 항목이 모두 나와야 한다.

```text
[OK] G1 USD payloads + stand/walk ONNX are real files
[OK] Isaac Sim 6.0.0.1
[OK] LeRobot ... (v3 writer API)
[OK] external latest-only upper IK worker import
[OK] host XRoboToolkit is listening on 127.0.0.1:60061
[OK] finite headset + two controller poses
[OK] Decoupled WBC preflight complete
```

## 4. 터미널 1: Authority

```bash
docker exec -it robofinals-sonic-planner bash -lc '
  cd /workspace/robofinals-sonic
  exec ./scripts/decoupled_wbc/run_decoupled_authority.sh
'
```

2~3회 로그를 보고 다음 상태인지 확인한다.

```text
[DECOUPLED TIMING] loop_hz=195~200 physics_hz=195~200 wbc_hz=48.8~50.0
[DECOUPLED ROOT] is_fixed_base=False runtime_quat_layout=xyzw
[DECOUPLED STATE] PUB tcp://0.0.0.0:5580 latest-only 30Hz
```

Authority가 안정된 다음에만 터미널 2를 실행한다.

## 5. 터미널 2: 4-view viewer와 WebRTC server

```bash
docker exec -it robofinals-sonic-planner bash -lc '
  cd /workspace/robofinals-sonic
  DECOUPLED_WEBRTC_PUBLIC_IP=192.168.0.4 \
  exec ./scripts/decoupled_wbc/run_decoupled_viewer.sh
'
```

정상 시작 로그:

```text
[VIEWER ROOT POSE] wire/runtime=xyz+xyzw applied_without_reorder
[VIEWER] 4-view RTX/WebRTC/cameras=30Hz
[CAMERA DATA READY] ... @viewer-clock(30Hz)
[LeRobot] PICO-button v3 capture ready ...
```

포트와 viewer 진행 상태를 확인한다.

```bash
ss -lntp | grep ':49200'

docker exec robofinals-sonic-planner bash -lc '
  log=$(ls -1t /workspace/robofinals-sonic/logs/decoupled_wbc/viewer_*.log | head -1)
  tail -F "$log"
' | grep --line-buffered -E '\[VIEWER\]|\[CAMERA DATA\]|\[VIEWER PHASE\]|\[LeRobot\]'
```

## 6. 터미널 3: WebRTC client

터미널 2에서 `49200 LISTEN`과 증가하는 viewer frame을 확인한 후 실행한다.

```bash
pkill -TERM -f '[i]saacsim-webrtc-streaming-client' 2>/dev/null || true
sleep 2
unset ELECTRON_RUN_AS_NODE
WEBRTC_PROFILE="$(mktemp -d /tmp/isaac-webrtc-client.XXXXXX)"

"/opt/Isaac Sim WebRTC Streaming Client/isaacsim-webrtc-streaming-client" \
  --user-data-dir="$WEBRTC_PROFILE"
```

Client 입력값:

```text
Server:     192.168.0.4
Signal:     49200
Stream:     48000
Resolution: 1920 x 1080 (FHD)
```

`4800`이 아니라 반드시 `48000`을 사용한다.

## 7. PICO calibration과 조작

WebRTC에서 G1이 바닥에 서 있고 head camera가 테이블을 보는 것을 먼저 확인한다. 양손
controller를 편안한 중립 위치에 둔 뒤 다음 조합을 한 번 누르고 놓는다.

```text
왼쪽 menu + 오른쪽 trigger 50% 이상
```

대체 입력은 `왼쪽 joystick click + 오른쪽 trigger`다.

정상 로그:

```text
[DECOUPLED PICO INPUT] activation edge ...
[DECOUPLED PICO] calibration captured ... first_target_error=0.0e+00
[DECOUPLED PICO] teleoperation ACTIVE
[DECOUPLED PICO] ... active=True stale=False ...
```

조작:

| 입력 | 동작 |
|---|---|
| left menu + right trigger | calibration 및 teleop 활성/정지 |
| left joystick | root 전후/좌우 이동 |
| right joystick 좌우 | root yaw 회전 |
| left/right trigger | 각 gripper |
| 오른쪽 controller A | 녹화 시작 / 다시 누르면 episode 저장 |
| 오른쪽 controller B | 현재 episode 폐기 |

상태 로그의 `Ljoy`, `Rjoy`, `nav`가 입력과 함께 바뀌어야 한다. `stale=True`이면 PICO
timestamp가 멈춘 것이므로 그 상태에서는 안전하게 stand/hold한다.

## 8. LeRobot v3 데이터 수집

기본 저장 위치:

```text
/mnt/sonic_capture/decoupled_wbc_lerobot_v3
```

수집 전에 이 경로가 실제 호스트 디스크에 bind mount됐는지 확인한다.

```bash
docker exec robofinals-sonic-planner \
  findmnt -T /mnt/sonic_capture
```

`SOURCE`가 `overlay`이면 데이터는 외장 디스크가 아니라 컨테이너 writable layer에 저장되는
상태다. `docker restart`에는 남지만 컨테이너 삭제·재생성 시 유실될 수 있다. 현재 확인된
호스트 저장 디스크는 `/mnt/storage`이며, planner 컨테이너의 `/mnt/sonic_capture`에는 아직
bind mount되어 있지 않다. 장시간 본 수집 전에는 반드시 실제 저장 경로를 다시 확인한다.

저장 내용:

- `observation.state`: G1 joint state
- `action`: Decoupled WBC action
- `observation.images.first_person_camera`
- `observation.images.left_hand_camera`
- `observation.images.right_hand_camera`
- task: `assemble the IKEA table`
- LeRobot v3 metadata/parquet + H.264 NVENC video

3인칭 global camera는 WebRTC 모니터링 전용이며 dataset에는 저장하지 않는다.

수집 순서:

1. calibration 후 `active=True`, `stale=False`를 확인한다.
2. 오른쪽 controller `A`를 한 번 눌러 episode 녹화를 시작한다.
3. 조작을 수행한다.
4. `A`를 다시 누르면 녹화를 끝내고 비동기로 episode를 저장한다.
5. 실패한 episode는 녹화 중 `B`를 눌러 폐기한다.

정상 로그:

```text
[LeRobot] PICO A: recording started
[LeRobot] capture_hz=29.7~30.2 recording=True record_queue=.../32 accepted=... written=... dropped=0
[LeRobot] PICO A: episode queued for save
[LeRobot] save_episode (... frames)
```

### 데이터 수집 세션 정상 종료

마지막 episode를 `A`로 저장한 뒤 viewer 터미널의 `record_queue`가 `0/32`이고
`save_episode`가 출력됐는지 확인한다. 그다음 터미널 2에서 `Ctrl+C`를 한 번 누르고 다음
로그를 기다린다.

```text
[LeRobot] dataset finalize()
```

`finalize()`는 현재 Parquet writer footer와 episode metadata를 닫는 필수 단계다. 이 로그를
확인하기 전에는 `docker restart`, `pkill -KILL`, 컴퓨터 종료를 하지 않는다.

저장 결과 확인:

```bash
docker exec robofinals-sonic-planner bash -lc '
  root=/mnt/sonic_capture/decoupled_wbc_lerobot_v3
  du -sh "$root"
  find "$root" -maxdepth 4 -type f | sort | tail -40
'
```

## 9. 30 Hz 판정: 설정값과 실제값을 구분할 것

현재 목표 설정은 다음 세 곳 모두 30 Hz다.

- visual-state publisher: 30 Hz
- viewer outer loop와 네 카메라 acquisition: 30 Hz
- WebRTC `primaryStream/targetFps`: 30 FPS
- LeRobot dataset metadata: `fps=30`

하지만 실제 source 주기는 반드시 런타임 로그로 판정해야 한다.

```text
[VIEWER] ... render_hz=30.0
[CAMERA DATA] global_camera=30.0Hz first_person_camera=30.0Hz ...
[VIEWER PHASE] ... total_ms=33.33 이하
```

`nvidia-smi`의 encoder FPS가 29~30이어도 source view가 30 Hz라는 뜻은 아니다. NVENC가 같은
source frame을 반복 인코딩할 수 있기 때문이다. 실제 view 자체의 갱신률은 `render_hz`, 실제
카메라 pixel 갱신률은 `[CAMERA DATA]`로 판단한다.

### 2026-08-07 최종 실측 결과

`sonic-pico-teleop-tg` 커밋 `96bfbd5a`의 검증된 GPU 표시 경로를 유지했다.

```text
Replicator RGBA
  -> 재사용 pinned CPU buffer
  -> 재사용 CUDA RGBA buffer
  -> ByteImageProvider.set_bytes_data_from_gpu(pointer)
  -> 4-view UI / WebRTC
```

화면을 CPU에서 1920x1080으로 재합성하지 않는다. `primaryStream/targetFps=30`, Kit 내부
pump=60 Hz, viewport tick=30 Hz이며 최종 출력과 데이터 카메라는 30 Hz로 제한된다.

Decoupled authority의 상판·다리 pose에는 정지 상태에서도 최대 약 18 um/frame의 PhysX
jitter가 있었다. 5개 object xform을 매 frame USD에 다시 쓰면 RTX scene 갱신이 매번 발생해
다음과 같이 느려졌다.

```text
수정 전: app_update=약 33~40 ms, 실제 camera=약 23~26 Hz
```

현재는 마지막으로 표시한 pose 대비 위치 `0.1 mm` 또는 회전 `0.001 rad` 이상 변한 object만
갱신한다. 임계값은 마지막 표시 pose 기준으로 누적 비교하므로 실제 물체 이동은 사라지지 않고
표시 오차만 위 범위 이내로 제한된다.

녹화 OFF 실측:

```text
[CAMERA DATA] 네 camera=29.7~30.2 Hz
[VIEWER PHASE] total_ms=약 26~28 ms
[DECOUPLED TIMING] physics_hz=약 199.7~200.3
```

3-camera LeRobot/NVENC 녹화 ON 장시간 표본 실측:

```text
[CAMERA DATA] 네 camera=29.7~30.2 Hz
[LeRobot] capture_hz=29.7~30.2 record_queue=0~1/32 dropped=0
[VIEWER PHASE] total_ms=약 29.7~31.6 ms
[DECOUPLED TIMING] physics_hz=약 199.8~200.2
```

따라서 현재 구현에서는 viewer와 head/양 wrist source pixel, LeRobot enqueue가 모두 실제 약
30 Hz이며 데이터 수집을 켜도 200 Hz authority가 영향을 받지 않았다. 순간적인 OS/GPU 스케줄링
jitter는 있을 수 있으므로 실전에서도 위 네 로그를 함께 확인한다.

## 10. 시간이 지나도 버퍼가 누적되는가

제어/상태 경로는 오래된 데이터가 쌓이지 않도록 구성되어 있다.

- PICO reader: 최신 완성 sample 하나만 교체
- IK request/response: ZMQ HWM=1 + latest-only
- Authority→viewer: publisher HWM=1, viewer가 대기 패킷을 모두 비우고 마지막 state만 적용
- Authority physics: camera, WebRTC, disk writer를 기다리지 않음
- LeRobot writer: 최대 32 frame bounded queue(30 Hz 기준 약 1.07초, 약 66 MB 상한)

따라서 처리량이 부족할 때 제어 명령이 무한히 쌓이며 지연이 점점 커지는 구조는 아니다.
대신 다음 중 하나로 나타난다.

- viewer/render FPS 저하
- 최신 state만 남기면서 중간 visual state 생략
- recorder queue가 가득 차면 가장 오래된 frame을 버리고 `dropped` 증가

아래 조건을 장시간 유지해야 정상이다.

```text
physics_hz=195~200
state_age_ms가 지속적으로 증가하지 않음
record_queue가 32에 붙어 있지 않음
dropped=0
PICO stale=False
NVST_R_BUSY 없음
```

이 구조는 **누적 지연을 방지**하지만 모든 frame 보존을 보장하지는 않는다. 저장 장치나
NVENC가 느리면 오래 기다리는 대신 frame을 drop한다.

## 11. 전체 종료

```bash
pkill -TERM -f '[i]saacsim-webrtc-streaming-client' 2>/dev/null || true
bash /home/user/Desktop/stop_decoupled_all_host.sh
```

PICO에서 Send를 끄고 XRoboToolkit 앱을 종료한다.

## 12. 현재 주의사항

- PICO 배터리가 충분하고 `Send`가 유지되는 상태에서 시작한다.
- preflight 성공 전 Authority를 시작하지 않는다.
- Authority와 viewer를 각각 한 번만 실행한다.
- WebRTC client는 viewer가 `49200`을 연 다음 실행한다.
- 잘못 calibration해 G1이 넘어졌다면 누운 상태에서 재calibration하지 말고 전체 재시작한다.
- 더 상세한 장애 판정은 `/home/user/Desktop/IrosIkea_decoupled_wbc_pico_final_TG.md`를 참고한다.
