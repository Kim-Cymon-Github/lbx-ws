# 오빗 컨트롤·제스처·애니메이션 라이브러리 설계

작성 2026-09-28. 상태: 설계 검토 중(구현 전).

## 1. 배경과 목표

svmdemo 3D 뷰의 오빗 조작은 lbx-gui `lbx_gui_orbit.cpp` 에 있고, 입력을 `ImGui::GetIO()` 에서 직접 읽는다. 그래서 엔지니어링 UI 모듈(lbx-gui-gl + svmdemo-ui)이 로드되지 않은 상태, 즉 필드 장비의 기본 상태에서는 3D 뷰를 조작할 수 없다. 오빗은 SVM 본 기능의 조작이지 엔지니어링 UI 의 기능이 아니므로 UI 와 분리해야 한다.

동시에 다음 기능이 필요하다.

- 터치 제스처: 한 손가락 회전, 두 손가락 핀치 줌·팬, 더블탭
- 관성: 손을 뗀 뒤 회전이 서서히 멈춤
- 애니메이션: 시점 프리셋 전환, 공전 재개. 선형 외의 보간 곡선
- 커버리지 기반 시점 제한: 합성되지 않는 영역이 보이지 않는 각도까지만 허용(P4)

기준 자료로 eyeL-2100R 을 조사했으나(2026-09-28), 그곳에는 선형 보간 애니메이터(`TLBAnimator`) 하나와 한 손가락 드래그 회전만 있었다. 핀치·관성·커버리지 제한은 없다. 따라서 2100R 을 이식하지 않고, 업계에서 널리 쓰는 구조를 따라 새로 짓는다.

## 2. 계층과 배치

```
LBX_HMI_INPUT ── OnWindowHMI (UI 유무와 무관, 항상 들어옴)
   │
   ├─ ① 제스처 인식기   lbx-intf  intf/lbx_gesture.h
   │      멀티터치 id 추적 → Pan / Pinch / Tap / LongPress, 상태·속도
   │
   ├─ ② 오빗 컨트롤러   lbx-geo   geo/lbx_orbit.h
   │      Z-up 구면 상태, 원시 조작 호출, 감쇠·관성·자동 공전, 제약
   │      └─ ③ 애니메이션  lbx-core  system/lbx_anim.h
   │             트윈(이징) · 스프링 · 감쇠(fling)
   │
   └─ ④ SVM 시점 제약   lbsvm-core (P4)  오빗 제약 콜백 구현
ImGui: lbx-gui OrbitHandleInput → ② 원시 호출 어댑터 (라이다 뷰·eyel2sdk 예제 호환)
```

의존 방향은 lbx-core ← lbx-intf, lbx-core ← lbx-geo 이며 lbx-geo 는 lbx-intf 를 모른다. ② 는 입력 구조체가 아니라 원시 조작(회전량·배율·이동량·속도)만 받으므로, HMI 경로와 ImGui 경로가 같은 컨트롤러를 공유한다. ①→② 연결은 앱 쪽 몇 줄(또는 lbx-intf 의 작은 도우미)이다.

## 3. ③ lbx-core `lbx_anim` — 애니메이션

### 3.1 참조 모델

| 업계 구현 | 가져올 것 |
|---|---|
| Robert Penner easing (CSS, Unity DOTween, GSAP, Android) | 이징 곡선 이름·수식 30종 |
| CSS `cubic-bezier()` / iOS `CAMediaTimingFunction` / Android `PathInterpolator` | 사용자 정의 3차 베지어 곡선 |
| CSS `steps()` | 계단 곡선 |
| Android `ValueAnimator`, GSAP tween | 지연·반복·요요·재조준 |
| iOS `UISpringTimingParameters`, SwiftUI `spring(response:dampingFraction:)`, Android `SpringAnimation` | 스프링(속도 보존 재조준) |
| Android `FlingAnimation`, iOS `decelerationRate` | 감쇠(관성) |

### 3.2 공통 원칙

- **소유는 호출자.** 2100R 처럼 전역 싱글턴을 두지 않는다. 구조체를 호출자가 품고 `Tick(dt)` 를 부른다.
- **N 차원 float.** 1~4 차원(`f32_t[4]`)을 한 번에 보간한다. 2100R 의 벡터 성분 버그를 구조로 없앤다.
- **시간은 dt(초).** `tick_now()` 를 안에서 부르지 않는다. 재생(play) 배속·일시정지·결정적 테스트가 그대로 된다.
- **재조준(retarget).** 진행 중에 새 목표를 주면 현재 값에서 이어 간다. 스프링은 현재 속도까지 보존한다.
- **C API, 헤더 영어 주석.**

### 3.3 종류

**트윈 (`LBX_TWEEN`)** — 시간과 곡선으로 정해지는 전환. 시점 프리셋 전환, UI 페이드.

```c
typedef struct {
    f32_t from[4], to[4], value[4];
    i32_t dim;
    f32_t duration, delay, elapsed;   /* s */
    LBX_EASE ease;                    /* 곡선 */
    i32_t repeat;                     /* 0 = 한 번, -1 = 무한 */
    b8_t  yoyo, running;
} LBX_TWEEN;

void  tween_start(LBX_TWEEN*, i32_t dim, const f32_t* from, const f32_t* to, f32_t duration, LBX_EASE ease);
void  tween_retarget(LBX_TWEEN*, const f32_t* to, f32_t duration);   /* 현재 값에서 재시작 */
b8_t  tween_tick(LBX_TWEEN*, f32_t dt);                              /* 끝나면 false */
```

**이징 곡선 (`LBX_EASE`)** — 값 하나로 표현한다: 종류 + (필요 시) 매개변수.

- 기본: `LINEAR`, `SMOOTHSTEP`, `SMOOTHERSTEP`
- Penner 10계열 × In/Out/InOut = 30종: Sine, Quad, Cubic, Quart, Quint, Expo, Circ, Back, Elastic, Bounce
- `CUBIC_BEZIER(x1, y1, x2, y2)` — CSS 와 같은 정의. 표준 프리셋 `ease`, `ease-in`, `ease-out`, `ease-in-out` 을 상수로 제공(CSS 명세 값)
- `STEPS(n, jump_start|jump_end)`

`f32_t ease_eval(LBX_EASE e, f32_t t)` 를 공개해 애니메이션 밖(셰이더 파라미터, 사용자 곡선 미리보기)에서도 쓴다.

**스프링 (`LBX_SPRING`)** — 목표를 향해 물리적으로 수렴. 카메라 추종, 조작 중단 후 전환.

- 매개변수는 두 방식 모두 받는다: `stiffness`/`damping_ratio`(Android), `response`(주기, s)/`damping_fraction`(SwiftUI). 내부는 하나로 환산한다.
- 임계감쇠(ratio = 1)면 넘침 없이 가장 빠르게 붙는다 — 카메라의 기본값.
- 해석해(closed form)로 적분해 dt 가 커도 발산하지 않는다(보드 프레임 드랍 대비).
- `spring_set_target()` 은 현재 속도를 보존한다. 전환 도중 사용자가 잡아 흔들고 놓아도 튀지 않는다.

**감쇠 (`LBX_DECAY`)** — 초기 속도에서 지수적으로 멈춤. 관성.

- `v(t) = v0·e^(−t/τ)`, 위치는 적분 해석해. 매개변수는 `friction`(Android) 또는 `deceleration_rate`(iOS, 프레임당 계수 0.998 류) 중 하나.
- 멈출 지점을 미리 계산(`decay_final_value`)할 수 있다 — 제약 경계를 넘을 관성이면 경계에서 스프링으로 넘겨 튕기듯 멈추게 할 때 쓴다(iOS 스크롤 뷰 방식).

### 3.4 미루는 것

타임라인·시퀀스(GSAP `timeline`, DOTween `Sequence`)는 지금 쓸 곳이 없다. 완료 콜백 대신 `tick` 반환값으로 충분하다. 필요해지면 트윈 배열 위에 얹는다.

## 4. ① lbx-intf `lbx_gesture` — 제스처 인식

### 4.1 참조 모델

iOS `UIGestureRecognizer`(Pan, Pinch, Tap, LongPress, Rotation)와 Android `GestureDetector`·`ScaleGestureDetector`. 인식기마다 상태 Possible → Began → Changed → Ended / Cancelled 를 두고, Ended 에 속도를 싣는다.

### 4.2 입력

`LBX_HMI_INPUT` 스트림을 그대로 먹는다.

- 터치 `T0`~`T9`: id 별 포인터로 추적(2100R 은 id 를 버렸다)
- 마우스 `ML`/`MR`/`MM`, 휠 `VW`/`HW`: 마우스는 포인터 하나로 취급하고, 버튼으로 의미를 가른다(데스크톱 관례 — 왼쪽 회전, 오른쪽·가운데 팬, 휠 줌)
- `FOCS`(포커스 상실) UP: 진행 중 제스처를 Cancelled 로 끝낸다. `PLV`(포인터 이탈)는 무시한다 — Windows 는 캡처 드래그 중에도 보내므로 이것으로 취소하면 창 밖으로 끄는 드래그가 끊긴다
- 터치 id 판별은 `LBX_HMI_TouchIndex()`(lbx_msg.h). fourcc 의 숫자가 둘째 바이트라 `LBX_HMI_TOUCH_0 + n` 은 틀린 id 를 만든다 — plat-glwl 이 이렇게 만들고 있어 P2 에서 고친다

### 4.3 출력

```c
typedef enum { GESTURE_NONE, GESTURE_PAN, GESTURE_PINCH, GESTURE_TAP,
               GESTURE_DOUBLE_TAP, GESTURE_LONG_PRESS, GESTURE_WHEEL } LBX_GESTURE_KIND;
typedef enum { GESTURE_BEGAN, GESTURE_CHANGED, GESTURE_ENDED, GESTURE_CANCELLED } LBX_GESTURE_STATE;

typedef struct {
    LBX_GESTURE_KIND kind;  LBX_GESTURE_STATE state;
    i32_t pointers;          /* 1 = 한 손가락/왼쪽 버튼, 2 = 두 손가락 */
    u32_t button;            /* 마우스면 ML/MR/MM, 터치면 0 */
    f32_t x, y;              /* 초점(두 손가락이면 중점), 창 좌표 px */
    f32_t dx, dy;            /* 이번 이벤트의 이동량 px */
    f32_t scale;             /* PINCH: 직전 대비 배율, WHEEL: 노치 수 */
    f32_t vx, vy;            /* ENDED 시 속도 px/s — 최근 샘플 회귀 */
} LBX_GESTURE_EVENT;
```

- **영역·포획**: 인식기는 영역(뷰 사각형)을 받는다. 제스처는 영역 안에서만 시작하고, 시작하면 손을 뗄 때까지 영역 밖에서도 유지한다. UI 가 포인터를 잡고 있으면(`ImGui WantCaptureMouse`) 앱이 인식기에 `blocked` 를 세워 새 제스처를 시작하지 않게 한다.
- **임계**: 터치 슬롭(드래그 판정 거리, 기본 8 px — 2100R 3 px 은 떨림에 민감), 더블탭 간격 300 ms, 롱프레스 500 ms. 모두 설정값.
- **속도 추정**: 최근 100 ms 샘플의 최소자승 기울기(Android `VelocityTracker` 방식). 마지막 이벤트 두 개의 차분은 잡음이 커서 관성이 들쭉날쭉해진다.
- 한 손가락 → 두 손가락 전환 시 Pan 을 Ended 없이 Cancelled 로 끊고 Pinch 를 Began 한다(관성 오발 방지).

## 5. ② lbx-geo `lbx_orbit` — 오빗 컨트롤러

### 5.1 참조 모델

three.js `OrbitControls`. 매개변수 이름과 의미를 가능한 한 그대로 쓴다: `enableDamping`/`dampingFactor`, `autoRotate`/`autoRotateSpeed`, `minPolarAngle`/`maxPolarAngle`, `minAzimuthAngle`/`maxAzimuthAngle`, `minDistance`/`maxDistance`, 터치 매핑 `ONE = ROTATE`, `TWO = DOLLY_PAN`.

### 5.2 상태와 좌표

- **Z-up(ISO8855) 구면**: 방위 `azimuth`(z 축 둘레, +x 에서 시작, 반시계 +), 앙각 `elevation`(지면에서 위로 +), 거리 `distance`, 주시점 `target`. 지금 svmdemo 의 `TLBView::SetEyeParams(dist, yaw, pitch)` 와 같은 규약이라 변환 없이 붙는다. 현 `lbx::OrbitCamera` 의 Y-up·부호 뒤집기(`SyncOrbitToView`)는 사라진다.
- 출력: `eye`/`target`/`up` 과 view 행렬. 라이다 뷰처럼 Y-up 을 원하는 쪽을 위해 축 선택 출력을 둔다.
- 상태는 **목표 상태**와 **표시 상태** 두 벌이다. 입력은 목표를 바꾸고, 표시는 스프링으로 목표를 따른다(`enableDamping` 에 해당). 감쇠를 끄면 표시 = 목표.

### 5.3 조작 API (원시 호출)

```c
void orbit_rotate(LBX_ORBIT*, f32_t d_azimuth, f32_t d_elevation);   /* rad */
void orbit_dolly (LBX_ORBIT*, f32_t scale);                         /* 거리 배율 */
void orbit_pan   (LBX_ORBIT*, f32_t dx, f32_t dy);                  /* 화면 기준, 거리 비례 */
void orbit_release(LBX_ORBIT*, f32_t w_azimuth, f32_t w_elevation); /* 놓을 때 각속도 → 관성 */
void orbit_grab  (LBX_ORBIT*);                                      /* 관성·자동 공전·전환 즉시 중단 */
void orbit_go_to (LBX_ORBIT*, const LBX_ORBIT_POSE*, f32_t duration, LBX_EASE);  /* 프리셋 전환 */
b8_t orbit_tick  (LBX_ORBIT*, f32_t dt);                            /* 변화 있으면 true */
```

제스처 → 조작 매핑(앱/도우미 몫, 기본값): Pan 1지 = rotate(px × `rotate_speed`), Pinch = dolly(1/scale) + pan(초점 이동), Pan 2지·오른쪽 버튼 = pan, Wheel = dolly(`zoom_factor`^노치), DoubleTap = 자동 공전 토글, Pan Ended = release(속도 × `rotate_speed`).

### 5.4 동작 규칙

- **관성**: `release` 가 `LBX_DECAY` 를 건다. 각속도가 임계 아래면 걸지 않는다(살짝 놓았는데 미끄러지는 느낌 방지).
- **자동 공전**: 방위 각속도 상수(`auto_rotate_speed`, rad/s)다. 조작이 들어오면 멈추고, 조작 없이 `auto_rotate_resume` 초가 지나면 스프링으로 속도를 올려 **현재 시점에서** 이어 돈다. 지금 svmdemo 의 "공전 재개 시 궤도 맞추기" 편법이 사라진다. 2100R 의 "5 초 뒤 프리셋 복귀"도 같은 틀에서 정책 하나로 표현된다.
- **제약**: 고정 한계(three.js 의 min/max 각·거리) 위에 콜백 하나를 둔다.

```c
/* 목표 자세를 받아 허용 범위로 고친다. 커버리지 제한(P4)이 여기에 들어온다. */
typedef void (*LBX_ORBIT_CONSTRAIN)(void* user, LBX_ORBIT_POSE* pose);
```

  제약은 목표 상태에 적용한다. 관성이 경계를 넘으려 하면 경계에서 멈춘다(넘친 뒤 되돌아오는 탄성은 선택 사항).

## 6. ④ SVM 시점 제약 (P4, 별도 설계)

2100R 에 선례가 없다. 방향만 적는다. 뷰 절두체가 보울 가장자리 밖(합성 영역 밖)이나 차 밑 미합성 영역을 포함하지 않는 가장 낮은 앙각을 방위별로 구한다. 보울·카메라 배치가 바뀔 때 방위 표(예: 5° 간격)로 미리 계산하고, 제약 콜백은 표를 보간해 앙각 하한을 건다. lbsvm-core 가 투영면 정보를 가지므로 그쪽 몫이다.

## 7. 연결 (P2)

- svmdemo: `OnWindowHMI` 에서 모든 입력을 제스처 인식기에 먼저 넣고, 3D 뷰 영역에서 시작한 제스처를 3D 뷰용 오빗 인스턴스로 보낸다(영역 판정·인스턴스 선택은 앱 몫). 렌더는 그 인스턴스의 출력 자세를 `view_3d` 에 옮긴다. 그 뒤 GUI 드라이버에도 그대로 넘긴다(지금과 같다). UI 모듈은 `WantCaptureMouse` 를 `SVM_DATA` 플래그로 알리기만 한다. `svmdemo_ui.cpp` 의 `UpdateOrbitFor3DView`/`SyncOrbitToView` 는 삭제한다. Auto Orbit 체크박스·렌더의 공전 코드는 오빗 컨트롤러의 자동 공전으로 바뀐다.
- lbx-gui: `lbx::OrbitCamera`·`OrbitHandleInput`·`OrbitViewMatrix` 는 이름을 유지한 채 내부를 `lbx_orbit` 으로 바꾼다(ImGui IO → 원시 호출). 라이다 뷰와 eyel2sdk 예제는 소스 수정 없이 재빌드로 따라온다.

## 8. 단계

| 단계 | 내용 | 배포 |
|---|---|---|
| P1 | `lbx_anim`(lbx-core) · `lbx_gesture`(lbx-intf) · `lbx_orbit`(lbx-geo) 구현, 각 test 프로젝트에 단위 테스트 | 각 minor 범프, lit-ship 체인(요청 시) |
| P2 | svmdemo HMI 직결, UI 모듈의 오빗 코드 제거, lbx-gui 어댑터화 | lbx-gui minor, lbsvm-core |
| P3 | 보드 터치(S3000ABR, clink 원격)로 감도·관성·스프링 튜닝, 기본값 확정 | — |
| P4 | 커버리지 기반 앙각 하한(§6) | lbsvm-core |

## 9. 범위 결정

- 두 손가락 회전(트위스트)으로 방위를 돌리는 조작은 넣지 않는다(2026-09-28 결정). 인식기에 Rotation 종류도 두지 않는다.
- 오빗 컨트롤러는 뷰(`TLBView`)에 묶인 부품이 아니다. 조작을 받아 카메라 파라미터를 내놓는 독립 컨트롤러이고, 무엇에 연결할지는 쓰는 쪽이 정한다. 연결점(노브)은 두 가지뿐이다.
  - **입력 노브**: 원시 조작 호출(§5.3). 어떤 영역에서 온 입력을 넣을지는 호출자가 판정한다.
  - **출력 노브**: 표시 자세(방위·앙각·거리·주시점)와 거기서 파생한 eye/up·view 행렬(축 선택 가능). 매 프레임 읽어 가져간다.
- 연결 예: SVM 3D 뷰 = 출력 자세 → `TLBView::SetEyeParams` + `look.at`. 라이다 3D 표시 = view 행렬 → 자기 렌더(`TLBView` 없음). lbx-gfx 테스트앱 = view 행렬·카메라 위치. 지금 연결하는 곳은 SVM 3D 뷰와 라이다 표시다.
- 회전이 없는 표시(예: 탑뷰 팬·줌)도 방위 회전을 잠그고 앙각을 90° 로 고정한 설정이면 같은 컨트롤러로 된다.
