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
  (기본 흰색/노멀/큐브 텍스처), tangent-space 노멀맵(TBN),
  **Cook-Torrance PBR(metallic-roughness, D/F/G) — Phong 과 공존(shading_model
  런타임 선택)**, **glTF 로딩(cgltf): 메시 + 머티리얼 + 텍스처 풀(albedo/normal/
  metal_rough/AO/emissive) + 실제 큐브맵(.lbi) 환경 반사 + ACES 톤매핑·sRGB 감마 +
  env roughness 감쇠 근사** — 양 백엔드 (build 210).

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

- **최근 (2026-06-23): glTF PBR 로딩 완료 (커밋 `54270cc`/`b06a86f`/`d860a91`)**.
  cgltf single-header 로더(test app 전용, 코어 밖). DamagedHelmet 으로 PBR 골든 검증.
  - **메시**: cgltf accessor(pos/normal/tangent/uv/index) → `GFX_MESH` interleave.
    노드 `cgltf_node_transform_world` → world matrix flatten(서브메시 리스트).
  - **머티리얼**: pbr factor + albedo/normal/metal_rough/AO/emissive 텍스처
    (glb 임베드 PNG/JPEG 를 `stb_image` 로 디코드, albedo/emissive=SRGB, 나머지 UNORM).
    VK 는 텍스처 set4/5/6 추가(pipeline 7 set, 임베디드 묶기 TODO), GLES 는 unit3/4/5.
  - **환경 반사**: lbsvm `.lbi` 6면 큐브맵을 `new_file_stream`+`LBX_IMAGE_LoadFromStream`
    으로 로드 → gfx 큐브맵. 금속이 실제 환경 반사.
  - **색**: ACES filmic 톤매핑 + linear→sRGB 감마(파스텔톤 해소), env 반사
    roughness 감쇠 근사(거친 호스 가짜 광택 제거).
  - **버그픽스(VK)**: PBR 셰이더 mediump→highp — half 에서 D(GGX) a2 언더플로 NaN
    (하이라이트 검정). 임베디드도 PBR 은 highp 필요.
  - → **차량 모델 흡수에 충분한 PBR 수준 도달.**

- **다음 (완성도 PBR 로드맵)**: 차량 흡수엔 현 수준 충분(thin layer 목표 달성).
  "진짜 완성 PBR"로 가면 IBL 만으로는 부족하고 아래가 함께 필요:
  - (a) **4단계 IBL** — 환경광 정식화. 가장 큰 한 걸음. **내일 바로 시작 가이드**:
    1. **자원**: Poly Haven `.hdr`(equirect, 2:1 파노라마, CC0 무료) → `test/assets/`.
       stb 의 `stbi_loadf` 로 float 로드(stb 이미 보유). 1k~2k 면 충분.
    2. **전처리(굽기)** = 3가지 맵 생성:
       - equirect → 큐브맵 변환(또는 셰이더서 equirect 직접 샘플)
       - diffuse **irradiance map**(환경 반구 적분, 작은 큐브 ~32) — 환경광 확산
       - specular **prefilter mip**(roughness 별 블러, base ~128 + mip 체인) — 거친 반사
       - **BRDF LUT**(2D ~512, (NdotV,roughness)→scale/bias, split-sum 적분)
    3. **선결 인프라**(현재 없음):
       - **큐브맵 mip 지원** — 현 `gfx_texture_cube_from_pixels` 는 mip 1. prefilter
         체인엔 mip levels 필요.
       - **HDR 포맷 RGBA16F** — 현 텍스처는 RGBA8. HDR 환경/irradiance/prefilter 는 float.
       - **오프스크린 큐브맵 렌더(face별 render target) 또는 compute** — 굽기 경로.
         (VK compute 는 plan 의 YOLO 용 compute 인프라와 공유 가능.)
    4. **셰이더**: 현 `ambient`(flat) + env 근사(×(1-roughness))를
       `ambient = irradiance(N)·albedo·ao + prefilter(R,rough)·(F0·lut.x + lut.y)` 로 교체.
    - 참고 구현: Filament, learnopengl.com IBL(split-sum, Epic 2013).
  - (a') **동적 환경맵 = 실시간 IBL — 야심 목표(2026-06-23 사용자 제기)**: 정적 prebake
    대신 **런타임에 환경맵을 갱신**. lbsvm 도메인 핵심 = **차량 카메라 영상(dma_buf
    zero-copy, plan §3.4 external image)을 실시간 환경 큐브맵으로** → 차량 크롬/유리에
    실제 주변(카메라 피드)이 비치는 SVM 킬러 기능. (대안: 장면 6방향 렌더 reflection
    probe.) **도전** = 매 프레임/주기 큐브맵 캡처 + **실시간 prefilter 재계산**(비쌈 —
    compute + 저해상도 + 주기적 갱신으로 분산). 정적 IBL(a) 인프라(prefilter/LUT)를
    그대로 쓰되 입력을 카메라로, 갱신을 런타임으로. plan Vulkan compute 와 직접 연관.
  - (b) **HDR 환경맵** — LDR `.lbi` → HDR(태양 등 1.0 초과 광)로 강렬한 반사.
  - (c) **그림자**(directional shadow map) — 입체감·접지감.
  - (d) **색 파이프라인 정합** — 현재 PBR 만 ACES/감마. Phong/blit/unlit 포함 전체
    linear→sRGB 일관(sRGB swapchain 또는 공용 톤매핑 패스).
  - (e) 후순위: MSAA/TAA(specular aliasing), clearcoat/sheen 등 glTF 확장.
    lbsvm 도메인 필요를 넘는 건 과투자(plan §0).
  - **lbsvm-core 차량 수직 슬라이스 이주**(분기점 도달). 착수 전 공개 헤더
    (device/texture/material/program.h) → lib/lbx deploy 사본 동기화 선행.

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
