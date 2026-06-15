# lit 워크스페이스 아키텍처

워크스페이스 전체 구조와 모듈 간 의존을 한곳에 정리한 문서다. 모듈 고유의 빌드/구현
세부는 각 모듈의 `CLAUDE.md` / `doc/` 를 따른다. 진행 현황과 다음 할 일은
[project-log.md](project-log.md) 를 본다.

## 레포 구성

`L:\` 는 `lit` 멀티레포의 얇은 루트다. 각 모듈은 **독립 git 레포**이고, 루트는 `.lit`
매니페스트로 느슨하게 묶는다(서브모듈이 아니다 — 의도된 설계). 멤버십은 `.lit` 의
`[[project]]`, 조합 스냅샷은 `[release.bundle]` 태그가 담당한다.

## 빌드 / 의존 순서 (.lit 기준)

`.lit` 의 `[[project]]` 목록은 거의 의존 순서대로다:

```
lbx-core -> lbx-intf -> lbx-cal -> lbx-geo -> lbx-gfx -> lbx-gl -> lbx-gui
  -> drv/avio-file -> drv/avio-v4l2 -> drv/plat-glfb -> drv/plat-glwl -> drv/plat-glwin
  -> cal/cal-flood -> lbsvm-core -> eyel2sdk
```

빌드/배포 도구는 `ltk`(`lit-build`, `lit-deploy`, `lit-update`, ...). 빌드 환경은 `.lit`
의 `[[linux.env]]`(x64, Telechips tcc803x/807x, Rockchip rk3568/3576)와 `[msbuild]`(VS2022).

## 계층

### 기초 라이브러리 (lbx-*)

- **lbx-core** — 외부 의존이 없는 저수준 공용 라이브러리. 모든 것의 기반.
  고정폭 타입(`i32_t` 등), `svec`/`ustr`/`var_t`, 스트림/직렬화, 수학, 이미지,
  로그(`Err_`/`Log_`), 메모리(`alloc_memory`/`free_memory`).
- **lbx-intf** — 모듈 인터페이스/IPC/소켓. 플랫폼 드라이버 인터페이스 헤더
  (`lbx_intf_plat.h` 의 `LBX_PLATFORM_DRIVER`, `LBX_WINDOW`)가 여기 있다.
- **lbx-cal** — 카메라 캘리브레이션.
- **lbx-geo** — 순수 기하(rect/roundrect/polygon/arc/nearest/투영). 외부 의존 없음.
- **lbx-gfx** — graphics API 독립 thin layer (개발 중). 한 레포에서 여러 DLL
  (공통 `lbx-gfx` + 백엔드 `lbx-gfxgl`/`lbx-gfxvk`). mesh/material/draw 중심이고,
  장기적으로 mesh 소유권이 이쪽으로 모인다.
- **lbx-gl** — OpenGL/ES 래퍼. 장기적으로 EGL 소유권을 `lbx-gfxgl` 로 옮기고 은퇴 예정.
- **lbx-gui** — ImGui 호스트(런타임 로딩). 렌더 백엔드(gl/vk)를 자체 보유.

### 폴리곤 / 도메인 (lbsvm-core 내부 계층)

- **lbsvm_poly** — Clipper2/CDT 의존. triangulation/offset/hole meshing. Shadow 외 재사용.
- **lbsvm_shadow** — 차량 그림자 도메인 정책만.

계층 방향: `lbx-geo <- lbsvm_poly <- lbsvm_shadow`. 범용 mesh 처리는 장기적으로
`lbx-gfx` 로 이동한다.

### 플러그인 (헤더리스, dlopen 로 로드되는 .so/.dll)

- **drv/avio-*** — 영상 입력 드라이버(v4l2 카메라, file 스틸샷).
- **drv/plat-*** — 플랫폼 GL 백엔드: `glfb`(fbdev/Mali), `glwl`(Wayland), `glwin`(Win32).
  공통 인터페이스는 `LBX_PLATFORM_DRIVER`(`OpenWindow`/`CloseWindow`/`ProcessEvents`/`Do`).
- **cal/cal-flood** — 캘리브레이션 플러그인.

### 응용 / 게이트

- **lbsvm-core** — SVM(Surround View Monitor) 본체이자 주 작업 대상. `lbsvm-sdk`/`lbsvm-demo` 부속.
- **eyel2sdk** — 릴리스 게이트(통합 검증 기준).

## 바이너리 전달 경로

각 모듈의 `lib/lbx`, `lib/lbsvm` 은 배포 전용 서브모듈(SSH URL)이다. 원 모듈에서
`lit-build`/`lit-deploy` 가 헤더와 바이너리를 넣어 커밋/푸시하면, 소비 모듈이
`lit-update` 로 pull 해서 가져간다. **사람이 손으로 복사하지 않는다.**

## Vulkan / GLES 렌더 구조 (lbx-gfx, 합의된 방향)

- 플랫폼(GLFW/SDL/Wayland)은 **surface 만** 제공한다. Vulkan 의 instance/device/
  swapchain/depth/render pass/command buffer 는 **gfx(또는 gfx 를 쓰는 앱)가 소유**한다.
  (GL 은 컨텍스트 자체가 플랫폼 소유라 `lbx-gfxgl` 이 기본 FBO(0)를 wrap 한다.)
- ImGui(`lbx-gui`)는 gfx 의 render pass 위에 오버레이로 그린다(런타임 로딩).
- 좌표/NDC 규약: `lbx-gfx/doc/ndc-convention.md` (canonical: NDC Z[0,1], +Y up, RH).
