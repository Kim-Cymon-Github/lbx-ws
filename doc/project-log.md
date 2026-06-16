# 프로젝트 로그 (크로스 레포 현황)

레포별 진행 상황과 다음 할 일을 한곳에 모아 "다음에 뭘 할지"를 빠르게 파악하기 위한
문서다. 시점 스냅샷이므로 작업이 끝나거나 방향이 바뀔 때 갱신한다. 상세 설계는 각
모듈의 `doc/` 를 본다. 정적 구조는 [architecture.md](architecture.md).

## lbx-gfx — graphics thin layer (활발)

- **완료**: device/swapchain(GLES+VK), mesh + `gfx_draw_mesh`(unlit/blit),
  external image CPU import 텍스처, **depth buffer(GLES+VK)**, VK 테스트를 gfx 소유
  swapchain 으로 이전(ImGui 는 오버레이), GPU 선택 유틸(통합 GPU 우선),
  표준 stdio/stdlib 호출을 lbx-core 래퍼로 전환, GFX_MESH 정점/인덱스 입력을
  `gfx_mesh_set_vertices`/`gfx_mesh_set_indices` 로(CPU 포인터 멤버 폐기).

- **최근 (2026-06-16)**:
  - test_vk(`test/test_vk/main.cpp`)를 원본 ImGui Vulkan 예제 대비 정리 — 죽은 `#if 0`
    블록(실은 1세대 실험 코드) 전부 제거, 변경점만 주석으로 남김.
  - **GFX_MESH 설계 재검토**를 `lbx-gfx/doc/plan.md` §3.5 에 기록: (A) CPU 입력/GPU
    자원/기하 설명 분리, (B) AoS·SoA 를 Vulkan binding 모델로 통합, (D) 멀티머티리얼=
    서브메시 다중 draw(`mesh`=한 draw granularity 유지), (E) map 중심 입력 + BUFFER 풀
    결합(`alloc_vertices(pool)`). 동적 그림자 SoA 워크드 예제 포함.
  - 그중 **A 의 1차 실행**으로 GFX_MESH 에서 CPU 데이터 포인터 제거 → `set_vertices`/
    `set_indices`(동기 복사) 도입, GLES/VK 양쪽 구현·빌드·실행 검증 완료.
  - **내일 이어서**: plan.md §3.5 open 체크리스트에서 시작 — (B) binding 배열 struct,
    (간극 1·3) vec2 position·vec4 f32 color attr, (E) `alloc_binding`/`map_attr` 동적
    경로. 이게 갖춰지면 lbsvm-core 동적 그림자 이주로 연결. 배포용 `lib/lbx` 서브모듈의
    헤더 복사본은 `set_*` 미반영 상태 — publish 동기화 별도 필요.

- **다음 (조명 로드맵)**: 1차 목표는 lbsvm-core 의 **차량 모델 렌더링 흡수**이고, 그건
  풀 PBR 이 아니라 **Phong + 큐브맵 반사**다. PBR 로 가는 길의 앞부분을 미리 까는
  셈이라 버리는 작업이 없다. 단계로 끊는다:
  1. **공통 조명 토대** (Phong/PBR 공유) — 조명/카메라 setter 실제 구현
     (`gfx_set_directional_light` / `gfx_set_camera_position` 은 현재 빈 껍데기),
     VK uniform 버퍼 경로(push constant 80바이트 한계 초과분 수용),
     다중 텍스처 바인딩 + NULL 슬롯용 1x1 기본 텍스처.
  2. **Phong 머티리얼 + cube 텍스처/반사** = 차량 모델 흡수. material kind 에 PHONG
     추가, `GFX_TEXTURE_CUBE` + `samplerCube` + `reflect(I,N)`. cube 토대는 이후 IBL 이
     재사용한다.
  3. **PBR** (metallic-roughness, Cook-Torrance) — 1 의 조명 토대 재사용. pipeline
     캐시(alphaMode/doubleSided 변형)와 선형/sRGB 색 파이프라인이 함께 와야 한다.
  4. **(나중) IBL** (환경광) — 2 의 cube 토대 재사용. 분량이 크게 늘어 후순위.
  - 연계: plan.md Phase B(Mesh+PBR — 텍스처 로딩, pipeline 캐시, glTF/OBJ),
    Phase C(external texture + textured-blit, dma_buf GLES).

- **복잡도 메모**: PBR/Phong 셰이더 본체는 정형화돼 보통 수준이다. 진짜 일은 1단계
  인프라(uniform 버퍼·다중 텍스처·조명 setter)와 pipeline 캐시·색 파이프라인이다.
  IBL 만 분량이 크게 늘므로 후순위로 둔다. 1~2 단계만 해도 "조명 받는 + 반사되는
  차량 큐브/메시"가 나오므로 거기서 끊을 수 있다.

- **보류 / 숙제**: VK 멀티뷰포트와 swapchain 리사이즈(재생성), 진짜 프레임
  파이프라이닝(슬롯별 command buffer 로 직렬화 제거), `calloc` -> 전용 zeroing
  alloc 함수 도입, 동적 mesh 스트림(vec2 position, per-frame color stream) —
  동적 그림자 연동의 전제.

- 설계: `lbx-gfx/doc/plan.md`, `lbx-gfx/doc/ndc-convention.md`.

## lbsvm-core — SVM 본체 (활발, 주 작업 대상)

- **진행**: Shadow/Poly/Geo 리팩토링(이름 변경보다 의존 방향 정리·중복 제거 우선,
  mesh 경계 선정리). working 브랜치는 `feature/shadow-upgrade`.
- **다음**: 동적 그림자 RADIAL_LOD 메시를 `GFX_MESH` 로 이주(위 lbx-gfx 동적 스트림
  숙제와 맞물린다). 차량 모델 렌더링은 lbx-gfx 가 Phong+큐브맵을 갖추면 그쪽으로 흡수.
- 설계: `lbsvm-core/CLAUDE.md`, `lbsvm-core/doc/`.

## lbx-geo (진행)

- **진행**: `lbx_rect` 를 `lbx_proj`(투영) + `lbx_poly`(폴리곤 생성)로 파일 분리.
  `rrect_* -> rect_*` API 통합은 완료.

## drv/plat-glwl — Wayland 플랫폼 드라이버 (예정/진행)

- `glwin`(Win32) 정리 -> `glwl`(`glfb` clone 기반 Wayland) 순서. EGL 등 컨텍스트
  소유권을 `lbx-gfxgl` 로 옮기고 `lbx-gl` 은퇴를 향한다.

## cal/cal-flood — 캘리브레이션 플러그인 (진행)

- cal 을 런타임 DLL 모듈로 분리. 설계 source of truth:
  `cal/cal-flood/doc/Cal_Module_Plugin_Design.md`.

## eyeL-2100R — 구형 레포 포팅 (별건)

- RADIAL 그림자를 구형 레포(`I:\eyeL-2100R`)로 이식(앱 좌표 직접 사용, radii/per-side
  재매핑). Clipper2/UI 연동은 보류.

## ltk — 빌드/운영 도구 (진행)

- `likit` 의 모놀리식 `lit.py` 를 기능별 독립 도구(`lit-build`/`lit-deploy`/
  `lit-version`/`lit-update`/`lit-commit` ...)로 분해해 재구현 중. `likit` 는
  레거시이며 함수 추출의 출처로만 참고한다.
