# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 이 레포의 정체

L:\ 는 `lit` 멀티레포 워크스페이스의 **얇은 루트 레포**이자, 전체 프로젝트의 **통합 스토리/메모리 허브**다.
추적 대상은 `.lit` 매니페스트, `README.md`, `CLAUDE.md`, `doc/` 뿐이고 모듈 폴더는 전부 무시한다
(화이트리스트 `.gitignore`).

현재는 모든 레포를 한 사람이 개발하지만, 장기적으로 레포별 담당자가 배정될 예정이다.
따라서 **공통 규칙과 크로스-레포 설계는 여기에**, 모듈 고유의 빌드/구조 정보는
**각 모듈의 CLAUDE.md에** 유지한다 (모듈이 독립적으로 인수인계 가능해야 한다).

핵심 규칙:

- **각 모듈은 독립 git 레포다.** 모듈 코드 변경은 해당 모듈 디렉토리에서 커밋한다.
  이 루트 레포에는 매니페스트/워크스페이스 문서 변경만 커밋한다.
- **서브모듈로 묶지 않는다 — 의도된 설계다.** 멤버십은 `.lit`의 `[[project]]`가,
  조합 스냅샷은 `[release.bundle]` 태그가 담당한다. 서브모듈 전환을 제안하지 말 것.
- 여러 레포에 걸치는 설계 문서만 루트 `doc/`에 둔다. 단일 모듈 문서는 그 모듈의 `doc/`에 둔다.
- 각 모듈의 `lib/` 서브모듈들은 전부 SSH URL이다 — HTTP로 바꾸지 말 것.

## 모듈 지도

`.lit`의 `[[project]]` 목록은 **거의 의존 순서대로** 나열되어 있다:
`lbx-core → lbx-intf → lbx-cal → lbx-geo → lbx-gfx → lbx-gl → lbx-gui → drv/* → cal/* → lbsvm-core → eyel2sdk`

- `lbx-*` — 기초 라이브러리 (LBX: LitBig Xross-platform). `lbx-core`가 기반(타입 시스템,
  svec/ustr, var_t, 스트림/직렬화, 수학, 이미지), `lbx-intf`(모듈 인터페이스/IPC/소켓),
  `lbx-cal`(카메라 캘리브레이션), `lbx-geo`(기하), `lbx-gfx`(graphics API 독립 thin layer — **현재 한창 개발 중**),
  `lbx-gl`(OpenGL/ES 래퍼), `lbx-gui`(ImGui 드라이버), `lbx-vk`(Vulkan).
- `drv/`, `cal/` — **헤더리스 플러그인 모듈**: `dlopen`으로 로드되는 `.so` 플러그인
  (예: `avio-v4l2.so`는 의도적으로 `lib` 접두사 없음). `drv/avio-*`(영상 입력),
  `drv/plat-*`(플랫폼 GL 백엔드: glfb/glwl/glwin), `cal/cal-flood`(캘리브레이션 플러그인).
- `lbsvm-core` — **SVM(Surround View Monitor) 라이브러리 본체이자 주 작업 대상.**
  lbx 라이브러리들에 의존. 3D 렌더링/프로젝션/씬그래프/차량 모델.
  `lbsvm-sdk`/`lbsvm-demo`는 부속. 자체 CLAUDE.md의 리팩토링 지침을 따른다.
- `eyel2sdk` — 릴리스 gate (통합 검증 기준 프로젝트).
- `ltk` — **빌드/운영 도구의 메인.** likit의 모놀리식 `lit.py`를 기능별 독립 도구
  (`lit-build`, `lit-deploy`, `lit-version`, `lit-update`, `lit-commit`, ...)로 분해해
  서로의 함수를 import해 조합하는 구조로 재구현 중. 도구는 `.bat`+`.py`+`.md` 3종 세트.
- `likit` — **레거시.** 원조 `lit` 도구. ltk로 분해 이전이 끝나면 제거 예정.
  겹치는 기능은 항상 ltk가 우선. 함수 추출/정제의 출처로만 참고.
- `build/`, `tmp/`, `test/` 등 — 빌드 산출물/작업 폴더.

## 빌드/배포

- **컴파일·배포는 `lit-build`로 한다.** conda env `ltk`가 필요하므로 `conda activate ltk` 후
  `lit-build ...`를 실행하거나, 활성화를 대신 해주는 래퍼 `L:\ltk\lit-build.bat`을 그대로 호출한다.
- `lit-build` — `.lit` 기반 멀티 프로젝트 빌드 자동화. 등록된 빌드 타겟(VS + Linux 크로스 환경)을
  **전부 빌드한 뒤 마지막 단계에서 배포**한다 (update/deploy in-process 통합). 일부 타겟만 빌드된
  채로 배포되는 것을 막는 구조이므로 정식 배포는 항상 이 경로를 쓴다.
- 특정 모듈만 빌드·배포: `lit-build drv/plat-glwl`. `-j`는 기본 0(= 그 프로젝트의 빌드 unit 수만큼
  전부 동시 실행)이라 보통 생략한다. 실시간 로그 스트리밍이 필요할 때만 `-j 1`.
- **개발 중 특정 타겟만 빠르게 테스트**할 때는 그 타겟만 직접 빌드(WSL `make`, msbuild 등)하고,
  모듈 디렉토리에서 `lit-deploy`를 실행해 배포한다 — lit-build의 전체 타겟 빌드를 기다릴 필요 없다.
- `lit-build <module>` / `--from <module>` / `--config Release` / `--env <환경>` — 부분 빌드
- `lit-build --release` — 릴리스 sweep 후 gate(`eyel2sdk`)가 green이면 검증된 조합에 번들 태그
  (`lib/lbx`는 lbx-core, `lib/lbsvm`은 lbsvm-core 버전 기준)
- `lit-deploy` — lit-build가 빌드 성공 후 자동 호출(in-process)하며 단독 실행도 가능(빠른 반복
  배포). 헤더(`src/*.h` → `lib/lbx/include/`)와 툴을 복사해 넣고, 빌드가 이미 가져다 둔
  바이너리(`lib/lbx/lib/{linux,vs}/<플랫폼>/`)와 함께 add·커밋·푸시한다. **sibling 모듈들은 자기
  `lib/lbx` 서브모듈을 pull해서 산출물을 소비한다** — 이것이 모듈 간 바이너리 전달 경로다.
- 다른 모듈에서 배포한 산출물을 현재 모듈에 반영: 모듈 디렉토리에서 `lit-update`
  (lit-build가 각 프로젝트 시작 시 자동으로 수행하기도 한다).
- 크로스 툴체인 env script는 WSL 쪽 `/mnt/share/toolchain/...`.
- `.lit`(TOML): `[[linux.env]]`(x64, Telechips TCC803x/807x, Rockchip RK3568/3576),
  `[msbuild]`(VS2022), `[[project]]`+`skip`, `[deploy]`, `[release]`.

### lib/lbx, lib/lbsvm 서브모듈 취급 주의 (중요)

- 각 모듈의 `lib/lbx`, `lib/lbsvm`은 **배포 전용 서브모듈**이다. 내용물은 전부 빌드·배포가
  만들어 넣는 산출물(빌드가 두는 바이너리 + deploy가 복사하는 헤더)이며, 사람이 직접 편집하는
  파일은 하나도 없다.
- **다른 모듈에서 수정한 `.h`나 바이너리를 소비 모듈의 `lib/lbx`에 손으로 복사해 넣지 말 것.**
  디버깅 중 임시로도 금지. 반드시 원 모듈에서 `lit-build <원 모듈>`로 배포하고,
  소비 모듈에서 `lit-update`로 반영한다.
- 단, **자기 모듈 바이너리가 자기 `lib/lbx/lib/...`에 갱신되는 것은 빌드할 때마다 일어나는
  정상 동작**이다(Makefile OUT_PATH mv / VS 출력 경로). 따라서 `lib/lbx`가 바이너리 변경으로
  dirty한 것 자체는 오염이 아니며, 다음 deploy가 헤더 복사와 함께 이를 커밋·푸시한다.
- 오염 판별 기준: ① `lib/lbx/include/` 아래 **미커밋** 변경 — 헤더는 deploy가 복사 직후 커밋하므로
  미커밋 헤더 diff는 손 복사 신호다. ② **자기 모듈이 만들지 않는** 바이너리의 변경.
- **pull을 막는 오염은 `lit-update`가 자동 복구한다** (git이 덮어쓰기 거부한 파일만 restore 후
  재시도; `lib/*` 서브모듈 한정, `--no-restore`로 끔). lit-build가 매 프로젝트마다 lit-update를
  돌리므로 전체빌드가 오염으로 막히지 않는다. pull과 겹치지 않아 조용히 남은 오염만 수동으로
  `git -C lib/lbx checkout -- <경로>` 후 `lit-update`로 정리한다 (통째 `reset --hard`는 자기 모듈의
  최신 빌드 바이너리까지 되돌리므로, 했다면 자기 타겟을 다시 빌드).

## 공통 코딩 규칙 (모든 C/C++ 모듈)

- **인코딩**: VS에서 컴파일되는 소스(`*.c`, `*.cpp`, `*.h`) → UTF-8 **BOM**. 그 외 → UTF-8 (no BOM).
- **줄바꿈**: LF. 예외: 윈도 전용 파일(`*.sln`·`*.vcxproj`·`*.filters`·`*.user`·`*.props`·`*.rc`·`*.bat`·`*.cmd`)은 CRLF(.editorconfig 에 명시). 배치는 BOM 금지.
- **들여쓰기**: Space 4. 예외: Makefile 등 Tab이 문법인 파일.
- **중괄호**: K&R 기본(제어문·함수 내부는 여는 괄호 같은 줄). **인라인이 아닌 일반 함수·멤버함수 정의**만 Allman(여는 괄호 다음 줄).
  `{}`는 단일 문장이라도 생략 불가. 짧은 return은 한 줄 허용: `if (!ptr) { return NULL; }`
- **주석**: `*.h/*.hpp`(외부용)는 **영어 + Doxygen 강제**(공개 API 계약만). `*.c/*.cpp`(내부용)는 **한국어**, Doxygen 허용 — 알고리즘 상세·설계 의도·트레이드오프까지 적어도 된다. 주석에는 이모지·유니코드 기호를 쓰지 않고 ASCII(`->`, `*`, `-`)만 쓴다(커밋 메시지·사용자 대상 텍스트는 예외).
- **타입**: raw 기본 타입(`int`/`short`/`unsigned`/`long`, 정수용 `char`)·stdint 대신 LBX 고정폭 별칭(`i32_t`, `u32_t`, `f32_t`, `b8_t`, `var_t`, `fourcc_t`, `LBX_HANDLE`). 새로 짜기 전에 lbx-core에 이미 있는 루틴/헬퍼를 먼저 확인해 재사용한다.
  헤더는 `extern "C"` — 공개 API는 C ABI. 네이밍 `[모듈]_[동작]`, 헤더 가드 `lbx_[name]H`.
- **lbx 편의 타입을 최대한 살려 쓴다**: 문자열은 `ustr_t`/`UString`, 변형값은 `var_t`/`lbx::var`.
  수동 `char[]`+`sprintf`·수동 refcount 대신 `UString`(RAII·`printf`/`sprintf`·`c_str`)과 `lbx::var`의
  brace-init(객체/배열 생성)·`Key(i)`/`Value(i)`(멤버 순회 — 조회에 `[]` 지양)·`str()`(임의 스칼라→
  `UString`)·`var_to_ustr` 등을 쓴다. 소유권은 **값 반환=move**(`Detach()`), 공동소유만 `Share()`.
  `Var`/`VarRef`/`Variant`는 폐기 예정이니 새 코드는 `lbx::var`. **없어서 아쉬운 기능(메서드·헬퍼)이
  있으면 워크어라운드로 때우지 말고 반드시 사용자에게 제안한다 — lbx는 전부 사용자 코드라 얼마든지
  추가할 수 있다.**
- **커밋 메시지**: 한국어, Conventional Commits 접두사(`feat:`, `fix(build):` 등 — 각 레포 최근 이력에 맞춤).

## lbx 라이브러리 사용 규율 (새 코드 짜기 전에 확인)

lbx는 전부 사용자 코드다. **표준 C나 직접 구현으로 때우기 전에 lbx에 이미 있는지 먼저 본다.**
아래는 실제로 중복 구현·오작동이 났던 자리들이다.

### stdlib 대체 (직접 호출 금지)

| 하려는 것 | 쓸 것 | 헤더 |
|---|---|---|
| `malloc`/`realloc`/`free`/`memcpy` | `alloc_memory`/`realloc_memory`/`free_memory`/`copy_memory` | `lbx_mm.h` |
| `printf`/`fprintf(stderr,...)` | `Err_` / `Warn_` / `Info_` / `Log_` / `Dbg_` | `system/lbx_log.h` |
| `fopen` | **`lbx_fopen(name, mode)`** (`FILE*` 반환 — 그대로 대체) | `system/lbx_file.h` |

`lbx_fopen`은 경고 회피용이 아니다 — Windows에서 UTF-16 변환 후 `_wfopen_s`라 **한글 경로가 실제로 열린다**
(raw `fopen`은 ANSI 코드페이지라 조용히 실패). 게다가 `<SDLCheck>true</SDLCheck>` 프로젝트에서는
C4996이 오류로 승격돼 **빌드 자체가 깨진다**. `calloc`은 당분간 그대로(제로초기화 alloc은 추후 lbx에 추가).

### var 읽기

- 깊은 경로는 **`v.Find("vehicle.spec.length")` / `v.Find("cameras[0].k[0]")`** 가 정석이다.
  키·인덱스 혼합, 기본값(`.f32(def)`), **트리 무오염** 전부 된다.
- 체인 `[]`는 **`const`로 받았을 때만** 안전하다. 비const var의 `[]`는 std::map처럼 **키를 생성한다**(쓰기 경로).
- C API는 NULL을 그대로 흘려도 된다 — `var_of_strkey`/`var_find_rawkey`/`var_prop_as_*_def`/
  `var_to_*_def`/`var_get_value_at_index`/`var_of_index` 전부 NULL 안전.
  **소비자 쪽에 NULL 가드 래퍼를 만들지 말 것.** `var_prop_as_f32_def`·`var_prop_as_fourcc_def`·
  `fourcc_from_str`처럼 **이미 있는 것을 다시 만드는 일**이 반복됐다.

### var 쓰기 / JSON

- **JSON 읽기·쓰기를 직접 만들지 말 것** — `var_json_stream_(S)` / `var_to_json_ex(&v, &opt)`.
  var가 문법·이스케이프·숫자 왕복을 소유한다. 사본을 만들면 곧 갈라진다.
- 숫자 출력 기본은 **왕복 무손실 최단 표기**다(`0.1`은 `"0.1"`, f32 유효숫자는 필요한 만큼).
  자릿수를 못박아야 하면 `VAR_JSON_WRITE_OPT{ indent, f32_fmt, f64_fmt }`를 넘긴다.

### C 소스 작성

- **선언은 블록 맨 앞**(C90). 문장 뒤 선언은 구형 툴체인에서 안 넘어간다.
  가드를 넣을 땐 몸통을 `if (x) { ... }`로 **감싸지**, 기존 선언 앞에 `if (...) return;`을 끼워넣지 않는다.
  확인: `gcc -fsyntax-only -Wdeclaration-after-statement <file>`

## 작업 방식 원칙 (lbsvm-core 지침에서 일반화)

- 이름 변경보다 **의존 방향 정리와 중복 제거가 먼저**다. 이름 변경은 마지막에 모아서.
- 한 단계에서 파일 이동·함수명 변경·구조체 재설계를 한꺼번에 하지 않는다.
- **매 단계 빌드 가능 상태를 유지**하고, 기존 동작 보존이 최우선이다.
- 새 추상화는 "정말 두 군데 이상에서 재사용되는가"를 먼저 확인. 불필요한 C++식 추상화보다
  C 구조체 + 명확한 함수 경계를 우선한다.
- **결과가 뻔히 예측되는 간단한 변경은 테스트를 생략한다.** 동작에 영향을 주거나 결과가
  불확실한 변경만 빌드·실행으로 검증한다.

## 대화/문서 규칙

- 대화는 한국어 합쇼체(~습니다/~입니다, 극존칭 금지). 문서는 해라체(~이다/~한다).
- 기존 한국어 문서/주석을 요청 없이 영어로 번역하지 말 것.

## 설계 문서 위치

- `lbx-core/doc/architecture.md`, `lbx-core/doc/users guide/ko/` (mdbook, 작업 중)
- `lbx-gfx/doc/plan.md`, `lbx-gfx/doc/ndc-convention.md`
- `cal/cal-flood/doc/Cal_Module_Plugin_Design.md` (Cal 플러그인 구조의 source of truth)
- `lbsvm-core/doc/` (차량 키네마틱 모델 등), `lbsvm-core/CLAUDE.md` (Shadow/Poly/Geo 리팩토링 지침)
- 루트 `doc/architecture.md` (워크스페이스 구조·의존), `doc/project-log.md` (크로스 레포 진행 현황·다음 할 일).
- 크로스-레포 문서는 루트 `doc/`에 추가한다.

## 실무 주의

- **L:\ 전체를 대상으로 glob/grep을 돌리지 말 것** — 빌드 산출물 폴더가 거대해서 타임아웃 난다.
  검색은 항상 `<모듈>/src`, `<모듈>/doc` 단위로 범위를 한정한다.
- 드라이브 루트(L:\) 직속 파일은 Write 도구가 EPERM으로 실패할 수 있다 — PowerShell `Set-Content`로 우회.
