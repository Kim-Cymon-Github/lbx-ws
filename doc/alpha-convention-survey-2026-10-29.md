# 알파 형식 조사 — 아틀라스 PNG 를 쓰는 곳 전부 (2026-10-29)

| 판 | 날짜 | 내용 |
|---|---|---|
| 초안 1 | 2026-10-29 | lbx-gl·lbx-gfx·lbx-gui·lbsvm-core·eyel2sdk·S3000ABR 앱의 텍스처 올리기·필터·블렌딩 조사, 약속 제안 |

## 0. 요약

- ltxm 이미지 뷰어에서 도형 겉에 검은 테두리가 보인 것은 편집기만의 문제가 아니다. 아틀라스를 straight alpha 그대로
  올리고 `SRC_ALPHA` 블렌딩 + LINEAR(밉맵 포함)로 그리는 곳은 모두 같은 테두리가 생긴다.
- 원인: 텍셀에 색이 들어 있는 그림(car.png 의 차·가이드, S3000ABR 가이드·UI 아틀라스)에서 투명 텍셀의 숨은 RGB 가
  검정이라 필터가 경계에서 그 검정을 섞는다. ImGui 글꼴은 모든 텍셀 RGB 가 흰색이고 색은 정점 색으로 넣으므로 이 문제가
  없다.
- 반대쪽 문제도 있다. `ONE` 블렌딩(premultiplied)으로 그리는 곳은 아틀라스가 premultiplied 라고 가정하는데, 이것을 보장하는
  코드가 없다. 주석("투명 텍셀은 검정")만 있다. car.png 의 차 그림에는 반투명 텍셀이 49,910 개 있고 그중 1,291 개는
  RGB > A 라 straight alpha 가 확실하다. 이 부분을 `ONE` 으로 그리면 가장자리가 밝게 뜬다. 가이드라인 행(y ≥ 480)은 반투명
  텍셀이 0 개라 지금은 문제가 드러나지 않는다.
- **텍스처를 올리는 어느 경로도 premultiply 하지 않는다.** lbx-core PNG 디코드, lbx-gl `TGLTexture2D::SetImage`,
  lbx-gfx `from_pixels`·ExtImage, S3000ABR lbgui `SetImage` 모두 straight 그대로 올린다.
- 조사한 저장소(L:\ 의 lbx-gl·lbx-gfx·lbx-gui·lbsvm-core·eyel2sdk, I:\s3000abr 앱 소스)의 main 작업 트리에는 블렌딩 관련
  미커밋 변경이 없다. 지금 고치는 중인 블렌딩은 이 밖(S3000ABR 의 subprojects: lbgui·lbsvm-app 등 별도 저장소)에 있을 것이다.

## 1. 제안하는 약속

1. **PNG 는 규격대로 straight alpha 로 본다.** 투명 텍셀의 RGB 는 의미가 없다(그리는 도구가 아무 값이나 쓴다).
2. **GPU 에 올릴 때 한 번 premultiply 한다**(RGB × A). 숨은 RGB 가 0 이 되어 LINEAR·밉맵·축소 어디서도 섞이지 않는다.
   가이드라인처럼 알파가 0 과 255 뿐인 영역은 premultiply 해도 텍셀이 하나도 바뀌지 않는다.
3. **그리기는 `ONE, ONE_MINUS_SRC_ALPHA`** 로 한다. 틴트(정점 색·`uColorScale`·gfx `color_scale`)도 네 채널을 함께
   스케일한다. 알파만 줄이면 페이드가 아니라 밝게 뜬다.
4. CPU 사본(ltxm·svmdemo uvmap 의 Save, 픽셀 값 표시)은 straight 그대로 둔다. premultiply 는 올리기용 별도 버퍼에서 한다.
   제자리에서 곱하면 Save 가 손실된 PNG 를 쓴다.
5. 8 비트 premultiply 는 알파가 아주 작은 텍셀의 색 정밀도를 잃는다. 화면에서 거의 보이지 않으므로 받아들인다.

## 2. 조사 결과 — 묶음별

표의 줄 번호는 2026-10-29 main 기준이다. 모두 커밋된 코드다.

### 2-1. 텍스처 올리기 (premultiply 를 넣을 자리)

| 위치 | 하는 일 | premultiply | 필터·밉맵 |
|---|---|---|---|
| lbx-gl `src/gl/lbx_gl_class.cpp:283-311` `TGLTexture2D::SetImage` | LoadTextureMap 아틀라스, 차량 부품 텍스처 모두 여기로 | 없음 | LINEAR / LINEAR_MIPMAP_LINEAR + `glGenerateMipmap` |
| lbsvm-core `src/lbsvm_model_2d.cpp:29-40` `LoadTextureMap` | car.png 아틀라스, CPU 사본 유지(KeepImageData) | 없음 | 위와 같음 |
| lbx-gfx `src/gfx/lbx_gfx_texture_gles.c:179-227` `gfx_texture_2d_from_pixels` | gfx 2D 텍스처 | 없음 | 호출자 지정(밉맵 없음) |
| lbx-gfx `lbx_gfx_texture_vk.c:315-436` | VK 판 | 없음 | min 필터 무시, 밉맵 없음 |
| lbx-gfx `lbx_gfx_external_gles.c:879-977`, `lbx_gfx_external_vk.c:504-545` | ExtImage CPU 경로 | 없음 | LINEAR |
| lbx-gui `imgui_impl_opengl3.cpp:682-740` | ImGui 글꼴·동적 아틀라스(흰색 RGB) | 해당 없음 | LINEAR |
| S3000ABR `subprojects/lbgui/.../lbgui_texture.cpp:589-607` | UI 아틀라스(stbi) | 없음 | LINEAR_MIPMAP_LINEAR + 밉맵 |
| S3000ABR `src/systems/GuidelineRenderSystem.cpp:322-344` | `guide-s3000.png` 가이드 아틀라스 | 없음 | LINEAR(밉맵 미사용) |

이미지 플래그 선례: lbx-gfx 는 `LBX_IMAGE_FLAG_ALPHA_OPAQUE` 를 보고 알파를 1 로 스위즐한다. 같은 방식으로
`PREMULTIPLY` 요청 플래그를 둘 수 있다.

### 2-2. 어두운 테두리가 생기는 곳 (straight 데이터 + `SRC_ALPHA` + LINEAR)

| 위치 | 그리는 것 | 비고 |
|---|---|---|
| lbsvm-core `test/src/svmdemo_core.cpp:4536-4565` `Render2DCarModel`, eyel2sdk `examples/src/svmdemo_core.cpp:1411-1440` | 2D 차 그림(car.png) | 블렌딩 함수를 정하지 않고 앞 단계 것(`SRC_ALPHA`)을 물려받는다. 밉맵까지 있어 축소 시 테두리가 넓다 |
| lbsvm-core `src/lbsvm_dpgl.cpp:257-385` `DrawDPGL`(구 경로) | 단일 카메라 뷰 PGL·DPGL | `SRC_ALPHA`(:314). `ONE` 은 "무조건 진하게 나옴" 주석과 함께 꺼져 있다(:313) — 틴트를 네 채널로 곱하지 않아서였을 것이다 |
| lbsvm-core `src/lbsvm_render.cpp:1407-1419` `SetVehiclePassState` | 차량 부품 텍스처 | 투명 구멍·데칼이 있는 텍스처일 때만 |
| lbx-gfx BLIT·PBR 를 `BLEND` 로 쓰는 곳 | gfx 로 그리는 PNG | `BLEND` = straight |
| lbx-gui ImGui 백엔드 `imgui_impl_opengl3.cpp:365` + 이미지 뷰어 | ltxm·svmdemo uvmap 아틀라스 보기 | 이번에 본 테두리 |
| S3000ABR `GuidelineRenderSystem.cpp:810-811` | 가이드 아틀라스 전체(띠·DPGL·앞 가이드·바퀴 궤적·PAS) | UV 가 스프라이트 텍셀 가장자리에 정확히 놓여(:516, :601, :715) 필터가 스프라이트 밖 이웃 텍셀을 반쯤 섞는다. 틈은 약 2 px |
| S3000ABR lbgui UI 아틀라스(ImGui 로 그림) | 모든 HMI 스프라이트·9-slice | 밉맵으로 축소 시 더 넓게 |

### 2-3. 밝게 뜨는 곳 (`ONE` 블렌딩인데 데이터가 premultiplied 라는 보장이 없음)

| 위치 | 그리는 것 | 비고 |
|---|---|---|
| lbsvm-core `src/lbsvm_guide_map.cpp:233-276` gfx 가이드 맵 굽기 | DPGL 띠 → 맵 FBO | BLIT `PREMULTIPLIED`, 아틀라스는 RAW car 텍스처. 주석(:234-236)이 "투명 텍셀 RGB 0" 을 가정 |
| lbsvm-core `test/src/svmdemo_core.cpp:4580-4718` raw GL 굽기 | 같은 일(옛 경로) | `glBlendFunc(ONE, 1-SA)` :4700 |
| 위 맵을 쓰는 데칼 전부 | `lbsvm_guide_map.cpp:311`, `svmdemo_core.cpp:4902·4986·5186·5488·5609` | 맵이 premultiplied 이면 맞다. 굽는 단계가 문제 |
| lbx-gfx GROUND_DECAL `lbx_gfx_material.h:68-71·283·294·310` | 지면 데칼 | 시험 앱만 CPU 에서 premultiply(test/src/app.cpp:541-556) |
| S3000ABR `subprojects/lbgui/src/lbgui_core.cpp:215` | 토스트 페이드 | `ONE` 블렌딩에 알파만 줄여 흐려지지 않고 밝게 뜬다 |

### 2-4. 이미 맞는 곳 (premultiplied 로 만들어 지는 FBO)

- 고스트 FBO(`lbsvm_render.cpp:1606-1689`)와 합성(`:1693-1745`, `uColorScale=(a,a,a,a)`, `ONE`).
- TP 누적(`lbsvm_tptopview.cpp:320-369`, `blit_tpfade` + `ONE`).
- ImGui 글꼴(흰색 RGB).

### 2-5. 함께 발견한 결함·위험

| 위치 | 내용 |
|---|---|
| lbsvm-core `src/lbsvm_tptopview.cpp:241-300` `TLBTransparentTopview::Draw` | premultiplied FBO 를 화면에 그리면서 블렌딩 함수를 정하지 않는다. 앞 단계가 `SRC_ALPHA` 면 반투명 영역에 알파가 두 번 곱해진다 |
| lbsvm-core `svmdemo_ui.cpp:1946`, S3000ABR `src/debug/TpFboDebugPanel.cpp:92` | premultiplied FBO 를 ImGui(straight)로 보여 가장자리가 어둡다(디버그 화면) |
| 블렌딩 함수를 설정하고 되돌리지 않는 곳 | lbsvm-core `TLBView::Draw`(`lbsvm_view.cpp:1573`), S3000ABR `GuidelineRenderSystem`(:1037)·`ObjectDetectionRenderSystem`(:1485·1674·1778)·`AllROIManagerSystem`(:1070·2835). `GL_BLEND` 만 되돌린다. 뒤 단계는 블렌딩 함수를 반드시 직접 정해야 한다 |
| lbx-gfx `lbx_gfx_device_gles.c:403` | 렌더 패스 시작 때 블렌딩 상태 캐시를 GL 에서 다시 읽지 않고 그대로 믿는다. 패스 사이 raw `glBlendFunc` 가 있으면 gfx 가 필요한 설정을 건너뛸 수 있다(코드 읽기로 본 위험, 재현은 안 함) |
| lbx-gfx GLES `BLEND`(accum_alpha 아님) | `glBlendFunc(SA, 1-SA)` 라 FBO 에 굽기 할 때 대상 알파가 SA·SA 로 쌓인다. VK 는 알파를 `ONE, 1-SA` 로 맞게 한다 |
| lbx-gfx `GFX_ALPHA_MODE_MASK` | `alpha_cutoff` 를 읽는 곳이 없어 OPAQUE 와 같다(미구현) |
| lbsvm-core `GuardKeyEdges` | 반 텍셀 가드는 밉맵 0 단계만 지킨다. 아틀라스 이웃 영역은 굵은 밉맵 단계에서 여전히 섞인다(알파 형식과 별개) |
| lbsvm-core svmdemo uvmap `TextureFilterCallback`(:8784-8806) | 뷰어를 한 번 열면 그 세션 동안 아틀라스 MIN 필터가 밉맵 없는 NEAREST/LINEAR 로 바뀐다. 뷰어를 열었는지에 따라 축소 품질이 달라진다 |
| lbsvm-core `doc/TpTopview_Handoff_2026-09-18.md:28-30` | DrawFadeComposite 블렌딩 설명이 옛 값(`SRC_ALPHA`). 지금 코드는 `ONE` |

## 3. 고치는 순서 제안

1. **올리기 단계에 premultiply 선택지를 둔다.** 기본은 끈다(지금 동작 그대로).
   - lbx-gl `TGLTexture2D::SetImage`: premultiply 플래그. CPU 사본은 그대로 두고 올리기용 버퍼에서만 곱한다.
   - lbx-gfx `from_pixels`·ExtImage: `LBX_IMAGE_FLAG_ALPHA_OPAQUE` 와 같은 방식의 이미지 플래그.
2. **아틀라스(LoadTextureMap) 경로에서 켜고 소비처를 맞춘다.**
   - `Render2DCarModel`(lbsvm-core·eyel2sdk): `ONE, 1-SA` 를 직접 정한다.
   - `DrawDPGL`: `ONE` + 틴트 네 채널 곱하기.
   - 가이드 맵 굽기 두 곳: 이미 `ONE` 이라 그대로 맞게 된다(반투명 텍셀까지).
   - `TLBTransparentTopview::Draw`: `ONE` 을 직접 정한다.
3. **lbx-gui: ImGui 에 premultiplied 블렌딩 콜백.** 샘플러 흉내와 같은 방식(플래그를 두고 `SetupRenderState` 에서
   읽음, 프레임 시작·`ResetRenderState` 에서 끔, VK 는 두 번째 파이프라인). 이미지 뷰어에 premultiplied 선택지.
   ltxm·svmdemo uvmap 뷰어와 TP FBO 디버그 화면이 쓴다. 틴트는 호출자가 네 채널로 곱한다.
4. **S3000ABR.** 가이드 아틀라스(`GuidelineRenderSystem`: 올릴 때 premultiply + `ONE` + `uColor` 네 채널)와 lbgui UI
   아틀라스(올릴 때 premultiply + 3 의 ImGui 콜백 + `layout(alpha)` 페이드 틴트). lbgui 는 별도 저장소라 담당자와 맞춘다.
5. **차량 부품 텍스처는 나중에 따로.** `SetImage` 를 같이 쓰지만 플래그를 끈 채 두면 영향이 없다. 바꿀 때는
   `SetVehiclePassState` 를 `ONE` 으로, 부품 셰이더에서 알파 스케일을 RGB 에도 곱한다.

1·2 만으로 car.png 를 쓰는 SVM 쪽(2D 차, 구 DPGL, 가이드 맵)은 테두리와 밝은 가장자리가 모두 없어진다. 3 은 편집기
화면, 4 는 S3000ABR 제품 화면이다.

## 4. 적용 (2026-10-29)

| 모듈 | 판 | 내용 |
|---|---|---|
| lbx-gl | 0.5.2 b6 | `TLBTexture::SetPremultiplyAlpha` — 켜면 RGBA·AB24·YA 를 올리기용 사본에서만 rgb×a 로 곱해 올림(밉맵도 곱한 데이터로). CPU 사본은 straight. 기본 꺼짐, 쓰지 않던 flags 비트라 클래스 배치 불변 |
| lbx-gui | 0.6.1 b4 | GL 백엔드 `DrawCallback_SetBlendPremultiplied`(다음 `ResetRenderState` 까지 `ONE, 1-SA`, 셰이더 변종 전환에도 유지). 코어 `lbxImGuiPush/PopPremultipliedBlend`·`lbxImGuiPremultiplyColor`·`lbxImGuiRenderImageInViewerEx(…, LBX_IMGUI_IMAGE_PREMULTIPLIED)`. `ImageViewerState`·기존 함수 불변. VK 백엔드는 미지원(straight 로 그림) |
| lbsvm-core | 0.9.3 | `DrawDPGL` `ONE` + 틴트 네 채널 곱(예전 "무조건 진하게"의 원인이 틴트였다). svmdemo: car 아틀라스 premultiplied, `Render2DCarModel` `ONE` + 이전 블렌딩 복원, uvmap·TP FBO 디버그 뷰어 Ex. 가이드 맵 계약 주석 |
| eyel2sdk | 0.5.2 | 예제 svmdemo: car 아틀라스 premultiplied, `Render2DCarModel` `ONE` + 복원 |
| ltxm | — | car 텍스처 premultiplied, 이미지 뷰어 Ex, 래티스 미리보기 사본을 알파 번짐 대신 premultiplied 로, `_draw_lattice` 를 Push/Pop 으로 감쌈 |

- TpTopview 그리기·누적 코드는 바꾸지 않았다. 블렌딩을 바꾸는 곳은 모두 이전 함수를 저장했다가 되돌린다.
- 확인: svmdemo(p2_dual)에서 가이드라인·2D 차, ltxm 이미지 뷰어(축소·확대)와 래티스 Preview 에서 가이드라인 L 모양의 경계에
  검은 테두리가 없다.
- car.png Truck 차체 바깥의 밝은 분홍 계단 줄은 그림에 불투명(A:255)으로 칠해진 텍셀이다. 원본에서 다시 내보내거나
  편집기에 테두리 정리(defringe) 기능을 두어야 한다.
- 남은 일: S3000ABR(가이드 아틀라스 `GuidelineRenderSystem`, lbgui UI 아틀라스·토스트), lbx-gfx `from_pixels`·ExtImage
  premultiply 선택지, VK ImGui 백엔드 콜백, 차량 부품 텍스처.
