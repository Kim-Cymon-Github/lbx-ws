# Benz_CLS.mastercal 렌즈 이상값 조사

- 작성: 김상구 (2026-10-01)
- 대상 파일: `Benz_CLS.mastercal`(V3 JSON + AES, 4368 B), 원본으로 알려진 `Benz_CLS.v1.mastercal`(V1, 284 B)
- 사본 위치: `L:\lbsvm-core\test\assets\`
- 상태: 원인 후보를 좁혔다. 이 세트가 어느 장비에서 만들어졌는지는 아직 확인되지 않았다.

## 1. 요약

1. V3 파일은 V1 을 변환한 결과가 아니다. 차량 제원, 카메라 자세, 렌즈 값이 모두 다르다. 캘을 다시 돌린 결과로 봐야 한다.
2. 후방·좌측 카메라의 f 가 약 0.5 이고 f.x ≈ f.y 다. 이 모양은 eyeL-2100R `lbsvm_cal.cpp` 의 `loadCamParams`/`dumpCamParams` 비대칭 결함과 정확히 맞는다. 쿼드 합성 입력(area 0.5)에서 수렴하지 못한 카메라는 f 가 절반이 된다.
3. 이 결함은 S3000ABR 이 쓰는 cal-smart·cal-flood·lbx-cal 이식본에는 들어온 적이 없다. 따라서 이 세트의 렌즈는 2100R 계열 장비(TCC803X/RK356X, 쿼드 합성 입력)를 거쳤을 가능성이 크다.
4. 우측 카메라는 이와 별개로 완전히 발산했다(f<0, k≈2e5, pitch 83°). 그런데도 저장된 것은 S3000ABR 저장 경로에 값 검증이 없기 때문이다.
5. 차량 제원(4740×1930)은 V1(4990×1890)과 다르고, 코드나 설정 어디에도 이 값이 없다. 저장 시점에 보드가 들고 있던 차량 정보가 V1 이 아니었다.

## 2. 파일 대조

V1 의 카메라 레코드는 `LoadV1CalFile` 과 같은 규칙(태그 없는 V1 레이아웃)으로 풀었다. V3 는 세트 AES 키로 복호한 JSON 이다.

### 2.1 차량 제원

| 항목 | V1 | V3 |
|---|---|---|
| length / width | 4990 / 1890 | 4740 / 1930 |
| wheelbase / rear_overhang | 2940 / 1180 | 2910 / 1000 |

V1 은 실제 CLS 제원과 맞는다. V3 값은 s3000abr 코드·설정, lbsvm-core 어디에도 없다.

### 2.2 렌즈

| 카메라 | V1 f(=scale) | V3 f.x | V3 f.y | V3 k0 | V3 f.x × k0 |
|---|---|---|---|---|---|
| 전방 | 1.032 | 1.0 | 0.961 | 458 | 458 |
| 후방 | 0.860 | 0.513 | 0.520 | 918 | 471 |
| 좌측 | 0.965 | 0.534 | 0.553 | 971 | 519 |
| 우측 | 0.983 | **-0.0101** | **-0.0074** | **228254** | — |

- V1 은 f 를 하나만 저장한다(f.x 는 1.0 으로 간주한다). 앞서 "전방 f.y 0.86" 으로 알려진 값은 실제로는 후방(두 번째 레코드) 값이다.
- f.x × k0 가 대략 일정하다. 투영 반경이 f × k(θ) 에 비례하므로, f 를 줄이고 k 를 키워도 같은 그림이 나온다(척도 모호성).
- 보드 camconfig(`Analog_FHD_C1156.camconfig.lbs`)는 네 카메라 모두 scale [1,1], k0 526.7, center (959.6, 515.9) 다. V3 렌즈는 camconfig 에서 온 값도 아니다.

### 2.3 자세 (일부)

| 카메라 | V1 y / z / pitch | V3 y / z / pitch |
|---|---|---|
| 좌측 | 1037 / 1228 / 35.2° | 925 / 991 / 50.7° |
| 우측 | -1001 / 1215 / 32.9° | -1734 / 216 / 83.4° |

## 3. 원인 후보: eyeL-2100R 캘 최적화기의 scale 처리

### 3.1 원본(2023-DH-KU-SVM)과의 차이

원본 위치는 `C:\LB\projects\2023-DH-KU-SVM\svm\src\lbsvm_cal.cpp`, 2100R 위치는 `I:\eyeL-2100R\lbsvm\lbsvm_cal.cpp` 다.

| 항목 | 2023 원본 | eyeL-2100R |
|---|---|---|
| ASPECT 로드 | `scale.y = src[i]` | `scale.y = src[i]*szfh; scale.x = src[i]*szfw;` |
| ASPECT 덤프 | `dst = scale.y` | `dst = scale.y` (szfh 로 나누지 않음) |
| center 로드 | `center = src` | `center = src*szfw/szfh` (덤프는 나누지 않음) |
| aspect 구속조건 | `penalty(scale.y, 0.7, 1.3, 100)` | 없음 |
| szf | 없음 | `#if TCC803X \|\| RK_356X` 에서 area 폭·높이. 그 외 빌드는 1.0 |

원본 설계는 f.x=1 고정, f.y(종횡비)만 미세 조정이다. 2100R 은 f.x 를 f.y 와 같은 파라미터로 묶었다.

### 3.2 `× szf` 가 틀린 이유

두 소스 모두 투영식이 area 를 이미 반영한다(`CalcLensCalibConstants`, 2100R `lbsvm\lbsvm_classes.cpp:1320`).

```cpp
uMapA.x = (area.right - area.left) / dims.x;   // area 반영
uMapA.x *= cam->scale.x;                       // scale 은 area 와 무관한 렌즈값
```

최적화기에서 area 를 또 곱하면 area 가 두 번 들어간다. 로드는 곱하고 덤프는 나누지 않으므로, 로드·덤프를 한 번 왕복할 때마다 scale 과 center 가 area 비율만큼 줄어든다. 쿼드 합성이면 0.5 배다.

### 3.3 f 가 절반이 되는 경로: `search_markers` (`lbsvm_cal.cpp:2413-2534`)

```cpp
bool is_better_score(...) {
    if (score < min_score) { DumpCamParams(backup, cam); return true; }  // 최고점 저장
    else { LoadCamParams(cam, backup); return false; }                   // 복원 → ×szf
}
...
LoadCamParams(cam, backup);   // 2528: 목표 점수 미달로 끝나면 최종 복원 → ×szf
```

- 목표 점수에 도달한 카메라는 복원 없이 끝나므로 f 가 정상이다. 전방 f=(1.0, 0.961) 이 이 경우다.
- 도달하지 못한 카메라는 최고점 backup 을 다시 로드하면서 f.x = f.y = 최고점 f.y × 0.5 가 된다. 후방(0.513)과 좌측(0.534)이 이 경우다.
- f.x 와 f.y 의 1~4% 차이는 그 뒤에 f.y 만 다루는 최적화(cal-smart/cal-flood 의 `CAL_OPT_ASPECT`)가 손댄 흔적으로 보인다.

### 3.4 이력

| 시점 | 작성자 | 커밋 | 내용 |
|---|---|---|---|
| 2021-11-22 | 최윤혁 | 0621efc6 | `svm_v` 원본. `scale.y` 만 대입한다. |
| 2022-03-30 ~ 04-08 | Henry | f9999091, b188ba29 | "player 기능 추가", "Calibration 파라미터값 area 비율 적용". `× szf` 와 `scale.x` 대입이 들어왔다. |
| 2023-01-06 | Henry | 410b3df07 | szfw/szfh 변수화 |
| 2024-07-30 | theo | 22e739fcd | "Smart Cal 업데이트(eye1100r 7/30일 버전)". 비트마스크로 다시 썼고 동작은 유지했다. 뒤바뀌어 있던 szfw↔szfh 축을 바로잡았다. |
| 2024-08-27 | Henry | e9397afed | `#if` 에 RK_356X 추가 |

aspect 페널티가 언제 빠졌는지는 아직 추적하지 않았다.

## 4. S3000ABR 쪽 확인 결과

### 4.1 변환 결함은 없다

V1 로드, V3 JSON 직렬화, 세트 저장 모두 렌즈 값을 그대로 옮긴다.

- V1 로드 경로는 `CalFloodBridge.cpp` `LoadMastercalSet_` → `load_mastercal_file(.., 1920, 1080)` → `LoadV1CalFile` 이다. 센터 픽셀 해상도는 `determine_resolution` 이 자동 판별하므로, 1920×1080 하드코딩이 값을 틀리게 만들지 않는다.
- k 역순, pitch 부호, 후범퍼 X 리베이스 모두 정상이다.
- 저장(`SaveMastercalFile`, `MastercalJson.cpp`)은 center/k/f 를 변환 없이 기록한다.

### 4.2 이식된 캘 모듈은 f.x 를 움직이지 않는다

- cal-smart·cal-flood 는 f.y 만 최적화한다(`CAL_OPT_ASPECT`). 이력 전체에 `scale.x = src` 나 `szfw` 가 들어온 적이 없다(lbx-cal 포함).
- cal-smart 는 08-31(fa205de) 이후 매 Run 마다 렌즈를 camconfig 로 덮어쓴다(`smart_impl.cpp:334`). 보드 camconfig 는 scale [1,1] 이다.
- 따라서 현재 S3000ABR 코드만으로는 f.x=0.513 이 나올 수 없다.

### 4.3 저장 단계에 방어가 없다

- 실패 카메라를 Run 시작 상태로 되돌리는 동작이 꺼져 있다(`CalFloodBridge.cpp:2146-2160`, 09-07). 주석에 "Apply 의 FAIL 게이트는 별도 결정(현재 없음)" 이라고 적혀 있다.
- `SaveMastercalFile`·`ApplySave`·`ExportUsbSet` 에는 f/k/pitch 범위 검사와 failed_mask 확인이 없다.
- 로드할 때 쓰는 `MastercalSetProblem_` 은 NaN/Inf 와 제원 양수만 본다. f<0 이나 k=2e5 는 통과한다.
- 모듈의 발산 가드 `ga_cam_diverged`(cal-smart `smart_optimize.cpp:727`)는 f 를 보지 않고 k 크기에도 상한이 없다.
- 저장은 자동이 아니다. 사용자가 Apply 나 Save to USB 를 눌러야 한다.

### 4.4 차량 제원이 V1 이 아니게 되는 경로

- USB 없이 Load Master 를 누르면 persist(`/home/root/lbsvm.cal`)가 먼저 실린다. 차량 교체는 `from_usb` 일 때만 일어난다(`CalFloodBridge.cpp:4005-4070`).
- 부팅할 때 persist 의 이전 세트가 V1 보다 먼저 적용된다(`svm_bootstrap.cpp:389-430`).
- `FindMastercalModel` 은 디렉터리에서 처음 나온 `*.mastercal` 을 고른다. USB 에 다른 차종 세트가 섞여 있으면 그 세트가 적용된다.
- V1 이 USB 로 정상 적용됐다면 보드 로그에 `vehicle spec replaced - USB mastercal payload (L=4990…)` 가 남아야 한다.

## 5. 확인이 필요한 것

1. 이 세트가 거쳐 온 장비. 2100R 계열(TCC803X/RK356X, 쿼드 합성 입력)에서 캘한 적이 있는지.
2. 세트를 만든 날짜와 그때의 앱·캘 모듈 버전.
3. 보드 로그. Load Master 의 source 문자열과 4.4 의 차량 교체 로그 유무.
4. 해당 보드의 persist 캘. f≈0.5 가 들어 있으면 3 장의 경로가 확정된다.

## 6. 수정 제안

### 6.1 eyeL-2100R `lbsvm_cal.cpp`

- `loadCamParams` 에서 ASPECT·CX·CY 의 `× szf` 를 뺀다(투영식이 area 를 처리한다).
- ASPECT 는 `scale.y` 만 대입한다. `scale.x` 대입을 없앤다.
- cost 에 aspect 구속조건(`scale.y` 0.7~1.3)을 복원한다.

### 6.2 S3000ABR 저장 게이트

- Apply / Save to USB 직전에 카메라별로 검사한다: f>0, f.x≈1, f.y 0.7~1.3, k0 범위, pitch·z 상식 범위, failed_mask.
- 실패 카메라가 있으면 저장을 막거나 사용자 확인을 받는다.
- `MastercalSetProblem_` 에도 같은 범위 검사를 넣어, 이런 세트를 로드할 때 거부한다.

### 6.3 캘 모듈

- `ga_cam_diverged` 에 f 부호·범위와 k 크기 상한을 넣는다.

## 7. 부록: 이번 조사에서 바꾼 것 (lbsvm-core, 미커밋)

- `LoadCalFileAuto()` 를 추가했다(`src/lbsvm_vehicle.h`, `src/lbsvm_mastercal_json.cpp`).
  - 내용을 보고 V3 평문 JSON / V3+AES / V1·V2 바이너리를 가린다.
  - JSON 으로 판별된 파일은 실패해도 V1 파서로 넘기지 않는다.
  - 한 바이트로 꽉 찬 파일(매체 읽기 실패)은 거부한다.
  - AES 키는 호출자가 넘긴다. 라이브러리에는 키를 넣지 않는다.
- `NormalizeLensFocal()` 을 추가했다. svmdemo Lens 패널의 `Norm` 버튼과 같은 변환이다(f.y /= f.x, k *= f.x, f.x = 1). 투영이 r(θ)·f 꼴이라 상은 바뀌지 않는다(테스트로 확인). `LoadCalFileAuto()` 가 로딩 직후 모든 카메라에 적용하고, `Norm` 버튼도 이 함수를 부른다. f.x≤0(발산)은 고칠 수 없으므로 경고만 남기고 그대로 둔다.
  - Benz_CLS 적용 결과: 후방 f=(1, 1.014) k0=470.6, 좌측 f=(1, 1.036) k0=518.6. camconfig(k0 526.7)·Analog 계열 실측(476~486)과 같은 범위로 돌아온다. 우측(f.x=-0.0101)은 발산이라 그대로 남는다.
- svmdemo `LoadCalFile` 이 이 함수를 쓰도록 바꿨다. 이제 위 V3+AES 세트를 PC svmdemo 에서 바로 열어 볼 수 있다(로그 `kind=0x103 ret=0`).
- `test-mastercal-codec` 에 판별 케이스 6종을 추가했고 전부 통과했다.
