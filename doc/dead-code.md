# 죽은 코드 정리 목록 (일괄 제거 후보)

var-move 인터페이스 리팩터링(2026-07-07) 중 지나가며 확인된 죽은/미사용 코드 모음이다.
전수조사가 아니라 작업 경로에서 눈에 띈 것만 기록한다. 발견 시 여기에 계속 누적한다.

**제거 전 주의**: `LBX_INTF_EXPORT`/`AVIO_FILE_EXPORT` 같은 export 심볼은 이 트리 밖
(out-of-tree) 소비자가 있을 수 있으니 확인 후 제거한다.

## A. 컴파일에서 제외된 파일 (lbx-intf/src/intf) — 통째 제거 후보

VS `.vcxproj` ClCompile 과 Linux Makefile `OBJ_NAMES` **둘 다에 없는** 레거시다.

- `lbx_classes.cpp` (+ `lbx_classes.h`) — 자기 헤더만 자가 include, 외부 컴파일 소비 없음.
  - 내부에 `#if 0` 블록(vplay 드라이버 로딩, 대략 702~727행)
  - `/*ww ... */` 주석 블록(대략 916~934행)
  - 745행 `drv->entry(drv, opt, reqInitialize)` — 존재하지 않는 구 enum `reqInitialize` 사용(= stale 증거)
  - ※var-move 때 미컴파일인데도 정합성 위해 손댐(VCapDriverCommand var_share, avio Open 4인자)
- `lbfw_utils.cpp` (+ `lbfw_utils.h`) — 미컴파일. `TAppInstance`/`process_app_args` 계열.
- `lbfw_ipc.cpp` (+ `lbfw_ipc.h`) — 미컴파일.

→ 이 3세트가 진짜 죽었는지: 다른 모듈이 헤더를 include 해서 링크하는지 최종 확인 후 제거.

## B. 헤더 내 죽은 타입/주석 블록 (lbx_intf_drv.h)

- `OPEN_ARGS` 구조체 — 실사용 0건(배포 사본 매치뿐). var-move 때 opt 필드만 move 로 바꿔둠 → 통째 제거 후보.
- 대형 주석 블록: `lbxHOST_API`(대략 122~130), `LBX_DEVICE_DRIVER`(fwd 143 + 주석 struct 136~150),
  CAN 프로토콜 주석(165~176).

## C. vplay 인터페이스 (lbfw_ifc_vplay.h) — 레거시/죽음

- `LBX_VPLAY_INTERFACE`, `init_vplay_file_interface` — 실호출 없음. 유일 참조가 A 의
  `lbx_classes.cpp` `#if 0` 안. var-move 때 1.0 범프 + `IsCompatible` 추가해뒀지만 실사용 없음.
  파일 통째 제거 후보.

## D. 미사용 함수/필드

- avio-file `get_vcap_file_interface` (`avio_file_main.cpp:1101` / `.h:28`) — opt 미사용 + 트리 내
  호출자 없음(등록은 `init_vcap_file_interface` 사용). `AVIO_FILE_EXPORT` 라 out-of-tree 확인 필요.
- avio-v4l2 `LBX_CAP_DEVICE_INSTANCE.opt` (`avio_v4l2.cpp:195`) — dead 필드(항상 NULL, 코드에 주석 有). 유지 중.
- lbx-intf `LBX_PLATFORM_DRIVER.opt` (`lbx_intf_plat.h`) — VAR_NULL 설정만 되고 읽는 곳 없음.
  var-move 때 타입만 var_t 로 바꿔둠.

## E. 주석 처리된 함수

- `lbx_intf_plat.c` 의 `InitRenderer`(대략 38~58행) — 통째 주석.
