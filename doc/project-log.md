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

## lbx-core — 토대 라이브러리 (활발)

- **lbx::var 도입 + RTTI 2.0 직렬화 (2026-07-01)**: `var_t` 를 값-중심 move 유통의 DOM
  허브로 확립하고, 그 위에 nlohmann-json 스타일 C++ 편의층을 얹음. 커밋 `5a83eb6`(클래스)
  → `8f106a3`(C++11 가드) → `c542a6e`(RTTI 직렬화) → `595bbc0`·`2db0d5c`(파서 누수 픽스).
  - **개명**: C++ `Variant` → `lbx::var`(namespace `lbx`, `class var : public var_t`).
    레거시 `Var`/`VarRef` 는 폐기 대상으로 존치. 사용처(test/svmdemo_core/cal_module) 전부 이관.
  - **nlohmann 초기화**: `initializer_list` 생성자 + 스칼라 변환 생성자.
    `{ {"k",v}, ... }` = 모든 원소가 [문자열,값] 2-원소 배열이면 객체, 아니면 배열.
  - **C++11 가드**: `lbx_type.h` 에 `LBX_HAS_CPP11`(`__cplusplus`/`_MSVC_LANG` 병행 —
    MSVC `/Zc:__cplusplus` 미지정 대비), `lbx::var` 전체를 `#if LBX_HAS_CPP11` 로 감쌈
    (pre-C++11 / TCC C 빌드 안전). 9개 크로스 타깃 컴파일 검증.
  - **문자열 헬퍼**: `var_get_strl`(SSTR/USTR/CSTR 통합 길이) 헤더 노출 + `StrLen()`/`IsString()`.
  - **RTTI 2.0 ↔ var (핵심 결정)**: **var 가 DOM 허브. RTTI 는 struct↔var 만.** JSON 은
    전적으로 var 가 담당(RTTI 는 JSON 을 직접 안 만짐). 이터레이터 직렬화안(`rtti_write_json`
    `#if 0`) 폐기 — SAX 는 짜기 번잡, DOM(=var)이 정답.
    - `var_from_struct`(struct→var 신규)/`var_to_struct`(var→struct, 2.0 스텁 채움).
      value-in/value-out move 규약. 스칼라 최타이트 인코딩, ustr 은 복사 대신 공유.
    - `LBX_REFLECT(T, 멤버…)` 매크로(멤버 이름만; `decltype` 로 타입 자동추론) +
      `rtti_of<T>()` 트레잇 + `to_var<T>()`/`var::get<T>()`. `ustr_t` 스칼라 RTTI 추가.
    - 결정 배경 상세: auto-memory `rtti-serialization-architecture`.
  - **JSON 파서 누수 픽스 (진짜 원인 규명)**: 잘못된 JSON(`{ "a": / }`) 파싱 시 21B 누수를
    WSL LSan 이 포착. **원인 = `read_string` 이 에러 경로(`lbl_error`)에서 자기 누적 버퍼
    `t` 를 안 푸는 것** — `SUPPORT_YAML` 이 켜져 lone `/` 를 YAML 중첩객체 시작으로 오해석
    → `read_string('}')` 호출 → `:`/종료 못찾고 lbl_error → t 누수. 픽스 = lbl_error 에
    `svec_drop(&t)` (+ `var_json_`/`var_read_json_str` 에러경로 부분트리 해제, 키 처리 move).
    - **얕은 refcount 모델·키 소유권과 무관**을 격리 재현 + 파서 내부 rc 프로브로 증명
      (키 rc 는 항상 `1→2→1→0` 균형).
  - **테스트**: `test/src/test_var.h`(init 파싱/StrLen/flow/JSON주석/reflection 왕복). 9타깃 빌드 +
    WSL ASAN 357/359 통과, 누수 0.
  - **미커밋(작업트리, 세션 이월)**: 진단용 `svec_refcount`(svec 읽기전용 rc 피크, `lbx_svec.{h,c}`)
    + `test_var_rc_probe`(키 소유권 rc 균형 검증). 유지·커밋 여부 미정.
  - **다음**:
    1. **얕은 컨테이너 RC 검증 스트레스 테스트** — 계획서 `lbx-core/doc/shallow-rc-validation-plan.md`.
       (`var_t` obj/array 는 최상위 버퍼 rc 만 보는 1단계 모델이라 완전검증 미완. **꼭 해야 함.**)
    2. `LBX_REFLECT` 배열/핸들러·`char[N]`/`const char*` 문자열 멤버 확장(코어는 배열 지원).
    3. RTTI 1.0(`LBX_REFL_*`) 사용 6파일 2.0 이주 → 1.0 코드 전부 제거.

## lbx-intf — 모듈 인터페이스 계약 (활발)

- **var 전달 borrow→move 전면 전환 (2026-07-07)**: 모듈 인터페이스에서 var 를
  `const var_t *`(borrow) 로 넘기던 것을 `var_t`(value=move) 로 전환. avio(1.0)에서
  실험한 패턴을 `.lit` 순서대로 전 모듈에 적용하고, 모듈마다 빌드→배포(push) 를
  인터리브해 하위가 새 헤더를 pull 받게 진행. 콜리가 소유권 인수 후 `var_drop`,
  콜러가 계속 쓰면 `var_share`, NULL 자리는 `VAR_NULL`. 상세·함정은
  auto-memory `var-move-interface-refactor`.
  - **범위**: 엔트리 체인(`LbxModuleInterfaceEntryFunc.arguments`,
    `LBX_MODULE_INTERFACE_Open/Close.options`)까지 move 통일 + 도메인 vtable
    (Open/Do/Run/opt/params 등) 전부. 저장 필드 중 `LBX_CAL_UI_TARGET.params`
    (호스트 소유 per-frame view)·container `Load`·레거시 팩토리는 borrow 유지.
  - **버전 범프**: major 0 → **1.0.0.0**(PLDR/UIDR/UIMD/CALM/APDR/PLAY). 이미
    major 1 인 **AVIO 만 1.0→1.1**. (`0x01000000` 은 minor 1 이라 APDR/PLAY 도 major 0 였음.)
  - **호환 가드 강화**: 각 인터페이스 `*_IsCompatible` 신설/강화 — 이제 major 뿐 아니라
    **minor 까지** 일치해야 호환. 각 모듈 entry 가 로드 시 호출해 불일치 거부.
  - **배포 완료**: lbx-intf `0.2.59 b670` → lbx-gui `0.1.29 b540` → avio-file `0.1.48 b231`
    → avio-v4l2 `0.1.3 b7` → plat-glfb `0.1.6 b11` → plat-glwl `0.1.0 b7`
    → plat-glwin `0.1.31 b127` → cal-flood `0.1.0 b7` → lbsvm-core `0.1.5 b2328`
    → eyel2sdk `0.1.0 b9`(release gate green). lbx-core/cal/geo/gfx/gl 은 인터페이스
    미사용이라 변경·빌드 없음.
  - **함정(호출부 이관 시)**: ① `NULL` 은 `var_t` 로 변환 안 됨 → 전부 `VAR_NULL`.
    ② `Var`/`lbx::var` 는 `var_t` 를 public 상속(IS-A)이라 값 자리에 넘기면 refcount 없이
    슬라이스 → 콜리 drop + 소멸자 drop = 이중해제 → `var_share` 필수(CreateContext/Run 다수).
    ③ 직접 `intf.entry(&intf, mrQuery, NULL)` 호출도 `VAR_NULL`.
    ④ CAL `Run` 의 `panel` 은 `params` 내부 포인터라 params 는 사용 완료 후 drop.
  - **다음**: 작업 중 발견한 죽은 코드 일괄 제거 — [dead-code.md](dead-code.md).

- **인터페이스 0.3 재설계 — GFX 서비스 번들 + cal 세션 SSOT (2026-07-08)**: 첫 재릴리스 전
  클린 브레이크(구 릴리스 소비처 동결 확인). 상세는 auto-memory `intf-03-gfx-services-redesign`.
  - **`LBX_GFX_SERVICES` 신설**(`lbx_intf_gfx.h`, 구 `lbx_intf_ext_image.h` 흡수·삭제):
    flat 번들 — ctx + ImportImage/UpdateImage/DestroyImage + **SetTexFilter**. `LBX_HOST_API.gfx`
    (0.3)로 노출. 생산자 = lbx-gfx `gfx_host_services()`(태그 u64) / lbx-gl `lbx_gl_host_services()`
    (부호 규약, eyel2sdk 예제용). 중첩 구조체 대신 풀어 합침(독립 인스턴스/버전주기/다중 부모
    아님). "TextureFilterCallback u64/gfx 계약화" 백로그가 이 설계로 해소.
  - **cal 계약 0.3**: `LBX_CAL_TARGET`{cams, cam_count, params}로 Run/RenderUI 인자 통합.
    params 는 move 에서 **세션 문서 차용(`var_t*`)** 으로 환원 — 전 호출자가 var_share 의식만
    치르던 실사용 증거. **SHARED-STATE RULES** 명문화: 문서=SSOT(양쪽이 읽고 씀), 변경 감지는
    포인터가 아닌 내용, 카메라 pos/lens 변경 시 모듈이 파생캐시 자가 갱신.
  - **버전 규율 통일**: 전 인터페이스 major+minor 일치(HOST_API IsCompatible 도 강화, cal entry
    의 정확일치 검사 제거). append 관용 폐기 — 락스텝 재배포가 규율.
  - **mrQuery "params" 명세**: 모듈이 필수/옵션 파라미터 스키마(type/unit/required/default/
    choices/desc)를 자기서술 — 헤드리스 호스트가 Run 전에 인지. 결과 doc "missing" 계약.
  - **배포(락스텝)**: lbx-intf `b673` → lbx-gfx `0.1.2 b220` → lbx-gl `0.1.70 b421` → lbx-gui
    `0.1.29 b543`(뷰어 콜백 user_data) → avio-file `b233`/avio-v4l2 `b9` → cal-flood `b16`
    → lbsvm-core `b2340` → eyel2sdk `b14`(gate green).

## lbsvm-core — SVM 본체 (활발, 주 작업 대상)

- **최근 (2026-07-10): RK3576 투영면 전멸 미스터리 해결 — 스테일 셰이더 바이너리 캐시 (WDLABD2411-578).**
  BSP 교체(libmali wayland-gbm → vulkan-wayland-gbm) 후 svmdemo 의 3D/탑뷰 투영면만
  전멸 + 차를 관통하는 랜덤 검정 삼각형. 차모델/PNG/ImGui/SingleView/TP 는 전부 정상.
  - **원인**: 투영면 cam 프로그램만 유일하게 `cam_prog.cache`(glProgramBinary)에서 로드하는데,
    프로그램 바이너리는 드라이버 빌드 종속이라 BSP 교체로 무효. 비호환 바이너리는 **GL 에러
    없이 GL_LINK_STATUS=FALSE 로만** 나타나는데(스펙 의도) lbx-gl `LoadBinary` 가 glGetError
    만 봐서 "성공" 통과 → 링크 안 된 껍데기 프로그램이 전 뷰 배정 → 릴리스 빌드라 GL_CHECK
    무력 → glUseProgram 조용히 실패 → **직전 바인딩된 남의 프로그램으로 투영면이 그려짐**
    (탑뷰=TP 블릿 uProjection=identity → 전부 클립, 3D=그림자 프로그램 → 퇴화 삼각형).
  - **진단 과정**(원격 자율 루프: adb + weston `--debug` 스크린샷 + 계측 빌드): depth/cull
    끄기 → 기각, 단색 강제 → 기각(래스터 자체가 안 됨), draw 직전 GPU 리드백(uProjection
    glGetUniformfv/GL_CURRENT_PROGRAM/attrib 상태)으로 "뷰 프로그램 h=9 인데 바인딩은 21/30"
    모순 포착, 행렬 수학은 보드 단위테스트로 무죄 증명 → PrepareRendering 의 캐시 로드 발견.
  - **픽스**: lbx-gl `f48feb8` — `TGLProgram::LoadBinary` + `stream_read_glprogram` 에
    GL_LINK_STATUS 검사(+에러 잔재 배수, 읽기 버퍼 누수 수리). 실패 시 0 반환 → svmdemo 가
    소스 리빌드 → SaveToFile 캐시 자동 갱신. 보드에서 스테일 캐시 복원 실검증(0.05s 검출 →
    리빌드 → 정상 렌더링). **lib/lbx 체인 배포는 미실시**(다음 lit-ship 때 포함).
  - **주의**: lbx-gui `IMGUI_LOAD_SHADER_CACHE` 빌드는 컴파일 폴백이 없어 스테일 캐시 =
    복구 불능(현재 Makefile 에선 꺼져 있음). 켜서 출하하는 제품은 드라이버 업데이트 시 캐시
    재생성 절차 필수. 상세는 auto-memory `gl-program-binary-cache-gotcha`.

- **진행**: Shadow/Poly/Geo 리팩토링(이름 변경보다 의존 방향 정리·중복 제거 우선,
  mesh 경계 선정리). working 브랜치는 `feature/shadow-upgrade`.
- **다음**: 동적 그림자 RADIAL_LOD 메시를 `GFX_MESH` 로 이주(위 lbx-gfx 동적 스트림
  숙제와 맞물린다). 차량 모델 렌더링은 lbx-gfx 가 Phong+큐브맵을 갖추면 그쪽으로 흡수.
- 설계: `lbsvm-core/CLAUDE.md`, `lbsvm-core/doc/`.

- **최근 (2026-07-08): 초기 cal 붕괴 회귀 픽스 + cal SSOT 반영 + 기본 8캠(개별 8장치, S3000ABR 기준) + 런타임 설정 README 문서화.** `b2340` 까지 배포.
  - **초기 cal 붕괴 회귀**(전 카메라 파라미터 엉망): `cam_input_order` 의 **이중 사용**이 원인 —
    LoadV1CalFile 은 "cal 레코드→카메라" 매핑으로 쓰는데 ApplyConfig(60aa741)의 vcap 클램프가
    기본 config(단일 vcap)에서 `[0,0,0,0]` 으로 뭉갬 → 레코드 4개가 전부 cams[0] 에 덮어써짐.
    `cal_cam_order`(+ 프리셋 `cal_record_order`)로 분리해 픽스. 교훈: 한 배열의 캡처 배선/cal
    레코드 매핑 이중사용 금지.
  - **cal SSOT 반영**: CalLoadModule 성공 시 CalResetCams 즉시(런타임 현행 cal = 초기해),
    Calibration 헤더 펼침 시 선택 모듈 자동 로드(기본 Ox4 도 최초부터 base_length 명세 UI 표출),
    mrQuery 명세 기반 제네릭 설정 UI(`ShowCalModuleParams` — 키 하드코딩 0).
  - **기본 = 개별 캡처 8대(S3000ABR)**: 보드 프리셋 NV12 8×1920 + `capture.devices[]` 기본값
    (`/dev/video0..3`+`/dev/video11..14`). x86 은 avio-file 이 `mcap_cap%d` 4연접 2장을 1920 요청폭으로
    자동 슬라이스해 동일 레이아웃 시뮬레이션. 구 4연접×2 는 `test/bin/mcap_2x4.config` 로.
    프리셋 8엔트리 + `CAM_PRESET_CNT`(배열 크기 유도, OOB 차단) + static_assert 정합 가드.
  - **런타임 설정 문서화**: README 에 3계층 병합(프리셋/`lbsvm.config`·`--config`/인라인 JSON)·
    병합 규칙·capture 매핑 모델(cam_input_order+cam_area, 플립=영역 반전 인코딩)·키 레퍼런스·
    레이아웃 예시 3종 신설.
  - **프레임 부재 견고화**: main 바인딩 all-or-nothing → 채널별 개별(소스 일부만 있어도 가용
    카메라 동작), Test UI 무가드 역참조 3곳 가드(VCap `vbuf[0]`/썸네일 `device_image`/상세 패널)
    — eyel2sdk 예제에서 노출된 잠복 결함(보라 배경 무합성 + 메뉴 진입 즉사).
  - **eyel2sdk 동기화**(`b13`~`b14`): svmdemo 최신판을 예제에 반영 — 범위는 "Test UI 전반 +
    서브는 TpTopview 까지 + Cal 연동"(UVMap/grid 는 SDK 예제 범위 밖, dpgl 은 core 의존이라 포함).
    main 은 vcap_n==1 특수분기 제거 수술, bin 에 mcap 에셋/cal json(base_length 판) 동기화.

- **(2026-07-07): CAM_CNT 런타임화(카메라 수·vcap 매핑 설정화) + avio-file 파일드라이버 일반화. 빌드 통과·배포 완료(사용자), 8채널 실테스트는 별도 진행 예정. 재개 메모는 [`lbsvm-core/doc/HANDOFF_2026-07-08.md`](file:///L:/lbsvm-core/doc/HANDOFF_2026-07-08.md).**
  - **동기**: `VCAP_CNT` 는 이미 `capture.count` 로 런타임화됐는데 `CAM_CNT` 는 매크로 `(4)` 로 고정
    → 카메라 수·`cam_area`·"어느 vcap 의 어느 영역" 매핑을 설정으로 못 바꿈. 특히 "4연접 영상 2채널"
    (Maxim 스트립 ×2)을 표현 불가. 이걸 보강(이전 항목의 "SVM 소비는 CAM_CNT=4 유지"를 대체).
  - **CAM_CNT → `cam_count` 런타임화** (`test/src/svmdemo_config.h`·`svmdemo_core.*`·`svmdemo_main.cpp`·
    `svmdemo_ui.cpp`): `CAM_MAX(8)`(배열 상한)+`CAM_CNT_DEFAULT(4)`+`capture.cam_count`(런타임, 1..CAM_MAX).
    `cams[]`/`cal_cams[]`/`cam_input_order[]`/`cal_pushed·bak[]` 를 CAM_MAX 로, 루프는 `cam_count` 로.
    SVM 투영(bowl)은 `CreateProjSurfaces` 가 count≥4 요구라 앞 4대 제약 유지 — 잉여 카메라(5..8)는
    뷰어 전용("camN" id 자동). cal(`CalRun`/`RenderUI`)도 `cam_count` 전달, CAL_IDS 는 앞4=rear/front/
    left/right, 그 이상 "camN".
  - **카메라↔캡처 매핑 정리**: 각 카메라 i 는 `cam_input_order[i]`(읽을 vcap 채널) + `cams[i].area`
    (그 영상의 정규화 영역)로 입력 특정. `ApplyConfig` 에서 `cam_input_order` 를 `[0,vcap_count-1]` 로
    **클램프** → `vcap_count==1` 이면 전부 0 으로 수렴. 덕분에 `svmdemo_main.cpp` 의
    `vcap_n==1 && CAM_CNT>1` **특수분기 2곳 제거**, 단일 대형영상·다채널을 한 경로
    (`vbuf[cam_input_order[i]]`)로 통일. UI "Visible Area" 에 **Source VCAP** 슬라이더 추가(런타임 편집).
    "4연접 2채널" = `cam_input_order=[0,0,0,0,1,1,1,1]`+영역 1/4씩, 또는 `count=2`+`cam_area` 통짜.
  - **avio-file 파일드라이버 일반화** (`drv/avio-file/src/avio_file_main.cpp`): 시뮬레이션 드라이버의
    본질 = "소스가 분리/연접이든 소비자 요청대로 조립"인데 "가로 4연접"에 하드코딩돼 있던 것을 일반화.
    - **모델**: 소스 파일들의 가상 연접 → 호스트가 연 각 디바이스 v 에게 폭 W(요청 fmt) 창
      `[v·W,(v+1)·W)` 를 준다. 파일 폭 F 대 W 로 — `F==W` 통짜 1채널(하위분할은 `cam_area` 몫),
      `F>W`(정수배) `F/W` 슬라이스(`h_to_v` 채널연속화). `Σ파일폭 = vcap_count×W` 정합.
    - **제거**: `VIN_CH=4` 하드코딩·단일 전역 `capman`·`video_no` 고정슬라이스·죽은 `get_texture`·
      open 의 `VIN_W` 해상도추정 heuristic. **신규** `TCapFileMan`(소스별 버퍼 + `EnsureFrame(template,W,H)`
      캐시 + `ChannelData(video_no)`). `%d` 는 0.. 존재 파일까지 개별 로드(없으면 단일).
    - 결과: wide1장→4채널 / wide2장→8채널 / wide1장 통짜(cam_area) / 개별8장 → **한 경로로 균일**.
      물리 파일간 concat·진짜 픽셀포맷 변환은 **미래 ffmpeg/gst 미디어 드라이버**로 분리(백로그).
  - **빌드**: VS2022 lbsvm-core 전 솔루션 + `drv/avio-file` Debug 오류 0. 배포(lib/lbx 서브모듈)는
    사용자가 완료. **8채널 실테스트 미완**(육안·실config 필요).

- **최근 (2026-07-07): svmdemo cal 모듈(cal-ox4/hkmc) 연동 완성 — Auto/Manual Cal + 실시간 연동 + 3계층 런타임 설정. 하루 6커밋(`e6170d6`→`eada20f`), 계획/진행 정본은 [`lbsvm-core/doc/Cal_Module_Integration_Plan.md`](file:///L:/lbsvm-core/doc/Cal_Module_Integration_Plan.md).**
  - **Step 0 — svmdemo 양측 동기화**: Test UI 는 eyel2sdk 최신(Camera Parameters 편집기
    역포팅: 썸네일+Lens/Area/Pose 편집), 하위 창은 lbsvm-core 최신(TpTopview 통합 Tuning,
    신형 그림자, 오도메트리)을 eyel2sdk 로. Driving Simulator 삭제, TP FBO Debug 체크박스화.
    **이후 eyel2sdk 는 당분간 동결** — lbsvm-core 만 작업.
  - **Step 1 — 3계층 런타임 설정**: 내장 기본값(`MakeDefaultDemoConfig`, 플랫폼 #ifdef 이 안에만)
    ← `lbsvm.config` ← 인자(`--config <path>` / `'{...}'` 인라인 JSON). `var_overlay`(객체 재귀,
    스칼라/배열 통째). NM12(텔레칩스)/NV12(RK)를 한 리눅스 바이너리로 커버.
    `VCAP_MAX(8)`+`capture.count` 분리(s3000abr 8캠 대비, SVM 소비는 CAM_CNT=4 유지).
  - **Step 2/3 — cal 브리지 + UI**: 런타임 DLL 로드(링크 無, `close_library` 명시 언로드),
    임시 `LBX_CAMERA[4]` 브리지(TLBCamera 는 vptr 라 배열 전달 불가, 소문자 id 필수),
    Calibration 메뉴(Method 라디오/Auto Cal/결과/Save .cal/Revert), Manual Cal =
    모듈 RenderUI 프레임 구동(`LBX_CAL_UI_TARGET` 을 lbx-intf 로 공개 승격, 창내 Auto Cal 버튼).
    `cal.autostart`("auto"|"manual") 훅으로 부팅 즉시 캘 경로 진입(자가 재현/헤드리스용).
  - **버그 3건 규명·픽스**:
    1. cal-flood `cal_module_run` 이 ok/failed_mask **미집계**(초기값 반환) → 전 카메라
       실패도 전체 OK. 집계 추가(`de09bb6`).
    2. **좌표 원점**: cal/패턴=뒷범퍼 원점, 런타임=차량 원점(`LoadV1CalFile` 이 extents.back
       감산). 축·각·렌즈 동일, x 만 다름 — 변환 누락으로 Apply 후 전 카메라가 전방으로 밀림
       ("초기치가 더 정확" 증상). 브리지 양방향 원점 변환.
    3. **Master Cal 오픈 50s(Debug)/10s(Release)**: `TextureFilterCallback` 이 GFX 태그
       u64 핸들을 GLuint 로 잘라 존재하지 않는 id 에 `glBindTexture`(GL 은 bind 로 객체 생성)
       → PVR 에뮬서 첫 호출당 수 초. native id/target 추출로 픽스, 계측 27,757ms→3ms.
  - **실시간 연동(구조 개정)**: `CalPullPush()` — `cal_pushed_*` 대비 변경 주체 판정, 모듈
    변경(Run/수동 solve)은 즉시 렌더로 pull(+`LBX_CAMERA_Update`), 호스트 변경(Camera
    Parameters 편집)은 즉시 모듈 오버레이로 push. 복사본의 존재 이유 = 배열 브리지 +
    Revert 스냅샷뿐. Master Cal 뷰어는 area UV 로 StdImage 표시(스트립/반전 흡수),
    그리드는 유동 배치(가용폭 랩핑, VCAP 뷰어도 통일).
  - **다음 세션**: ① `TextureFilterCallback` 개선 — gfx 와 원활히 맞물리려면 `texture_id`
    를 `u64_t` 로 바꾸거나(콜백 계약 4곳: lbx-gui `ImageViewerState`·lbx-intf
    `LBX_CAL_UI_TARGET`·cal_ui·svmdemo), 아예 gfx 연동해 GFX_TEXTURE_2D 핸들을 계약으로.
    ② 캘 영상으로 Ox4 실검증(스트립 슬라이스 표시 육안 확인 포함).
    ③ cal 모듈 USB 핫로드(검색 경로 지정 — cfg `cal.module_dir` 또는 UI 입력).

- **(2026-06-26 오후): tptopview Step 2 핵심 완성 — FBO 를 동적그림자 메시 채움 소스로 (커밋 `57e1a11`+`4ab9e3a`)**.
  - FBO 합성 모드 3종(Off / Mix[fbo.a 무시] / Overlay[fbo.a로 빈영역 Laplacian]), shadow
    color 최종 곱(FBO 도 회색 그림자 따라 어두워짐), flat 투영면 구멍 `shadow_bounds` 자동매칭
    (+margin), FBO 핑퐁 2벌 디버그(ImGui).
  - **핵심 버그픽스 — `tp.Clear()` variable shadowing**: 바깥 `i` 가 안쪽 for `i` 에 가려져
    `Release()` 누락 → FBO bind 잔류 → 메인 화면 clear(배경색)가 FBO 에 오염. 그동안의
    '배경색 오염·차오름'·'displacement 가드 먹통'의 **실제 원인**(alpha 누적·가드 다 헛다리).
    한 줄짜리 shadowing 이 하루를 잡아먹음.
  - **교훈**: FBO 투영면을 bowl 과 분리(scene_tp/flat)해 따로 가져간 설계가 **옳았음 확인** —
    flat 구멍을 그림자 영역만큼 키워야 과거영상이 채워진다(작은 구멍이면 현재영상만 덮어써짐).
  - 상세·담주 TODO(FBO flat 의 전후방 커버 쿼드 제거): `doc/Transparent_Topview_Composite_Design.md` §8.

- **최근 (2026-06-26 오전): 동적그림자 외곽 블렌딩을 바깥 스커트→안쪽 페이드로 전환 (커밋 `bf60cb5`)**.
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

- **최근 (2026-07-08): 계약 0.3 적응 + 헤드리스 Ox4 성립 + SSOT + 전환 크래시 픽스.** `b16` 배포.
  - **base_length 유도를 Run 경로로**: UI 전용 TU 에 갇혀 있던 앵커 유도를 알고리즘 TU
    (`cal_ox4.h/cpp`)로 이관 — 헤드리스 호스트가 `params["base_length"]` 주입만으로 AutoCal 완주.
    앵커 우선순위: 패널 anchors → params anchors(구형식) → `desc->resolve_anchors` 유도(필수 키
    누락 시 "missing" 보고 + 캘 스킵). **cal-ox4.json 을 유도 결과(anchors) 대신 유도 입력
    (base_length=1000/veh_length=5155)으로 전환 — 실영상 autorun score 소수 7자리 동치로 회귀 증명.**
  - **SSOT**: cal_ox4_ui 정적 사본 전면 제거(params 디렉트 바인딩 — 호스트 제네릭 UI 와 같은 키라
    구조적으로 항상 일치), cal_ui 재로드를 포인터 동일성 → 앵커 내용 지문으로, 카메라 pos/lens
    스냅샷 비교 → LBX_CAMERA_Update 자가 갱신.
  - **전환 크래시 3중 픽스**: 전환 프레임의 device_image NULL 공백 중 스냅샷 latch → 파생캐시
    영영 미갱신 → 쓰레기 투영 → PCM OOB 사망. ①latch 금지 ②Run 이 calibrate 직전 Update 자체
    보장 ③PCM 샘플 경계 가드(OOB=ctNotCorner 탈락). 재현은 `CAL_HOST_TEST_SWITCH` env 훅.
  - **cal-host 하니스**: 모듈 적용 시 기본 .cal 자동 로드, svmdemo 의 Camera Parameters 섹션 이식
    (LBX_CAMERA C API, VCap 설정 제외), mrQuery 명세 제네릭 설정 UI, 중복 채널 뷰어 삭제.

## eyeL-2100R — 구형 레포 포팅 (별건)

- **최근 (2026-07-01): CAN 연동 완성 — 휠펄스 오도메트리 이식 + 실차 기어 ID 픽스. 화면검증 완료, 차량테스트 대기.**
  상세는 위 RESUME 문서 §3. 커밋 `b42e9f98`(gear) + `ca1771d3`(오도메트리).
  - **원인 규명**: "휠펄스 들어오는데 바퀴 안 돎" = 활성 CAN 핸들러가 `parse_kia_carnival`(JSON_CAN_DB
    off라 DB주도 `parse_vehicle_json`은 죽은 코드)인데, 실차 gear=**0x111(TCU11)**만 미처리(0x52A/0x59B
    Kia Carnival/K7만 봄) → gear 'P' 고착 → `ani.Begin`이 D/R서만 불려 정지. SAS 0x2b0·휠펄스 0x387은
    원래 완비. MCU는 PHY필터 아니라 `rxCanId[]` 사전등록 ID만 UART 포워딩 → 0x111 등재+parse case 추가.
  - **오도메트리**(lbsvm-core `update_pulse_diff` 이식, 2축 적응): 254→0 오버플로·WHL_DIR방향·기어폴백
    (방향마스크 0x03f→0x03 버그픽스). `UpdateOdometry`(프레임당): displacement=후륜Δ평균×mm/pulse[rear],
    바퀴각=거리/(2πr)×360 (mm/pulse·굴림반경 정확값, 구 rot_inc 7.8° 매직 폐기).
  - **가상 카운트업**(기본ON=엔지니어링, 실차 시 해제)이 동일 update_pulse_diff 경로로 전 로직 검증.
    튜닝 UI: 기어 P/R/N/D 버튼, mm/pulse Link F=R, Wheel radius, Virtual pulse(±pulses/frame).
  - **다음**: ① 실차 mm/pulse·SAS범위·전륜조향각 튜닝. ② lbsvm-core 역반영(가상카운트업·바퀴애니·0x03
    픽스; 최대한 동일코드로 이식해둠).

- **(2026-06-30): TpTopview + 동적그림자(셰이더 블렌딩) 역포팅 핵심 완료, 다음=CAN 연동.**
  상세 현황·재개 메모는 **[`I:\eyeL-2100R\docs\TPTOPVIEW_PORT_RESUME.md`](file:///I:/eyeL-2100R/docs/TPTOPVIEW_PORT_RESUME.md)** 참조
  (어디까지 됐고 뭐 더 해야하는지, 빌드/실행, 커밋상태, CAN 연동 계획까지 그 문서만 보면 이어갈 수준).
  - **완료(화면검증)**: TpTopview 누적(컬러전용 FBO·viewport 우회·의미축 어댑터 `veh_world_vec`),
    동적그림자 신방식(스커트 제거→edge_dist+셰이더 smoothstep), TP를 그림자 셰이더에 합성(탑뷰·3D뷰),
    그림자 클립↔TP 투영면 연동(see-through), Shadow Color/alpha, 선회부호, UI를 lbsvm-core
    "Topview & Shadow Tuning"에 상응(extents Lh/Fr/Rh/Bk, Save/Load=nlohmann json), mm/pulse(축별).
  - **lbsvm-core 원본에도** 의미축 어댑터 반영(동작보존, ISO=항등; `feature/shadow-upgrade` 미커밋).
  - **다음**: CAN 연동 — 0x2b0(SAS)/0x111(gear)/0x387(휠펄스) 등록, **가상 disp→4바퀴 휠펄스 가산
    (254→0 오버플로)→모션+바퀴애니**, 후진=기어. lbsvm-core에도 적용 대상. (자세히는 위 RESUME 문서)
  - 2100R 커밋 `2759206c`(핵심) + 이후 UI/mm-pulse/어댑터 미커밋.

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

- **최근 (2026-07-08): lit-build 미등록 프로젝트 임시 빌드 지원**. `.lit [[project]]`
  에 없는 프로젝트도 이름(basename)으로 빌드 가능 — 경로로 매치 안 되면 루트 아래를
  제한 깊이(4)로 검색해 빌드 레이아웃 가진 동명 디렉토리를 찾아, `.lit` 의
  `[msbuild]`/`[linux]` 환경 정보를 그대로 써서 임시 컴파일한다. 미등록(one-off)
  대상은 배포와 "변경 없음 스킵"을 끄고 항상 컴파일하되 **버전 bump 는 유지**
  (version 파일 있으면 올림). `lit-build ltc` 로 검증. (`lit-build.py`/`.md`)

- **최근 (2026-07-08): lit-ship 릴리스 경로 첫 실전 검증 — 전 15모듈 minor 릴리스 완료**.
  lbx-core **v2.2.0** / lbx-intf **v0.3.0** / 나머지 13모듈 **v0.2.0**
  (plat-glwl·cal-flood 는 첫 릴리스). 번들 태그 **lib/lbx v2.2.0 · lib/lbsvm v0.2.0**
  (lit-bundle 매니페스트 annotation, 수동 생성).
  - **검증된 절차**: ① minor bump 예측(minor+1, patch/build=0) → ② 모듈별
    RELEASE-NOTES.md 에 `## MODULE vX.Y.0 (날짜)` 큐레이션 섹션 선작성(diff 정독 기반)
    + 선커밋 → ③ `lit-build --release --minor --force-build --pick auto --json` 일괄
    (19분) → ④ 실패 모듈은 `--release --no-bump` 재실행. 큐레이션 섹션이 있으면
    lit-deploy -r 이 LLM 없이 그 본문을 서브모듈/메인/태그 메시지에 사용 — 배포 저장소
    사용자에게 개선사항이 그대로 전달됨(하이브리드 경로 설계 의도대로 작동).
  - **사고 1건**: eyel2sdk 의 스테일 `index.lock`(sweep 이전부터 존재)으로 lib/lbsvm
    pull 이 WARN 으로 넘어가 **낡은 lbsvm(0.1.5)으로 gate 빌드·태그** → 번들
    all-or-nothing 가드가 검출(0.1.5≠0.2.0). lock 제거 → v0.2.0 태그 삭제·재릴리스로 복구.
  - **도구 개선 백로그(내일)**: ① 릴리스 모드에선 서브모듈 pull 실패 fatal + 스테일
    lock 감지 ② Generated-by 오표기(큐레이션인데 `qwen3.5:9b`) — curated/외부 작성자
    표기 ③ Dependencies 에 자기 배포 대상이 `-dirty` 로 기록 ④ 서브모듈 commit 실패에도
    `LIT_JSON ok:true` ⑤ 부분 재실행용 "번들 태그만" 모드 ⑥ lib/lbx 의 lbx-vk.dll
    VERSIONINFO 없음(스테일 산출물) 정리 ⑦ lit-ship 스킬에 릴리스 절차 섹션 추가
    ⑧ 번들-gate 연동 태깅 방침 결정(아래).
  - **번들 버전 방침(논의 중)**: 현재는 `.lit [release] version_from` 대로 대표 모듈
    dll 버전을 번들 태그로 사용(lbx=2.2.0, lbsvm=0.2.0). eyel2sdk 릴리스와의 연동은
    eyel2sdk 태그의 서브모듈 SHA 고정 + 번들 태그 매니페스트로 이미 추적 가능.
    gate 버전 일치 태깅이 필요하면 별도 alias 태그(`eyel2sdk-v0.2.0`) 추가가 절충안.
    → **07-09 확정**: 아래 릴리스 도구 개선 항목 참조.

- **최근 (2026-07-09): 릴리스 도구 개선 — 어제 백로그 8건 완료 (WDLABD2411-579)**.
  - **번들-gate 연동 방침 확정(⑧)**: 번들 태그는 종전대로 `version_from` 대표 모듈
    버전(artifact=진실)을 쓰되, **gate green 빌드 직후 · gate 배포 전**에 걸도록 순서
    변경 — gate(eyel2sdk) 릴리스 노트의 Dependencies(`git describe`)가 해시 대신
    `v2.2.0` 같은 태그명으로 기록된다. "SDK 버전 ↔ 번들 버전" 대응을 버전 정보로
    읽는 것이 목적(엔지니어 진입점 = eyel2sdk). 서브모듈 포인터 = 태그 대상 커밋이므로
    별도 pin 절차 불필요. 사전 점검 실패 시 gate 릴리스도 중단(낡은 조합 봉인 방지).
  - **lit-update**: pull 실패 시 종료코드 1(→ lit-build 가 프로젝트 중단, 낡은
    서브모듈로 조용히 빌드되는 사고 방지) + 스테일 `index.lock`(30분+) 자동 제거·재시도(①).
  - **lit-deploy**: 릴리스 중 서브모듈 pull/commit/push 실패 시 메인 커밋·태그 전에
    중단(①④, ok:false 반영), 큐레이션 릴리스의 Generated-by 를
    `curated RELEASE-NOTES.md` 로 사실대로 표기 + `--generated-by` 릴리스 지원(②),
    Dependencies 에서 자기 배포 대상 서브모듈 제외(`-dirty` 노이즈 제거)(③).
  - **lit-build**: `--tag-bundles`(빌드 없이 번들 태그만 — 부분 재실행 보완)(⑤).
  - **lib/lbx 정리(⑥)**: 폐기된 lbx-vk 스테일 산출물(dll/lib/헤더) 제거(`dd04fd2a`) —
    lit-bundle 매니페스트 경고 해소. lbx-gl 은 구형 코드가 남아 있어 유지
    (lbx-vk/lbx-gl → lbx-gfxvk/lbx-gfxgl 대체가 최종 수순).
  - **lit-ship 스킬(⑦)**: 검증된 릴리스 절차(노트 큐레이션 선작성→선커밋→sweep→복구)를
    스킬 문서에 명문화. 도구 .md 3종(build/deploy/update)도 갱신.


## ltc — Litbig Terminal Controller (진행)

UDP/UDS 소켓으로 명령을 주고받는 터미널 컨트롤러. 임베디드 앱의 clink 엔지니어링
메뉴 입력 등에 쓴다. `tool/ltc` 독립 레포(`litbig-git/ltc`).

- **최근 (2026-07-08): 플랫폼별 빌드 정비 + 버전 임베드 + 기본 도메인 정리 (커밋 `8cae9a8`)**.
  - **기본 도메인**: 비-Windows 는 `AF_UNIX`(UDS) 기본으로 (Windows 는 `AF_INET` 유지).
    recv 루프도 `AF_UNIX` 처리 추가. 런타임 `>domain inet|unix` 전환은 그대로.
  - **Makefile**: lbx-core 방식으로 재구성 — 중간 산출물을 `build/linux/$(PLATFORM_ID)/`
    로 격리(arch 교차 빌드 시 `.o` 안 섞임), `vpath`+패턴룰+`-MMD -MP` 헤더 의존성 추적,
    실행파일은 `bin/$(PLATFORM_ID)/` 에 `$ORIGIN` rpath 로 배치, x64 는 lbx-core(ASan)
    와 링크 호환 위해 ASan 계측.
  - **버전 파일/rc**: `src/version.txt`(숫자) + 배포된 `lbx_version.h` 로 lbx-core 와
    동일한 단일 원천 버전 임베드. Linux 산출물에 `LBVERINFO=ltc M.m.p.b`
    (`LBX_EMBED_VERSION_INFO()`), Windows 는 `build/vs/ltc.rc` VERSIONINFO. `.gitignore`
    가 `build/*` 로 새 `.rc` 를 삼키던 문제도 수정(build/vs 프로젝트 소스만 추적).
  - x64 크로스 빌드로 임베드까지 검증. 실행 방법은 [`d:/doc/embedded-run-guide.md`](file:///D:/doc/embedded-run-guide.md).
