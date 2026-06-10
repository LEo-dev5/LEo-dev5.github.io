---
title: "Nav2 Jazzy에서 로봇이 안 움직일 때: Twist vs TwistStamped"
date: 2026-06-10
categories: [Robotics, ROS2]
tags: [ROS2, Jazzy, Nav2, Gazebo, TurtleBot3, 디버깅]
---

## TL;DR

ROS2 Jazzy + Nav2 + TurtleBot3 Gazebo 시뮬레이션에서 Nav2 Goal을 찍어도 로봇이 1mm도 안 움직였다.
원인은 **Nav2는 `Twist` 타입으로 속도 명령을 발행하는데, Gazebo 브리지는 `TwistStamped` 타입만 구독**하고 있었기 때문.
ROS2에서는 토픽 이름이 같아도 메시지 타입이 다르면 통신이 되지 않는다.
해결은 `nav2_params.yaml`에서 cmd_vel을 다루는 모든 노드에 `enable_stamped_cmd_vel: true`를 추가하는 것.

## 환경

- Ubuntu 24.04 (네이티브 설치)
- ROS2 Jazzy
- Nav2 (`nav2_bringup`)
- Gazebo Sim + TurtleBot3 (burger)

실행 명령:

```bash
# 터미널 1: Gazebo
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py

# 터미널 2: Nav2
ros2 launch nav2_bringup bringup_launch.py use_sim_time:=True map:=$HOME/my_map.yaml

# 터미널 3: RViz
rviz2 -d /opt/ros/jazzy/share/nav2_bringup/rviz/nav2_default_view.rviz
```

## 증상

SLAM으로 지도를 만들고, 2D Pose Estimate로 초기 위치까지 잡은 상태에서 Nav2 Goal을 찍었는데:

- RViz의 Nav2 패널: `Feedback: aborted`, Recoveries가 21회, 31회까지 치솟음
- Gazebo의 로봇은 제자리에서 미동도 없음
- 터미널 로그에는 같은 패턴이 무한히 반복:

```
[controller_server]: Passing new path to controller.
[controller_server]: Failed to make progress
[controller_server]: [follow_path] [ActionServer] Aborting handle.
[behavior_server]: Running backup
[behavior_server]: Collision Ahead - Exiting DriveOnHeading
[behavior_server]: backup failed
```

![증상: RViz에서 Navigation inactive, Feedback unknown 상태](/assets/img/posts/nav2-jazzy-symptom.png)

Nav2는 분명히 일을 하고 있다. 경로도 만들고, 컨트롤러에 전달도 하고, 실패하니까 후진 복구까지 시도한다.
그런데 로봇 몸체는 안 움직인다. 즉, **"두뇌는 멀쩡한데 명령이 바퀴까지 안 닿는" 상황**이다.

## 디버깅 과정

### 1차 시도: 위치추정 의심 → 아님

처음에는 AMCL 위치추정이 틀어진 줄 알았다. 그런데 확인해 보니:

```bash
ros2 topic echo /amcl_pose --once   # 위치 정상
ros2 run tf2_ros tf2_echo map base_footprint   # TF 체인 정상
ros2 topic hz /scan   # 라이다 5Hz 정상
```

모두 정상이었다. 라이다 점들도 RViz 지도의 벽에 정확히 정합되어 있었다. 위치추정은 범인이 아니다.

### 2차: 격리 테스트 — teleop으로 두뇌 건너뛰기

전체 파이프라인을 둘로 쪼개서 어느 쪽이 죽었는지 확인했다.

```
[Nav2 두뇌] → /cmd_vel → [ros_gz_bridge] → [Gazebo 바퀴]
```

Nav2를 건너뛰고 키보드로 직접 명령을 보내봤다:

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

**로봇이 움직였다.** 바퀴 쪽 배선(bridge → Gazebo)은 멀쩡하다는 뜻.
그럼 문제는 Nav2가 내보내는 명령이 bridge에 닿지 않는 구간에 있다.

### 3차: 결정타 — `/cmd_vel` 배선 검사

사실 디버깅 초반에 이상한 출력이 하나 있었다:

```bash
$ ros2 topic echo /cmd_vel
Cannot echo topic '/cmd_vel', as it contains more than one type:
[geometry_msgs/msg/Twist, geometry_msgs/msg/TwistStamped]
```

**하나의 토픽에 두 개의 타입이 섞여 있다**는 경고. 처음엔 넘어갔는데, 이게 핵심 단서였다.

```bash
ros2 topic info /cmd_vel --verbose
```

결과를 정리하면:

| 노드 | 역할 | 타입 |
|---|---|---|
| `collision_monitor` | Nav2 명령의 최종 출구 (발행) | `Twist` |
| `docking_server` | 발행 | `Twist` |
| `teleop_twist_keyboard` | 발행 | `TwistStamped` |
| `ros_gz_bridge` | Gazebo로 가는 유일한 다리 (구독) | **`TwistStamped`** |

이제 모든 게 설명됐다:

- teleop은 `TwistStamped`로 말함 → bridge도 `TwistStamped`를 들음 → **통한다**
- Nav2(collision_monitor)는 `Twist`로 말함 → bridge는 `TwistStamped`만 들음 → **bridge 입장에서 Nav2는 존재하지 않는 것과 같다**

ROS2의 DDS 통신에서는 토픽 이름이 같아도 메시지 타입(정확히는 타입 해시)이 다르면 매칭되지 않는다.
그래서 Nav2가 아무리 속도 명령을 뿜어도 Gazebo까지 한 발짝도 못 갔고,
progress checker는 "명령을 내렸는데 로봇이 안 움직이네" → `Failed to make progress` → abort를 반복한 것이다.

## 원인: 버전 과도기에 끼인 것

이건 버그가 아니라 ROS2 생태계의 마이그레이션 경계에서 생긴 호환성 문제다.

- Nav2는 `enable_stamped_cmd_vel` 파라미터로 `Twist` / `TwistStamped`를 선택할 수 있는데, **Jazzy까지는 기본값이 `false`(= Twist)**, Kilted부터 `TwistStamped`가 기본이다
- 반면 TurtleBot3 시뮬레이션의 Gazebo 브리지는 이미 새 방식인 `TwistStamped`로 마이그레이션됨

즉 Nav2는 아직 "옛 방식"으로 말하고, 시뮬레이터는 이미 "새 방식"만 듣는 상태로 만난 것.

참고: [Nav2 Jazzy → Kilted 마이그레이션 가이드](https://docs.nav2.org/migration/Jazzy.html)

## 해결

방향은 두 가지가 있다.

- (A) Nav2를 `TwistStamped`로 올리기 ✅
- (B) bridge를 `Twist`로 내리기

생태계 전체가 `TwistStamped`로 가고 있으므로 (A)가 미래 호환 방향이다.

### 1. params 파일을 홈으로 복사

`/opt` 시스템 파일을 직접 수정하면 패키지 업데이트 시 덮어써지므로 복사본을 만든다.

```bash
cp /opt/ros/jazzy/share/nav2_bringup/params/nav2_params.yaml ~/my_nav2_params.yaml
```

### 2. `enable_stamped_cmd_vel: true` 추가

기본 params 파일에는 이 파라미터가 아예 없다 (= 전부 기본값 false로 동작 중).
cmd_vel을 다루는 **5개 노드** 각각의 `ros__parameters:` 아래에 추가한다:

```yaml
controller_server:
  ros__parameters:
    enable_stamped_cmd_vel: true   # 추가
    controller_frequency: 20.0
    # ...
```

대상 노드:

- `controller_server`
- `velocity_smoother`
- `collision_monitor` ← /cmd_vel의 최종 발행자
- `behavior_server` ← 후진/회전 복구 동작도 cmd_vel 사용
- `docking_server`

하나라도 빼먹으면 평소 주행은 되는데 복구 동작만 안 되는 식의 반쪽짜리 고장이 나므로 전부 챙길 것.
YAML은 들여쓰기가 문법이므로 스페이스 4칸을 정확히 지킨다.

### 3. 수정한 params로 Nav2 재실행

```bash
ros2 launch nav2_bringup bringup_launch.py \
  use_sim_time:=True \
  map:=$HOME/my_map.yaml \
  params_file:=$HOME/my_nav2_params.yaml
```

### 4. 검증

```bash
ros2 topic info /cmd_vel --verbose | grep "Topic type:"
```

발행자/구독자 타입이 전부 `TwistStamped`로 통일되었으면 성공.

## 결과

| | 수정 전 | 수정 후 |
|---|---|---|
| Feedback | aborted | **reached** |
| 소요 시간 | 190초 (실패) | **7초** |
| Recoveries | 21회 | **0회** |

Nav2 Goal을 찍자 경로가 그려지고, 로봇이 기둥 사이를 통과해 목표 지점에 도착했다.

![결과: Gazebo에서 TurtleBot3가 정상 동작하는 모습](/assets/img/posts/nav2-jazzy-result-gazebo.png)

![결과: RViz에서 Feedback reached, 7초, Recoveries 0 확인](/assets/img/posts/nav2-jazzy-result-rviz.png)

## 배운 것

1. **ROS2에서 토픽 이름이 같아도 타입이 다르면 남남이다.** `ros2 topic echo`가 "more than one type" 경고를 뱉으면 그게 핵심 단서다. 흘려보내지 말 것.
2. **격리 테스트의 힘.** "Nav2 → bridge → Gazebo" 파이프라인에서 teleop으로 두뇌를 건너뛰는 테스트 한 번으로 용의자 범위가 절반으로 줄었다.
3. **`ros2 topic info --verbose`는 배선 검사 도구다.** 누가 어떤 타입으로 말하고 누가 어떤 타입으로 듣는지 한눈에 보인다.
4. **에러 메시지는 증상이지 원인이 아니다.** `Failed to make progress`, `Collision Ahead`, `backup failed` 전부 "로봇이 안 움직인다"는 같은 사실의 다른 표현이었다. 로그를 따라가되, 로그가 가리키는 곳이 아니라 데이터가 흐르는 경로를 의심해야 했다.
5. **버전 마이그레이션 경계를 의식할 것.** Jazzy는 Twist→TwistStamped 전환의 과도기 버전이다. 같은 배포판 안에서도 패키지마다 마이그레이션 속도가 다를 수 있다.

같은 증상으로 헤매는 분에게 이 글이 닿기를.
