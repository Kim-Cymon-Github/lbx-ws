# 프로젝트 로그 (크로스 레포 현황)

레포별 진행 상황과 다음 할 일을 한곳에 모아 "다음에 뭘 할지"를 빠르게 파악하기 위한
문서다. 시점 스냅샷이므로 작업이 끝나거나 방향이 바뀔 때 갱신한다. 상세 설계는 각
모듈의 `doc/` 를 본다. 정적 구조는 [architecture.md](architecture.md).

## lbx-gfx — graphics thin layer (활발)

- **완료**: device/swapchain(GLES+VK), mesh + `gfx_draw_mesh`(unlit/blit),
  external image CPU import 텍스처, **depth buffer(GLES+VK)**, VK 테스트를 gfx 소유
  swapchain 으로 이전(ImGui 는 오버레이), GPU 선택 유틸(통합 GPU 우선),
  표준 stdio/stdlib 호출을 lbx-core 래퍼로 전환, GFX_MESH 정점/인덱스 입력을
  `gfx_mesh_set_vertices`/`gfx_mesh_set_indices` 로(CPU 포인터 멤버 폐기),
  **GFX_MESH binding 배열 모델(AoS/SoA/SoAoS)** + per-attr `attr_format` 오버라이드,
  **조명 토대(Scene UBO[VK]/개별 uniform[GLES] + 방향광·ambient·카메라 setter,
  lit draw=N·L diffuse+ambient, Blinn-Phong specular), 큐브맵 환경 반사
  (`GFX_TEXTURE_CUBE`, `reflect`+metallic 혼합), albedo 텍스처 경로 양 백엔드 정합
  (기본 흰색/노멀/큐브 텍스처), tangent-space 노멀맵(TBN)** — 양 백엔드 (build 210).

- **최근 (2026-06-22)**: GFX_MESH 구조·백엔드 binding 모델 개편 완료(커밋 `3a78eb1`).
  plan.md §3.5 "결정 확정(2026-06-22)" 참조.
  - **결정 A** (CPU 데이터 포인터 제거 → `set_vertices`/`set_indices` 동기 복사): 완료.
  - **결정 B** (AoS·SoA 를 Vulkan binding 모델로 통합): `GFX_MESH.bindings[8]` +
    `binding_count` + per-attr `attr_binding[]` 도입·구현. AoS=binding 1개 degenerate
    (기존 코드 무손), SoA/SoAoS=N개. GLES/VK 양쪽 bind 경로 구현·빌드 검증.
  - **간극 1·3** (vec2 position / f32 color): per-attr `attr_format[]` 오버라이드
    (`F32_2`/`F32_4`)로 해결. 0(DEFAULT)=슬롯 기본 포맷이라 기존 메시 무손.
  - **결정 D** (멀티머티리얼=서브메시 다중 draw, `mesh`=한 draw granularity 유지): 확정.
  - 그 전(06-16): test_vk 죽은 `#if 0` 코드 제거, plan.md §3.5 설계 재검토 기록.

- **최근 (2026-06-23): 조명 로드맵 1단계 완료 (VK 커밋 `40e78e5` + GLES `b9fe629`)**.
  조명/카메라 setter 실구현 + lit(PBR) draw 경로(최소 조명: N·L diffuse + ambient).
  - 공개 API: `gfx_set_directional_light(dir,color,intensity)` + `gfx_set_ambient_light`.
  - VK: `_VK_SCENE_UBO`(192B std140, set 0) + PBR pipeline(set0=UBO/set1=albedo,
    push=model+factors) + 1x1 기본 흰색 텍스처. 직렬 submit 이라 UBO 단일 버퍼.
  - GLES: 개별 uniform 경로(UBO 대신, ES2/ES3 호환). builtin.gles 에 pbr 셰이더
    추가 + build_shaders.bat 으로 .enc 재생성.
  - 검증: 양 백엔드 빌드 통과 + test 안정 실행(크래시 0, VK 큐브 음영 시각 확인).
    CLI 헤드리스 실행 DLL 셋업은 메모리 [[lbx-gfx-test-run-env]].
  - 남은 것: 다중 텍스처 바인딩/조명 setter 의 GLES albedo 텍스처는 GLES 기본 텍스처
    인프라 도입 시. mesh 동적 경로(E의 `alloc_binding`/`map_attr`, VK multi-binding
    pipeline, DYNAMIC 링버퍼)는 동적 그림자 이주 직전(plan.md §3.5 `[~]`).

- **최근 (2026-06-23): 조명 2단계 완료 (specular `34e7bd2` + 큐브맵 `8c2f5f1`)**.
  Blinn-Phong specular + 큐브맵 환경 반사(`reflect(-V,N)`, metallic 으로 lit↔반사
  혼합) 양 백엔드. `GFX_TEXTURE_CUBE` 신설(VK 6-layer CUBE_COMPATIBLE / GLES
  CUBE_MAP), `GFX_MATERIAL_PBR.env_cube`. **PHONG 별도 kind 대신 PBR 셰이더 발전으로
  통합**(plan §3.3 "phong/matte/reflect/chrome → PBR 통합" 정합). roughness→shininess,
  metallic→반사강도 해석. 빌드+test 안정 실행 검증(시각 검증은 사용자 VS).
  **= lbsvm-core 차량 모델 흡수 분기점 도달.**

- **다음**: 두 갈래 —
  - (a) **lbsvm-core 차량 수직 슬라이스 이주**(분기점 도달, 전략 아래). 착수 전
    공개 헤더(device/texture/material.h) → lib/lbx deploy 사본 동기화 선행.
  - (b) **3단계 PBR** (metallic-roughness Cook-Torrance). **현 Phong 셰이더를 대체하지
    않고 별도 program 으로 공존**(2026-06-23 결정 — 저사양=Phong, 고사양=PBR). 현
    "PBR" kind 는 내용이 Blinn-Phong+큐브맵이라 **PHONG 으로 정명**하고 진짜 PBR 을
    신설. 머티리얼에 셰이딩 모델 선택(런타임) + 저사양은 PBR program 미컴파일 가능.
    plan §3.3 의 "PBR 단일 통합"을 이 공존 모델로 수정함.

- **lbsvm-core 이주 전략 (2026-06-23 결정, plan.md Phase D)**: "수평으로 조금씩"이
  아니라 **수직 슬라이스**로 이주. 분기 기준 = 경로별 API 안정성(PBR 완성 여부 아님).
  **1호 = 차량 렌더링, 시점 = 로드맵 2단계(조명+Phong+큐브맵) 완료** (풀 PBR/IBL
  안 기다림). 기존 lbx-gl 경로와 **병행 검증**으로 빅뱅 리스크 제거. 동적 그림자는
  별도 트랙·후순위. PBR/IBL은 흡수 후 데이터 주도 업그레이드(재작업 없음).

- **glTF 로딩 방침**: 로더는 **lbx-gfx 코어 밖(test app)** — `cgltf` single-header만
  `lib/`에 추가, glue가 `GFX_MESH`+`GFX_MATERIAL_PBR`로 변환. 텍스처는 기존 `lbx-stb`
  재사용(추가 의존 0). glTF는 PBR 검증용이고 차량은 PLY(`lbsvm_model_ply`)라 별개.

- **조명 로드맵 상세**: 1차 목표는 lbsvm-core 의 **차량 모델 렌더링 흡수**이고, 그건
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
