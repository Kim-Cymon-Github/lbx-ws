# 프로젝트 로그 (크로스 레포 현황)

레포별 진행 상황과 다음 할 일을 한곳에 모아 "다음에 뭘 할지"를 빠르게 파악하기 위한
문서다. 시점 스냅샷이므로 작업이 끝나거나 방향이 바뀔 때 갱신한다. 상세 설계는 각
모듈의 `doc/` 를 본다. 정적 구조는 [architecture.md](architecture.md).

## lbx-gfx — graphics thin layer (활발)

- **완료**: device/swapchain(GLES+VK), mesh upload + `gfx_draw_mesh`(unlit/blit),
  external image CPU import 텍스처, **depth buffer(GLES+VK)**, VK 테스트를 gfx 소유
  swapchain 으로 이전(ImGui 는 오버레이), GPU 선택 유틸(통합 GPU 우선),
  표준 stdio/stdlib 호출을 lbx-core 래퍼로 전환.
- **다음**: Phase B(Mesh + PBR — PBR 셰이더, 텍스처 로딩 `gfx_texture_2d`,
  pipeline 캐시, glTF/OBJ 로드), Phase C(external texture + textured-blit,
  dma_buf GLES 구현).
- **보류 / 숙제**: VK 멀티뷰포트와 swapchain 리사이즈(재생성), 진짜 프레임
  파이프라이닝(슬롯별 command buffer 로 직렬화 제거), `calloc` -> 전용 zeroing
  alloc 함수 도입, 동적 mesh 스트림(vec2 position, per-frame color stream) —
  동적 그림자 연동의 전제.
- 설계: `lbx-gfx/doc/plan.md`, `lbx-gfx/doc/ndc-convention.md`.

## lbsvm-core — SVM 본체 (활발, 주 작업 대상)

- **진행**: Shadow/Poly/Geo 리팩토링(이름 변경보다 의존 방향 정리·중복 제거 우선,
  mesh 경계 선정리). working 브랜치는 `feature/shadow-upgrade`.
- **다음**: 동적 그림자 RADIAL_LOD 메시를 `GFX_MESH` 로 이주(위 lbx-gfx 동적 스트림
  숙제와 맞물린다).
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
