# s3000abr 이식 설계 — cal 로더 통일 · TpTopview 차분 · 고스트 렌더 · Color Consistency

- 작성: 2026-10-02
- 대상: s3000abr(I:\s3000abr) + lbsvm-app(subprojects/lbsvm-app) + lbsvm-core(L:\lbsvm-core)
- 전제: s3000abr 은 lbsvm v0.9.2 / lbx v2.9.1 로 Windows 빌드·기동 확인됨(10-02 15:23). lbsvm-app #207 머지됨.

## 0. 한 줄 요약

| # | 항목 | 어디 | 크기 | 선후 |
|---|---|---|---|---|
| 0 | cal 로더를 lbsvm-core `LoadCalFileAuto` 로 통일(평문 V3·AES V3·V1 자동 판별 + 로딩 시 `NormalizeLensFocal`) | lbsvm-app(+eyel2sdk) | 작음 | 먼저. 김태호 프로 증상 후보 |
| 1 | TpTopview 호스트 차분 3건(카메라 준비 게이트·그림자 변경 시 TP 재구성·side_overlap 기본값) | lbsvm-app / s3000abr | 작음 | 0 다음 |
| 2 | 차량 고스트 2패스(DrawOpaque/DrawTransparent + blit) 로 전환, 그리기 순서 재배치 | lbsvm-app | 중간 | 1 다음 |
| 3 | Color Consistency: 코어에 포인터 배열 오버로드 추가 → ECS 시스템 + 설정 + 튜닝 패널 | lbsvm-core 0.9.3 → s3000abr | 큼 | 마지막 |

gfx 투영 경로(proj.renderer=gfx)는 **이번에 안 간다**. TP 합성이 아직 gl 이고 s3000abr 전체가 gl 이라 이득이 없다. CC 톤 곡선·비네팅은 gl 카메라 패스에서 이미 적용된다.

## 1. cal 로더 통일 (lbsvm-app, eyel2sdk)

### 현황
- lbsvm-app `src/svm/MastercalJson.cpp` 의 `load_mastercal_json` / `mastercal_write_json` 은 코어 `LoadV3MastercalJson` / `SaveV3MastercalJson` 과 같은 문서("format":"lbsvm-mastercal", vehicle/spec, pos x/y/z/yaw/pitch/roll)를 다루는 원본 구현이다. 코어로 흡수된 뒤 lbsvm-app·eyel2sdk 에 원본이 남아 있다.
- 코어 v0.9.2 의 `LoadCalFileAuto(S, aes_key, res, cams, unit, tire, NULL, inc, n, order, v1spec, &kind)` 는 '{' / AES / 바이너리를 내용으로 판별하고, 성공 시 카메라마다 `NormalizeLensFocal()` 을 건다(f.x≠1 이면 f.y/=s, k*=s, f.x=1 — 상은 같다).
- lbsvm-app 로더는 정규화를 안 한다. V3 세트 중 f.x≈0.5 인 것(Benz_CLS, eyeL-2100R 최적화기 결함 산물)을 그대로 쓰면 투영이 틀어진다.
- 키: s3000abr `config_bundle_format.h kSetBundleKey` == `pctool_dll.cpp kLbsKey` (같은 32바이트, eyeL 'litbig'). 보안 비밀이 아니라 난독화 키다.

### 김태호 프로 증상과의 연결(가설)
PC 툴 목록에서 특정 세트를 고르면 탑뷰가 "cal 0 초기화" 처럼 보이고 cal 모드에 들어가면 정상. f 가 절반인 V3 세트를 정규화 없이 로드하면 투영 반경이 반으로 줄어 그렇게 보일 수 있고, cal 모드는 cal-flood 가 다시 풀어 주므로 정상으로 돌아온다. 검증은 Benz_CLS 세트(L:\lbsvm-core\test\assets\Benz_CLS.mastercal)를 PC 툴 목록에 넣고 재현 → 1 적용 후 재확인.

### 변경
1. lbsvm-app `CalibrationLoader.cpp`
   - mastercal 경로(static·persist 둘 다)를 `LoadCalFileAuto` 한 번으로 바꾼다. `mastercal_is_json` 분기·`load_mastercal_json`·`LoadV1CalFile` 직접 호출을 지운다.
   - `aes_key` 는 프레임워크가 모르는 값이므로 **호스트 주입**: `app::svm::set_mastercal_key(const char* key32)` 또는 `SVMConfig` 에 콜백 하나. s3000abr 는 부트에서 `kSetBundleKey` 를 넣는다. eyel2sdk 는 자기 키를 넣는다.
   - 반환 `kind` 를 로그에 남긴다(`source='mastercal:v3-json'` 식).
2. lbsvm-app `MastercalJson.cpp`
   - 읽기 함수 삭제. 쓰기 `mastercal_write_json` 은 코어 `SaveV3MastercalJson` 호출로 교체(v1spec verbatim 보존 규칙은 코어가 같은 규칙).
   - `mastercal_is_json` 은 코어 `looks_like_json` 과 중복이나, 호출처가 남으면 코어 쪽으로 돌린다.
3. s3000abr `svm_bootstrap.cpp load_v1_vehicle` 도 같은 함수로(세대 무관).
4. eyel2sdk: 같은 교체. 별도 PR.
5. 검증: persist(I:\home\root\lbsvm.cal, V3 평문) · static(HMC_Xcient_12m.mastercal, V1) · AES V3 세트 세 가지로 기동 로그의 source/kind 와 화면 비교. NormalizeLensFocal 로그가 Benz_CLS 에서만 찍혀야 한다.

## 2. TpTopview 호스트 차분

라이브러리(lbsvm_tptopview) 는 v0.9.1 부터 같고 s3000abr 헤더도 HEAD 와 동일하다. 차이는 호스트 배선 3건이다.

| | svmdemo | s3000abr | 조치 |
|---|---|---|---|
| A. 누적 게이트 | `cams_ready && disp≠0` 일 때만 `tp.Update` (8f35a0b, 09-28) | `tp_initialized && travel≠0` 만 | `update_transparent_topview` 에 라이브 카메라 텍스처 준비 게이트 추가(SVMSceneSystem.cpp:697-720). 부팅 직후·카메라 끊김 중 주행 시 검은 프레임 누적 방지 |
| B. 그림자 변경 → TP | 그림자 재생성 때 `RebuildTpProjSurface` 도 호출 | `on_shadow_config_changed` 가 `m_needs_shadow_rebuild` 만 켬 | `clip_auto` 이면 `m_needs_rebuild`·`m_needs_tp_reinit` 도 켠다(SVMSceneSystem.cpp:2000-2003). Shadow 패널에서 expand 바꾸면 TP 구멍이 옛 경계에 남는 결함 |
| C. side_overlap 기본 | 0 (FBO 누적을 앞뒤로 덮어 방해) | 1 (TpTopviewConfig.h:52, app-cross.json, settings.json) | 0 으로 내림. 프로님 판단 필요 — 출하 settings.json 섹션 단위 대체라 마이그레이션 없음(값만 바뀜) |

선택(이번엔 안 함): FBO 뷰어 rgb/a 분리 표시, `shift_min_mm` 키 노출, `fbo_max_px` 2048 CAPPED 확인(차 길이 10,384mm 초과 시), 3D 뷰 `tp.Draw` 금지.

## 3. 차량 고스트 렌더 2패스 (lbsvm-app)

### 코어 계약(v0.9.2, lbsvm_render.h:271-296)
- `DrawOpaque(draw_wheels)` → 투영면 → 그림자 → `DrawTransparent(draw_glasses)`. 두 패스 모두 같은 V 행렬·뷰포트.
- 고스트 조건 `GhostMode()`: `!legacy_draw && blit != NULL && 0 < shell.col.w < 1`. 구 `Draw()` 는 `legacy_draw=true` 로 묶여 **고스트가 절대 안 켜진다**.
- 고스트 FBO 는 창 표면 해상도로 한 번 할당(3edc931), MSAA 는 `ghost_samples`(기본 4). 옵션 `ghost_wheel_alpha`, `ghost_show_hidden_wheels`.
- 모든 패스가 GL 상태를 저장·복원한다.

### s3000abr(lbsvm-app) 현재
- `Vehicle3DRenderer.cpp:443` 단일 `Draw()`, `blit` 미연결(:179-182, :271-274).
- `SVMRenderSystem.cpp:1311-1328` 반투명이면 투영면 → 차 순, 그림자는 차 뒤(:1336-1410). 블렌드 직접 리셋(:436-437, :449-455).
- 결과: v0.9.2 에서도 파트별 알파 그대로(안쪽 면 비침), 불투명 모드에서 유리가 투영면 앞에 그려지는 결함 유지.

### 변경
1. `Vehicle3DRenderer` 초기화에서 `renderer.blit = &shader.prog_blit`(기존 공유 blit 프로그램 재사용. `GetBuiltinShaderProgram("blit")` 는 프로세스당 한 번).
2. `render_vehicle_3d` 를 `render_vehicle_3d_opaque` / `_transparent` 로 분리. depth clear 는 opaque 한 번만.
3. `SVMRenderSystem::render_views` 순서: TP 바닥 → 차 opaque → `v.Draw`(투영면) → 그림자 → 차 transparent → car_overlay. `veh_transparent` 분기 삭제, 블렌드 수동 리셋 삭제. **그림자는 반드시 합성 앞**(고스트 DrawOpaque 는 화면 depth 를 안 쓰므로 뒤에 두면 그림자가 차 위에 덮인다).
4. `Vehicle3DModelConfig` 에 `ghost_wheel_alpha`, `ghost_show_hidden_wheels`, `ghost_samples` 추가 → Shadow/TPTV 패널 또는 Vehicle 패널에 노출(선택).
5. 죽은 `render_vehicle_3d_varnish` 삭제.
6. 검증: Windows 에서 투명도 10~100% 슬라이더로 안쪽 면 비침 사라짐·바퀴 옵션 동작, 불투명에서 유리 뒤 투영면 보임. 보드는 EGL_SAMPLES 4 라 기본값 유지.

## 4. Color Consistency 이식

### 코어 쪽 갭(lbsvm-core 0.9.3 필요)
- `TLBColorConsistency::Build/Sample/WedgesFromScene` 이 연속 배열 `const TLBCamera* cams` + 인덱스 산술을 전제한다(lbsvm_colorcon.cpp:120, 318, 461, 599). s3000abr 카메라는 엔티티별 `CalibrationComponent::camera` 라 연속이 아니고, TLBCamera 는 가상 소멸자가 있어 사본 배열은 쓰지 않는다.
- 조치: `const TLBCamera* const* cams` 오버로드를 추가(내부는 포인터 배열로 통일하고 기존 연속 배열 오버로드는 포인터 배열을 만들어 위임). svmdemo 는 무변경. → lbsvm-core v0.9.3 릴리스·s3000abr 범프.

### s3000abr 구조
1. **설정** `src/config/ColorConsistencyConfig.h` (POD, 기본값 = svmdemo 기본 config)
   - settings tier 섹션 `ColorConsistency`: enable, gain[4], hi_begin, hi_end, hermite, vig_apply, vig_n, auto_enable.
   - app.json 출하값 섹션 `ColorConsistencyTune`: stats(hz, cell_mm, margin_x/y, outer, min_wedge_deg, flip_y, band_half, min_inlier, min_count), auto(ref_cam, max_history, min_frames, mad_k, reject_score, min_pair_count, gain_min/max, ema_alpha).
   - 등록 3곳 필수: `ConfigRegistry.cpp init_reflection()` MetaBuilder `.data<>` 전 필드, `register_serialization_types()`, `SettingsPath.h settings_json_types()`. 누락 필드는 조용히 소실된다.
2. **시스템** `src/systems/ColorConsistencySystem.{h,cpp}` (`ISystem`, priority 10 — SVMScene(-50) 뒤, Render(50) 앞)
   - init: "svm" 그룹 `CalibrationComponent` 를 `name`(front/rear/left/right)으로 4슬롯 포인터 배열에 매핑. `ref_cam` 도 이름 기준.
   - 재빌드 신호: `on_construct/on_update<ProjectionConfig>`, `on_update<VehicleComponent>`, `on_construct/on_update<CalibrationComponent>` 에 connect → `m_dirty`. shutdown 에서 disconnect.
   - update(매 프레임): `ApplyCC()` — `cam.cc = (on? gain : 0, hi0/255, hi1/255, hermite)`, `cam.vig = (vig_apply&&!bypass ? n : 0, 0,0,0)`, `cc_est.SetVignette`. 카메라 컴포넌트에 **직접 쓴다**(`reg.patch` 금지 — 투영면 전부 재빌드됨). 캘 재로딩으로 TLBCamera 가 교체돼도 매 프레임 쓰므로 자동 복구.
   - BuildCC: `WedgesFromLegacy(FRONT, BACK, LEFT, RIGHT, v1_proj_params.PROJ_ANGLE_FR/BLEND_FR/ANGLE_BK/BLEND_BK, min_wedge_deg, left_sign, stat.clip_area, …)` → `cc_est.Build(ptrs, 4, …)`. 첫 프레임(device_image 없음)은 미룬다.
3. **샘플링 레이어** `ColorConsistencySampleLayer` (`IRenderLayer`, `bind_camera_textures` 뒤·LbGUI 앞에 등록)
   - TickCC: `stats_hz` 주기로 `Sample(ptrs,4)` → `ComputeStats` → `PushHistory` → `auto ? Solve` → (`vig_diag ? VigAccumulate/VigSolve`). 이 자리여야 `planes[0].data` 가 이번 프레임 버퍼(pin)다. 비용 ms EMA 계측.
   - solve 성공 시 `gain[i] = GainSmoothed(i)` 를 설정 구조체(런타임 사본)에 반영 → 다음 update 의 ApplyCC 가 민다.
   - blink: `blink_ms` 마다 bypass 토글.
4. **패널** `src/debug/ColorConsistencyDebugPanel.{h,cpp}` (`IDebugPanel`, ShadowDebugPanel 골격·`kPanelWindowFlags`·`value_row`/`step_spinner`·`EditCommitThrottle`)
   - 섹션: Enable / 카메라별 gain(0.5~2.0) + Reset / Bypass(A) · Blink A/B · ms / highlight fade Y(hi0,hi1) / Hermite / Tone curves(Y'−Y) PlotLines(`LBSVM_CC_CurveBuild/Eval`) / Collect · Log · rate Hz · Rebuild · cell/margin/outer/min wedge/flip(변경 시 dirty) / 상태(samples·wedges·round·valid pairs·round ms avg/max) / cc_pairs 표 / Vignetting(Accumulate·Reset·Apply·n·Use fitted·fit 표시) / Auto · Log · Reset history · 상태 / Save · Revert · Reset All(ShadowDebugPanel 규칙 그대로).
   - 오버레이(웨지 선·샘플 점) 는 **2단계**. s3000abr 은 gfx 패스가 없으므로 GuidelineRenderSystem 식 native GL 선·점으로 다시 쓴다. "CC Samples" 썸네일 창도 2단계.
5. **등록**: `S3000abrApp.cpp` add_system / `rs.add_ui_layer` / `m_debug_layer->add_panel`, `src/meson.build` 소스 3개 추가. 파일은 UTF-8 BOM, 주석 한국어, 로그 `LogIP/LogWP`, `LOG_MODULE("sys.cc")`.
6. **보드 전제**: Sample 은 `planes[0].data` CPU 포인터가 필요. 보드 dmabuf 프레임도 mmap 주소를 가지며 `bind_camera_textures` 가 넣어 준다(0 이면 그 카메라는 건너뜀).

### 검증
- Windows: carnival(정지 PNG 4장, app-native.json sources cam0~3) 또는 Xcient 세트로 Collect → 웨지·쌍 수·round ms 표시, Enable+gain 슬라이더로 이음매 밝기 변화, Auto 로 gain 수렴, Blink A/B. Save 후 재기동 시 settings 복원.
- 보드: round ms(기준 3.5~8.3 ms @5Hz, 4웨지 11.7만 샘플) 와 프레임 시간 확인.

## 5. 순서와 릴리스

1. lbsvm-core: CC 포인터 배열 오버로드 → v0.9.3 (lit-ship). svmdemo 회귀 없음 확인.
2. lbsvm-app: §1 로더 통일 + §2 A·B + §3 고스트 2패스 → PR(번들 lbsvm v0.9.3 포함).
3. s3000abr: §2 C(side_overlap) + §4 설정·시스템·레이어·패널 → PR.
4. eyel2sdk: §1 로더 통일 → 별도 PR.
5. Windows 검증 → 보드 검증(보드 입수 후).

## 6. 미결·판단 요청
- §2-C side_overlap 1→0: 출하값 변경이라 확인 필요.
- §1 AES 키 주입 방식: 프레임워크 setter vs SVMConfig 콜백. 제안은 setter(단순, 부트 한 줄).
- §4 ref_cam 기본: svmdemo 는 front. s3000abr 도 front 로.
- CC 오버레이·샘플 창은 2단계로 미룸.
