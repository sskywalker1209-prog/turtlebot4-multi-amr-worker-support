> **이 저장소에 대해** — Doosan Robotics ROKEY BOOT CAMP 9기 팀 프로젝트 원본 [Jik-Kim/mro-multi-amr](https://github.com/Jik-Kim/mro-multi-amr)의 포크입니다. 아래 원본 README 는 팀 전체 프로젝트 설명이고, 이 안내 절만 이시율이 추가했습니다.
>
> - **프로젝트**: TurtleBot4 기반 Multi-AMR 작업자 추종 · 정비 지원 시스템 (2026.08.31 ~ 09.11). 미니 단계 4인 조 → 본 단계 8인 통합팀(PM 있음)
> - **시연 영상**: [Google Drive](https://drive.google.com/file/d/11n-J_89sI_TaFnmyXjEvgcy3bLyjbCYH/view?usp=sharing)
> - **이시율 담당**
>   - 미니 단계: 4인 조 팀장, Vision(외부 웹캠 YOLO 검출 → TurtleBot4 탐색 · 접근)
>   - 본 단계: 통합팀 주행 파트 2인 공동 — 작업자 추종 개선(남은 거리의 절반 지점을 목표로 설정), 도크 복귀 확장, 안전 복구(속도 명령 중재), 구역 사용권 반납 결함 수정, 셔틀–추종 통합 시험, 실주행 시험, AMR 파트 발표
> - **팀 담당**: 저장소 골격 · 인터페이스 계약 · 리뷰 · 병합(PM), 셔틀 미션, 관제 · 대시보드, 인지(YOLO · ArUco), 웹캠 등록, Discovery Server 구성
> - **함께 해결**: 두 대 동시 운용 시 Nav2 기동 중단 · 센서 데이터 끊김 → 주행 기록을 함께 분석해 순차 기동, 필요한 노드만 통신 정보 수신, 시각화 · 중복 카메라 구독 축소
> - **결과**: 한 대는 전 구간, 다른 한 대는 카메라 고장으로 도크 복귀를 뺀 구간을 수행한 팀 시연

---

# mro-multi-amr — MRO Multi-AMR 정비 지원 시스템 (Phase 1 실증)

정비사(스케일 인형+RC카)가 관제센터에서 자재를 요청하면 TurtleBot4가 창고(A/B/C)에서 가져와 대기공간에서 만나고,
작업공간까지 따라다니며, 두 대가 사람 개입 없이 공존한다. 요구사항: **BRD v3.6** · 설계: **SDD v1.0** (3장 시나리오 S1~S10, E-1~E-7).

- **실행 명령의 정본은 [docs/demo_run_procedure.md](docs/demo_run_procedure.md) 다.** 아래 표는 어느 PC 가
  무엇을 띄우는지 보는 용도이고, **실제로 칠 명령·순서·게이트는 절차서를 따른다.**

## 기준 문서

- **[docs/demo_run_procedure.md](docs/demo_run_procedure.md) 실행 절차 (명령어 정본)** · [docs/setup.md](docs/setup.md) 환경 구축 (PC 마다 한 번)
- [docs/spec/MRO_Multi-AMR_BRD_v3.5.md](docs/spec/MRO_Multi-AMR_BRD_v3.5.md) 요구사항 (문서 안 버전 **v3.6**, 파일명은 링크 유지를 위해 v3.5) · [docs/spec/MRO_Multi-AMR_SDD_v1.0.md](docs/spec/MRO_Multi-AMR_SDD_v1.0.md) 설계
- [PROJECT.md](PROJECT.md) 개요·역할 · [AGENTS.md](AGENTS.md) 작업 규칙
- [docs/SOT.md](docs/SOT.md) 확정 결정 · [docs/architecture.md](docs/architecture.md) 데이터 흐름 · [docs/interfaces.md](docs/interfaces.md) 토픽·서비스 계약
- [docs/todo.md](docs/todo.md) 파일 단위 작업 목록 · [docs/issues.md](docs/issues.md) 개발 이슈 리스트

## 구조

```text
ros2_ws/src/
├── mro_interfaces/    # msg/srv 계약 (ament_cmake + rosidl)             [PM]
├── mro_perception/    # OAK-D 검출·거리 보정·재식별 게이트, ArUco          [인지]
├── mro_webcam/        # 웹캠1 등록, 웹캠2 힌트, 호모그래피, KPI 로거       [웹캠]
├── mro_navigation/    # 셔틀, 추종·재탐지·Standby, safety_watchdog, 복귀  [주행]
├── mro_fleet/         # DB, fleet_manager, 관제 UI (구역 세마포어는 주행 소관)  [관제]
└── mro_bringup/       # launch(PC 역할별), params(로봇별·아레나), maps    [PM]
tools/                 # Nav2 복구 스크립트, KPI 리포트
docs/
```

각 앱 패키지는 `nodes/`(rclpy) 와 `core/`(ROS 비의존) 로 나뉜다. 미니프로젝트(Kn1_mini)에서 이관한 파일은 파일 상단 `[이관]` 헤더에 변경점과 TODO 가 있다.

## 빌드

**클론만으로는 돌지 않는다.** 모델(`worker.pt` 19 MB)·맵·호모그래피·보정값은 저장소에 들어 있지만,
아래 넷은 PC 마다 한 번 준비해야 한다 (상세 [docs/setup.md](docs/setup.md) 「준비」).

| 밖에 있는 것 | 무엇 |
|---|---|
| `~/turtlebot4_ws` | `turtlebot4` · `turtlebot4_desktop` · `turtlebot4_navigation` 빌드 |
| `/etc/turtlebot4_discovery/setup.bash` | `ROS_DISCOVERY_SERVER=";192.168.108.41:11811"` (PC-F 단일 서버) |
| venv `rokey_venv` | `ultralytics` · `opencv-python` · `numpy` · `flask` |
| apt | `ros-jazzy-{tf2-geometry-msgs,message-filters,cv-bridge}` |

```bash
cd ros2_ws && colcon build --symlink-install && source install/setup.bash
```

## 실행 — **명령은 [docs/demo_run_procedure.md](docs/demo_run_procedure.md) 가 정본이다**

아래는 역할 배치만이다. **로봇 제어 PC 는 터미널 14개**를 순서대로 띄우고, 그 사이에
`map→odom` 하드 게이트와 2대 운용 규칙(`node` 모드·기동 순서·대기공간 간격)이 있다 —
전부 절차서에 있고, 이 표에는 없다.

| PC | 역할 | 띄우는 것 |
|---|---|---|
| PC-R3 / PC-R8 (로봇당 1대) | 로봇 제어 | localization · 셔틀 리그 · 셔틀/복귀 노드 · Nav2 · 추종 리그 — **절차서 T0~T14** |
| PC-W1 / PC-W2 | 웹캠 | `webcam1_pc.launch.py` / `webcam2_pc.launch.py` |
| PC-F | 관제 | 디스커버리 서버 → `fleet_pc.launch.py` → http://localhost:5000 |

모든 PC·로봇은 `ROS_DOMAIN_ID=3`, **PC-F `192.168.108.41` 의 단일 offboard Discovery Server** 경유
(9/5 확정·검증). 터미널 첫 줄은 `source tools/shuttle_env.sh` 가 대신한다.

## 상태

**Phase 1 실증 단계.** 9/10 실주행에서 단독 완주(관제 미션 → 언독 → 베이 → 탈출 회전 → 대기공간 →
추종 → 관제 RETURN → 도킹)를 확인했다. 남은 것은 2대 동시 운용이고, 진행 중 이슈는
[docs/issues.md](docs/issues.md), 파일 단위 작업은 [docs/todo.md](docs/todo.md) 에 있다.
