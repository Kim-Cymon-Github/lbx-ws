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

- **최근 (2026-06-24): 실제 차량(creta) 흡수 + PBR 품질 대폭 개선 + 차량 리그/IBL 설계**.
  실제 차량 glTF(`2022_hyundai_creta.glb`, 56 prim·15 mat)로 전환하며 발견한 버그·품질을
  연쇄 해결. lbx-gfx 커밋 다수(아래는 lbx-gfx repo 해시). **남은 항목·우선순위는 이제
  `lbx-gfx/doc/pbr-status.md`(PBR) 와 `vehicle-rig.md`(차량 리그)** 로 분리·상세화.
  - **descriptor pool 고갈 수정**(`fea861a`) — VK pool(maxSets 20/sampler 16)이 56 prim
    텍스처에 고갈 → 모델+PBR 큐브 전체가 안 그려짐(Scene UBO set 할당까지 실패). 512 상향.
    (cgltf 파싱 오류처럼 보였으나 실제는 pool — 파일·cgltf 무죄 standalone 확인.)
  - **텍스처 dedup**(`fb671e4`) — (cgltf_texture,format) 캐시. primitive 마다 중복 로드를
    유니크 이미지(10)로. ORM(occlusion=metalRough 동일 이미지)도 자동 1개. 소유권 캐시
    이전 → 중복 해제 방지. (최적화 백로그 ① 일부 선반영.)
  - **투명(유리) 블렌딩**(`fb671e4`) — 로더 alphaMode 읽기, OPAQUE→BLEND 2패스, VK PBR
    파이프라인 src-over 전역. GLES depth test/write 분리(이전엔 test 를 write 에 묶어 투명이
    가림 무시). 단 굴절(transmission) 미구현 → 유리는 어두운 틴트.
  - **specular IBL cheap 버전**(`167a340`) — env 큐브 **mip 체인 GPU 생성**(VK blit /
    GLES glGenerateMipmap — 매 프레임 재생성 가능 = 실시간 카메라 큐브의 토대) + roughness
    LOD(blur) + **EnvBRDFApprox**(split-sum BRDF LUT 해석적 대체). 거친 타이어 가짜 광택·
    엣지 글로우 해소. = 로드맵 (a) specular IBL 의 cheap 선구현.
  - **diffuse IBL cheap 버전**(`70acb2a`) — 평평 ambient → env 흐린 mip(LOD4)을 N 으로
    샘플(방향별 환경색) + env 반사 desaturate(0.8). = (a) diffuse irradiance 의 cheap 선구현.
  - **clearcoat**(`4987d94`) — 차 도장 광택. base BRDF 위 유전체(F0=0.04) 반사층(직접광+env,
    base 를 (1-Fc) 감쇠). material 필드 + VK push/GLES uniform + ImGui 슬라이더 + 로더가
    "paint" 머티리얼에 자동 부여. (로드맵 (e) → pbr-status.md B2 "차량 핵심"으로 승격.)
  - **env PNG 로딩 + HDR→cube 변환**(`53c7f10` + `tool/equirect_to_cube.py`) — stb 를
    LBX_IMAGE 핸들러로 통합(svmdemo_main 패턴), env 큐브를 6면 PNG 로(없으면 .lbi 폴백).
    Poly Haven `german_town_street_1k.hdr` 를 numpy+cv2 스크립트로 equirect→cube 6면 PNG
    변환(sRGB 인코딩). 아직 **LDR** — 진짜 HDR float(RGBA16F) 은 후속.
  - **설계 문서 신설(lbx-gfx/doc/)**: `pbr-status.md`(구현 현황·미구현·우선순위),
    `vehicle-rig.md`(부품 분리·노드 extras 태깅·도어 3종 모션[힌지/슬라이드/경로 가이드라인]·
    바퀴 템플릿 인스턴싱·3D N-slice 치수 변형[화물칸 길이]·반투명 부위별 FBO 합성).
  - **함정 기록**: 셰이더 **BOM**(에디터 BOM 저장→glslc 실패; `.editorconfig` 셰이더 no-BOM
    override), **stale DLL 그림자**(exe-dir 복사본이 lib 신선 빌드 가림 → vcxproj PATH 운용,
    DLL 복사 금지). 둘 다 메모리/문서화.

- **최근 (2026-06-25): PBR 품질 두 축 해결 + 머티리얼 처리 데이터 주도화 + per-material
  에디터** (lbx-gfx 커밋 `d03c3e6`~`bb22e28`, 7개). 코어/셰이더는 트랙 A·B 이후 무변경 —
  나머지는 전부 테스트 앱(데이터 주도 설계의 보상).
  - **ui.cpp 분리**(`d03c3e6`) — 데모 ImGui 위젯을 main.cpp(양 백엔드 중복)에서 `test/src/
    ui.cpp` 한 곳으로. main.cpp 의 "Hello, world!" 창은 예제 원본 복원, `ui_draw()` 한 줄 호출.
  - **트랙 A — specular occlusion + exposure + env-spec 노브**(`dc75b8f`). 타이어 코로나
    (실루엣 스침각 env 반사)를 horizon(반사 R 이 기하면 아래로 향하면 페이드) × spec AO
    (Lagarde)로 억제. roughness 올리면 사라지는 걸로 BRDF 버그 아닌 specular IBL 현상 확정.
    exposure(ACES 전 곱)·env_spec 전역 노브. `gfx_set_render_tuning`, Scene UBO `.w` 패킹.
  - **트랙 B — SH-9 diffuse IBL**(`2fe3497`). cheap diffuse(env 흐린 mip 단일 샘플)의 회전
    시 ambient 깜박임(저해상도 방향성 노이즈) 해소. env 큐브를 SH-9 계수로 투영(앱 측,
    Ramamoorthi/Green) → 셰이더가 매끈한 반구 적분 irradiance 평가. `gfx_set_irradiance_sh`,
    VK Scene UBO `vec4 sh[9]` 확장. **= 정식 diffuse IBL(pbr-status.md A 절반) 완료**, 동적
    카메라 큐브로도 직결(SH 재투영이 쌈).
  - **clearcoat 데이터 주도화**(`862789f`) — 이름 휴리스틱(`"paint"` 부분일치→1.0; 그릴까지
    먹어 요철 죽던 버그) 폐기, `KHR_materials_clearcoat` 읽기(없으면 0=glTF 기본). 엔진이
    콘텐츠를 추측하지 않는다(vehicle-rig.md "이름에 파라미터 박지 않는다"와 정합).
  - **재색칠/머티리얼 편집** — 처음엔 이름 타게팅 override(`0e4451c`: base_color 교체 +
    틴트/단색)였다가, **per-material 에디터로 일반화**(`bb22e28`): 머티리얼 선택 → base_color/
    metallic/roughness/clearcoat/normal/emissive/albedo 직접 편집(override 토글 폐기, draw 가
    편집 레코드 직접 사용). recolor·clearcoat·metal/rough override 가 에디터로 수렴(코드 순삭).
    조명/노출/카메라는 씬 전역 유지. **코어 0 변경**(GFX_MATERIAL_PBR 이 이미 전 필드 보유).
  - 부수: auto-rotate 토글(정지 관찰), 마우스 오비트 카메라는 별도 작업(`task` 큐).
  - 함정: 차량 3종(creta/santa_fe/prius) paint 머티리얼은 baseColorTexture 없음(색=factor)이라
    재색칠은 단색 교체=틴트 동일. 셰이더 toolchain 가용(glslc=VulkanSDK, python=miniforge ltk).

- **다음 — 차량 애니메이션(vehicle-rig.md)**: 부품 분리·변환 토대는 섰고(per-material 편집·
  서브메시 리스트), 이제 리그/모션. 바퀴 회전·조향·리프트, 도어 3종 모션은 설계대로 가능 예상.
  - **미해결 — 캐터필러(무한궤도) 표현**: 타겟 차량에 **굴착기(excavator)** 가 있어 트랙이 필요.
    바퀴 인스턴싱(강체 복제)과 결이 다름 — 궤도는 연속 벨트(스프로킷·롤러 위를 도는 링크 체인)라
    별도 표현 방식 필요(UV 스크롤 / 링크 인스턴싱 경로구동 / 셰이더 변형 등 후보). 설계 미정.

- **다음 (완성도 PBR 로드맵)**: 차량 흡수엔 현 수준 충분(thin layer 목표 달성).
  "진짜 완성 PBR"로 가면 IBL 만으로는 부족하고 아래가 함께 필요:
  > **상세 미구현·우선순위는 `lbx-gfx/doc/pbr-status.md` 로 이관(2026-06-24)** — 아래는
  > 큰 그림. cheap IBL specular/diffuse + clearcoat 는 06-24 선구현, 정식은 pbr-status.md A/B2.
  - (a) **4단계 IBL** — 환경광 정식화. **diffuse 절반 = SH-9 정식 완료(06-25, 트랙 B)**.
    cheap specular(mip blur+EnvBRDFApprox) 06-24 선구현; **정식 specular prefilter/BRDF LUT 는
    미구현**(남은 절반). diffuse 는 cheap mip→SH-9 로 교체 완료.
    **내일 바로 시작 가이드**:
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
    - **빈 영역(위/아래/측면 부족) 채우기 — 아이디어 단계(확정 설계 아님)**: 정적
      스카이로 덮기는 부족(차는 실내 주차장·터널·야간·흐린 날도 다님 — "상상으로
      채우기"가 필요). 후보:
      - (A) **전방 카메라 상단의 먼 하늘/천장을 큐브맵 위 면에 투영** — 환경 적응적
        (실외=하늘, 실내=천장 자동 반영). 가볍고 실시간. 위 면은 대체로 균일 +
        prefilter 흐림이라 전방 한 방향 샘플로도 근사가 통할 가능성. [실용 1순위 후보]
      - (B) **가벼운 생성형 AI inpainting 으로 빈 영역 생성** — "상상 채우기"의 정답에
        가깝지만 실시간·임베디드 비용 큼. prefilter 로 어차피 흐려지는 영역이라 ROI
        검토 필요. plan 의 Vulkan compute/NN(YOLO) 인프라와 연관. [미래/고사양 옵션]
      - (참고) **시간적 누적**(차 이동 중 이전 프레임에 본 위/측면을 큐브맵에 누적)으로
        커버리지 보강 가능 — A 와 결합 시 정지 시점에도 직전 주행의 천장/주변 활용.
  - (b) **HDR 환경맵** — LDR `.lbi` → HDR(태양 등 1.0 초과 광)로 강렬한 반사.
  - (c) **그림자**(directional shadow map) — 입체감·접지감.
  - (d) **색 파이프라인 정합** — 현재 PBR 만 ACES/감마. Phong/blit/unlit 포함 전체
    linear→sRGB 일관(sRGB swapchain 또는 공용 톤매핑 패스).
  - (e) 후순위: MSAA/TAA(specular aliasing), sheen 등 glTF 확장.
    lbsvm 도메인 필요를 넘는 건 과투자(plan §0). **clearcoat 는 차 도장 핵심이라 06-24
    구현 완료**(pbr-status.md B2 로 승격).
  - **최적화 백로그(언젠가 — 현재 미적용)**:
    - ① **ORM 텍스처 패킹** — 지금은 occlusion(R) / metal_rough(G,B)를 **별도 텍스처·
      별도 descriptor set** 으로 로드한다. glTF 가 같은 이미지를 가리키면(ORM 관례)
      **한 장(R=AO, G=roughness, B=metallic) + 1 샘플 + set 1개**로 합칠 수 있다.
      텍스처/메모리 절약 + 임베디드 `maxBoundDescriptorSets`(4) 완화.
    - ② **material 텍스처 set 묶기** — 현재 VK 는 albedo/normal/metal_rough/AO/emissive
      를 set 1개씩(총 7-set). 한 set 의 여러 binding 으로 묶어야 임베디드 4 한계 충족.
    - ①②는 **차량 이주 / 임베디드 타겟 적용 시 함께** 정리(PC 검증 단계에선 현 구조로 동작).
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

- **최근 (2026-06-26): 동적그림자 외곽 블렌딩을 바깥 스커트→안쪽 페이드로 전환 (커밋 `bf60cb5`)**.
  - tptopview 역포팅 선결로 동적그림자(RADIAL_LOD)를 손보던 중, 외곽 블렌딩이 차량
    footprint 바깥에 스커트 링(RADIAL=`append_staggered_band` C2, GRID/CDT=
    `build_skirt_mesh` Clipper offset)을 생성하는 방식이라 번거롭고, Clipper offset 은
    코너 정점 몰림 아티팩트가 있었다.
  - 변경: **per-vertex edge distance**(각 정점→boundary loop 최단거리)를 정점 attribute 로
    싣고 셰이더가 `smoothstep(0, uFadeWidth, edge_dist)`로 **안쪽 페이드**. 스커트 생성
    전면 제거. 메시 boundary 를 블라인드스팟+margin 으로 키워(GRID/CDT 는 `skirt_offset_mm`
    재활용) 최외곽이 색경계가 되게 한다. 세 mesh_type(RADIAL_LOD/GRID/CDT) **동일 방식 통일**.
  - 셰이더 `drawing_vert/frag` 에 `aEdgeDist`/`uFadeWidth`(>0일 때만 — 공유 솔리드 셰이더
    보호) + `.enc` 재생성. `RenderShadow` 가 `uFadeWidth`(`shadow_fade_width`, 기본 100mm) 바인딩.
  - UI: **두 축 분리·용어 통일** — `Margin`(블라인드스팟→색경계, 메시 범위) / `Blend Width`
    (페이드 구간). Blend Width 는 셰이더 유니폼이라 실시간 조정(재빌드 불필요). 무력화된
    C1→C2 슬라이더 제거.
  - **WHY 안쪽 페이드**: 임의 외곽(비대칭/타원 R)엔 중심거리 fade 불가 → 외곽선까지 실측
    거리를 정점에 굽는 게 정답(barycentric 은 삼각형 크기 의존이라 부적합). 색 샘플을
    블라인드스팟+margin 으로 잡으면 바깥 스커트 geometry 가 불필요해진다(안쪽 페이드).
  - 설계문서(`doc/Transparent_Topview_Composite_Design.md`) §0·§3 갱신 — Step 2 가 가정하던
    '스커트(C2) 알파'를 'edge_dist 페이드 알파'로.

- **최근 (2026-06-25): CAN 입력 흡수 + transparent topview 투영면 이중화 (CAN/tptopview 세션 — lbx-gfx 그래픽 세션과 별개)**.

  **A) MCU UART → CCAN 입력 브리지** (`test/src/mcu/`: navitech.* + svmdemo_mcucan.*).
  - 2100R `Tifboard`(navitech) 흡수. R5 프레이밍/`0x38` 파싱은 그대로 두고, cmd `0x38`
    콜백에서 `swap_endian32`로 CAN ID 뽑아 `LBX_MSG('CCAN', LBX_CAN_PACKET)` Enq →
    기존 VCP(TCP)·SL-CAN(UDP) 과 **동일 버스(`LBX_CQ`)·동일 소비(`ProcessMsgs`)**.
  - **WHY**: "드라이버를 어떻게 만들든 이만큼 작고 안전하게(앱은 출처 모름, 드라이버는
    버스 모름) CCAN 버스에 붙는다"는 연결 패턴 시연. 입력=msg 단일 체계. 발단은
    tptopview 에 odometry(휠펄스→이동거리, 조향→회전중심)를 먹이려던 것.
  - 시리얼은 **lbx-intf 의 크로스플랫폼 comX**(fredslab, MIT)로 통일 — termios
    `UartDevice`·Teunis `RS232_*`(GPL) 폐기. **WHY**: lbx 가 win/linux 공통 시리얼을
    이미 제공하므로 전용/플랫폼 코드가 불필요(이득 0), GPL 오염 회피. 프레이밍이 가치지
    포트 여닫는 코드가 가치가 아님.
  - 구 RCM 가변벡터 API → 신형 SVEC 로 직접 개명(`svec_length`/`SVEC_DROP`/`SVEC_APPEND`),
    compat 셔임 제거(친절 과잉이라 판단). 매핑 정본 = `I:\eyeL-2100R\lbx\lbx_compat.h`.
  - **lbx-intf 변경**: `rs232_exports.def` 로 comX 11개 export(소스 무수정, 기존
    dllexport 와 병합), 4 config `<Link>` 에 연결. Linux 는 기본 노출이라 무관. `rs232`
    를 lbsvm-core `lib/rs232` **서브모듈**로 추가(헤더용; `.c` 는 미컴파일, 심볼은
    lbx-intf 에서 import).
  - 검증: test-lbsvm-core x64 Debug 링크 통과. comX export 는 dumpbin 으로 확인.

  **B) transparent topview(tptopview) 투영면 이중화** (`src/lbsvm_projection.cpp`).
  - 배경: tptopview 는 lbsvm-core 엔 있고 2100R 엔 없다(아래 eyeL-2100R 항목 = 2100R 로
    역포팅 계획). 그 선결로, 현 lbsvm-core tptopview 가 **일반 topview 를 깨뜨린 채
    방치**돼 있던 것을 먼저 고침.
  - 문제: `BuildBowlProjSurface`/`BuildTopviewProjSurface` 가 통합 `BuildFullProjSurface`
    로 위임되며, tptopview 용 블렌딩 둘(①바닥 뿌리 투명 `:351`, ②사이드오버랩 중간밴드
    투명 `:297`)이 **bowl 에도 무조건 적용** → 위에서 볼 때 **H자 빈틈**. (2100R 원본
    bowl 엔 둘 다 없고 전부 불투명.)
  - 해결: ①②를 `if(flat)` 로 가드 → bowl(일반 topview)은 전 정점 불투명 = 2100R 동작
    복원. flat(tptopview FBO 전용)만 블렌딩 유지. **= 투영면 이중화**.
  - flat 면: 클립 round-rect **균일 100mm 옵셋확대**(`TP_SIDE_OFFSET`) + 프로파일
    **스커트 200mm**(`TP_BLEND_SKIRT`, 안쪽 v0 α0 → 바깥 α1 램프). **WHY 균일확대**:
    좌우만 넓히면 앞↔옆 접합부(코너) 가 어긋남 → 균일 옵셋이라야 접합 유지. 첫 100mm
    미생성 = 차량근접부를 띄워 조향 시 바퀴 튀어나옴 회피. **WHY 이중화**: bowl 은
    불투명·완전커버, tptopview FBO 는 평면·블렌딩·차체밑 누적 — 요구가 정반대라 한 면으로
    불가.
  - 빌드 통과. **시각/실차 검증 미완**(H자 해소·flat 품질은 화면이라야 보임, 헤드리스 불가).
  - 다음(Step 2): 차에서 직접 튜닝하는 UI(`svmdemo_ui.cpp` 기존 슬라이더 인프라 위에
    `TP_SIDE_OFFSET`/`TP_BLEND_SKIRT`/`mm_per_pulse`/전륜 조향각 범위 추가). **WHY**:
    record/replay 가 없어 실차에서 튜닝해야 하므로.

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

- **transparent topview 역포팅 검토 (2026-06-25)**: lbsvm-core 에 CAN 을 붙이려던 목적이
  tptopview 인데, 정작 제품(현역 2100R)엔 그 기능이 없다. **lbsvm-core → 2100R 포팅
  가능성 분석 결과 = 가능**(난이도 낮음~중간). 두 레포가 **같은 lbx 조상**이라 토대가
  2100R 에 이미 다 있음 — `TGLFrameBufferObject`/`TGLProgram`(lbx_gles2.h), `TLBView` +
  `TLBShape::LocalCoord()`(lbsvm_classes.cpp:983), odometry(휠펄스+조향→회전반경 `CalcR`
  lbsvm_model.cpp:24). 신규는 `TLBTransparentTopview` 클래스(~200줄, FBO 2개 핑퐁 +
  odometry 모션 시프트로 차체밑을 과거 프레임으로 채우는 see-through) + 셰이더 2개
  (`general_vert`+`extimg_frag`, 표준 텍스처 패스스루)뿐. 나머지는 신↔구 이름 정합 +
  소소한 어댑터(2100R FBO 는 단일샘플 → `multisamples=1`; float 텍스처 우려는 기우 —
  누적이 아니라 텍스처 시프트 재드로라 RGBA8 충분).
  - **WHY 역방향**: 제품에 기능을 빨리 태우는 데는 2100R 이 지름길 — odometry/CAN 을
    이미 네이티브로 가져서 CAN 흡수가 불필요. lbsvm-core 의 CAN 작업은 PC/신규 타깃에서
    실차 데이터로 돌리는 독립 가치로 유지(두 트랙 목적이 다름).
  - **선결**: lbsvm-core tptopview 가 일반 topview 를 깨뜨려 둔 상태라, 투영면 이중화부터
    수리(위 lbsvm-core 2026-06-25 B 항목). 그게 안정되면 2100R 로 클래스+셰이더 이식.

## ltk — 빌드/운영 도구 (진행)

- `likit` 의 모놀리식 `lit.py` 를 기능별 독립 도구(`lit-build`/`lit-deploy`/
  `lit-version`/`lit-update`/`lit-commit` ...)로 분해해 재구현 중. `likit` 는
  레거시이며 함수 추출의 출처로만 참고한다.
