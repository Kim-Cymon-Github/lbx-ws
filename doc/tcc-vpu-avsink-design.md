# 텔레칩스 VPU 레코딩 — avsink-tcc 설계안

2026-09-10. 대상은 **D+(tcc803x)** 와 **D5(tcc807x)** 둘뿐이다. D3(tcc805x)는 검토 대상에서 뺀다 —
보드가 없어 실기 판정을 할 수 없고, 판정 못 하는 분기를 코드에 넣으면 죽은 경로가 하나 늘 뿐이다.
다만 805x 는 807x 와 같은 IP 조합(C7 + WAVE512 + WAVE420L)이라, 807x 가 돌면 805x 는
빌드 타깃 추가만으로 따라올 자리에 둔다. **레코딩(인코딩)이 먼저고 재생(디코딩)은 그다음이다.**

근거 조사는 [메모리: 텔레칩스 VPU 레코딩 경로](tcc-vpu-recording-paths.md) 에 요약돼 있다.

---

## 1. 무엇을 만드는가

`drv/avsink-tcc` — `LBX_AVSINK` 계약을 구현하는 새 드라이버. `drv/avsink-mpp` 의 형제다.

기존 `avsink-mpp` 는 이름과 달리 Rockchip 전용이 아니라 **`LBX_AVSINK` 구현 + MPP 백엔드**가 한
저장소에 붙어 있는 형태다. 여기에 tcc 백엔드를 `#ifdef` 로 더 넣지 않는다. 이유는 셋이다.

- MPP 와 tcc 는 **버퍼 모델이 근본적으로 다르다**(dma-buf fd vs 32비트 물리주소). 프레임 제출부터
  갈라지므로 공유할 코드가 인코더 주변에는 거의 없다.
- `avsink-mpp` 의 Linux 빌드는 이미 rk3576 하나로 `skip` 이 걸려 있다. tcc 를 넣으면 `.lit` 의
  skip 목록이 SoC별 조합으로 부풀고, 한쪽을 고칠 때마다 다른 쪽 빌드가 깨진다.
- 코덱 라이브러리 링크가 다르다(`-lrockchip_mpp` vs `-ltc_video_encoder`). 캡처 드라이버에서
  코덱 의존을 떼어낸 것과 같은 이유로, 두 코덱 백엔드도 한 `.so` 에 있을 이유가 없다.

**공유하는 것은 소스가 아니라 계약과 컨테이너 규약이다.** 세션 메타데이터, 파일명 규약(`REC_` +
`chain.json`), 봉인(seal) 동작, `rec.*` Do() 동사 집합은 `avsink-mpp` 와 동일해야 한다 — 호스트와
PC 재생 도구가 양쪽을 구분하지 않아야 하기 때문이다. 이 부분은 **소스를 이식**해서 맞춘다.
(장기적으로 공통 부분을 `avsink-core` 로 뽑을 수 있으나, 백엔드가 둘 뿐인 지금은 이르다.)

---

## 2. 현황 — 두 보드가 서 있는 자리

| | **D+ (tcc803x)** | **D5 (tcc807x)** |
|---|---|---|
| 인코더 IP | C7 (**H.264 만**) | C7 + **WAVE420L (HEVC)** |
| 디코더 IP | C7 + WAVE410 | C7 + WAVE512 4K-D2 |
| 유저스페이스 스택 | **있음** — `libtc_video_encoder` + `video_encoder.h` | **없음** |
| 실기 검증 도구 | `vpu_test` rootfs 에 존재 | 없음 |
| 툴체인 | `tcc803x-yp4.0-a53-aarch64` (aarch64 전용) | **두 벌 다 서브코어용** (§2.1) |
| 커널 | 5.10.205-tcc, vpu_v3 v3.1.6 | 6.1.85(A) / 5.10.205(B), **VPU 드라이버 off** |
| 지금 인코딩 가능 | **가능성 있음** (P0 판정) | **불가** — 코어 배치부터 정해야 함 |

### 2.1 D5 는 툴체인 선택 문제가 아니다

두 툴체인을 비교했더니 **둘 다 `MACHINE=tcc8070-sub`** 였다. `.lit` 의 `tcc807x-evm` 이라는 라벨은
오해다 — 그쪽 원본 경로가 `/opt/d5-adt`("Dolphin5 ADT")일 뿐 EVM 메인코어용이 아니다.

| | **A** `oecore-x86_64` (`.lit`: `tcc807x-1.0.0-a55-subcore`) | **B** `/mnt/temp/d5-adt` (`.lit`: `tcc807x-evm`) |
|---|---|---|
| MACHINE | `tcc8070-sub` | `tcc8070-sub` |
| Distro / 커널 / gcc | poky 5.0.4(scarthgap) / 6.1.85 / **13.3.0** | poky 4.0.15(kirkstone) / 5.10.205 / 9.2.1 |
| 빌드일 | 2025-07-31 | 2025-07-25 |
| vpu_v3 UAPI | **v3.3.14** (버전 헤더 존재) | 버전 헤더 없음 (구) |
| Mali | r48p0, `libmali.so.0` | r45p0, `libMali.so` |
| CC 기본 플래그 | 없음 | **하드닝 강제** `-fPIE -pie -D_FORTIFY_SOURCE=2 -Werror=format-security` |
| `libgbm` | 없음 | 있음 (단 `libdrm` 부재로 반쪽) |

**A 가 전 항목에서 최신이다.** B 의 강제 하드닝 플래그는 우리 라이브러리 빌드에서 예상 못 한 실패를
부를 수 있으므로, D5 표준 툴체인은 **A** 로 삼는다.

다만 그 선택은 **VPU 와 무관하다.** 양쪽 커널 모두:

```
CONFIG_TCC807X_CA55_SUB=y
# CONFIG_TELECHIPS_VPU_DRV is not set
# CONFIG_TELECHIPS_VPU_V3_DRV is not set
```

그리고 `tcc_vpu_wbuffer.h` 가 이 매크로로 고르는 pmap 프로파일
(`tcc807x-vpu-linux-ivi-subcore-customized.h`, A/B 바이트 동일)은 코덱이 전부 주석 처리돼 있다:

```c
//#define CONFIG_SUPPORT_TCC_VPU
//#define CONFIG_SUPPORT_TCC_WAVE420L_VPU_HEVC_ENC
#define INST_1ST_USE      (0)   // 디코더 0개
#define INST_ENC_1ST_USE  (0)   // 인코더 0개
```

전처리해서 값을 뽑으면 **`PMAP_SIZE_VIDEO` = `PMAP_SIZE_ENC` = `VPU_INST_MAX` = 0** 이다. 서브코어에는
VPU 물리 메모리가 한 바이트도 할당되지 않는다. 같은 헤더의 메인코어(`ivi`) 프로파일은 483MB/52MB 를
잡는다.

**즉 D5 의 문제는 "어느 SDK 로 빌드하나"가 아니라 "인코딩을 어느 코어에서 하나"다.** 카메라
(ISP/MIPI-CSI2/CAP_MEDIA)는 서브코어에 켜져 있고 VPU 는 메인코어에 있는, 전형적인 SVM 서브코어
구성이다. 우리 앱이 서브코어에 있는 한 VPU 를 직접 부를 수 없다.

선택지는 셋이고, 셋 다 **텔레칩스 확인이 선행**된다:

- **(a) 메인코어로 간다.** 메인코어용 SDK 와 pmap 프로파일(`ivi`)을 받아 앱을 그쪽에 올린다. 그러면
  카메라 캡처가 코어를 건너와야 한다 — 지금 구조가 뒤집힌다.
- **(b) 서브코어에 VPU 를 켠 커널 구성을 받는다.** pmap 을 서브코어에 나눠 주는 프로파일이 존재하는지
  텔레칩스에 물어야 한다. 헤더에는 그런 조합이 없다.
- **(c) 코어 간 IPC 로 인코딩을 요청한다.** 양쪽 커널에 `CONFIG_TCC_IPI_PROTOCOL=y`,
  `CONFIG_TELECHIPS_CAMIPC=y`, `CONFIG_TCC_IPC` 가 켜져 있다. 이 통로로 메인코어의 인코더를 쓰는
  경로가 텔레칩스 쪽에 있는지 확인이 필요하다. (추측 — 헤더만으로는 판정 불가)

표시 경로는 부수적으로 확정됐다. **양쪽 다 wayland/weston/libdrm 이 없고 EGL 네이티브 윈도가
fbdev 다.** D5 에서 우리 앱은 `plat-glfb` 계열이 맞다 — `.lit` 의 현재 skip 설정과 일치한다.

**D+ 의 HEVC 인코더는 없다.** `vpu_hevc_enc_dev.ko` 가 2.7KB 빈 모듈인 게 증거다. D+ 는 H.264 로
간다. 이건 우회할 수 없는 하드웨어 사실이고, 설계에서 코덱 선택을 런타임 caps 로 두어야 하는 이유다.

**32비트 툴체인(`tcc803x-3.1.0/3.2.0-arm32`)은 쓸 수 없다.** VPU 헤더·라이브러리가 sysroot 에 하나도
없다. `.lit` 에서 `avsink-tcc` 는 이 둘을 `skip` 한다.

---

## 3. 핵심 제약 — 인코더 입력은 32비트 물리주소다

이게 설계 전체를 규정한다.

```
libvenc_v3_interface.so  venc_encode+0x404:
    ldr x0, [x0]          ; input_y (64-bit load)
    str w1, [x0, #616]    ; pic_y_addr (32-bit store)  ← truncate
```

`VideoEncoder_Encode_Input_T.pucVideoBuffer[]` 에 넣는 값은 가상주소가 아니라 **물리주소**이고,
32비트로 잘려 저장된다. dma-buf fd 를 그대로 줄 수 없다.

반면 `LBX_AVSINK_FRAME` 은 dma-buf fd 만 나른다:

```c
typedef struct tagLBX_AVSINK_FRAME {
    void  *buffer;   i32_t fd;   u32_t size;   u32_t tick;   u32_t sequence;
} LBX_AVSINK_FRAME;
```

fd 에서 물리주소를 얻는 표준 경로는 유저스페이스에 없다. 텔레칩스가 주는 건 역방향
(`TCC_VIDEO_CREATE_DMA_BUF`: 물리주소 → fd)뿐이다.

### 3.1 그런데 물리주소는 이미 우리 손에 있다

`avio-v4l2` 가 캡처 버퍼를 잡을 때 이미 물리주소를 채워 둔다:

```c
// drv/avio-v4l2/src/avio_v4l2.cpp:585
bi->addrs_p[idxpln] = (uintptr_t)vid_buf.reserved;
```

텔레칩스 V4L2 드라이버가 `v4l2_buffer.reserved` 에 물리주소를 실어 주는 관례를 쓰고 있고,
`LBX_BUFFER_INSTANCE.addrs_p[]` 가 그걸 보관한다. 2100R 이 `dev->buffer.planes[0].pdata` 로 읽던
바로 그 값이다. **계약이 나르지 않을 뿐, 값은 존재한다.**

> 검증 필요: 위 루프는 `idxpln` 에 무관하게 같은 `vid_buf.reserved` 를 넣는다. plane 별 물리주소가
> 아니라 버퍼 단위 시작 주소일 가능성이 높다(NV12 는 단일 fd + 오프셋이므로 그래도 무방하다).
> 보드에서 실제 값을 찍어 확인한다. → P0

### 3.2 해법: 계약에 물리주소 슬롯을 tail-append 한다

`LBX_AVSINK` 를 0.2 → **0.3** 으로 올리고 `LBX_AVSINK_FRAME` 끝에 필드를 붙인다.

```c
typedef struct tagLBX_AVSINK_FRAME {
    void  *buffer;
    i32_t  fd;               /* DMA-BUF fd. -1 = 없음(물리주소만 있는 소스). */
    u32_t  size;
    u32_t  tick;
    u32_t  sequence;
    /* 0.3 tail-append */
    u64_t  phys;             /* 프레임 plane 0 의 물리주소. 0 = 모름.
                              * dma-buf fd 에서 물리주소를 얻을 표준 경로가
                              * 유저스페이스에 없어서, 물리주소를 요구하는
                              * 인코더(텔레칩스 VPU)를 위해 소스가 아는 값을
                              * 그대로 실어 보낸다. fd 를 받는 인코더는 무시한다. */
} LBX_AVSINK_FRAME;
```

`LBX_AVSINK_FMT` 에도 대응 플래그를 하나 둔다:

```c
#define LBX_AVSINK_FMT_FLAG_PHYS_CONTIG  (0x02u)  /* 이 소스의 모든 프레임이
                                                   * 물리적으로 연속이고 phys 가 유효하다. */
```

이유: 물리주소가 **유효한지**는 프레임이 아니라 소스의 성질이다. 카메라 버퍼는 항상 유효하거나
항상 무효하지, 프레임마다 바뀌지 않는다. `AddSource` 시점에 판정해서 거부할 수 있어야 한다 —
tcc 인코더는 `PHYS_CONTIG` 없는 소스를 첫 프레임이 아니라 등록에서 거절한다.

**왜 `u64_t` 인가**: 인코더가 32비트로 자르는 것은 텔레칩스 쪽 사정이다. 계약이 미리 잘라 둘 이유가
없고, 다른 SoC 가 32비트 밖 물리주소를 쓸 수 있다. 자르기 전에 백엔드가 상한을 검사한다.

**메모리 계약 규율과의 정합**: 프레임 제출은 고빈도 경로이므로 `var` 가 아니라 typed 슬롯이 맞다
(`var_t vs ABI 계약 기준`). tail-append + 양쪽 minor 범프 + 락스텝 배포는 계약 헤더가 명시한 성장
방식 그대로다.

### 3.3 물리주소가 없는 소스는 어떻게 하나

화면 녹화(`lbx_intf_export.h` 프레임 export seam)는 GL 프레임버퍼에서 오므로 물리 연속 보장이 없다.
선택지 셋:

- **(a) 지금은 지원하지 않는다.** `AddSource` 에서 `PHYS_CONTIG` 없으면 거절. 카메라 녹화만 된다.
- **(b) 바운스 버퍼.** `/dev/dma_heap/reserved` 또는 pmap 에서 물리 연속 버퍼를 잡아 CPU 복사.
  1080p NV12 한 장 3MB, 30fps 면 초당 93MB 복사 — 보드 메모리 대역폭에서 현실적이지 않다.
- **(c) VIOC WDMA 라이트백.** 2100R 이 쓴 방식. VIOC 가 합성 출력을 카브아웃 물리메모리에 NV12 로
  직접 써 넣고 그 물리주소를 인코더에 넘긴다. 제로카피이고 검증된 경로지만, 디스플레이 파이프라인에
  손을 대야 하고 GL 합성 결과와 WDMA 출력이 같은 그림이라는 전제가 필요하다.

**(a) 로 시작한다.** 화면 녹화는 지금 요구사항이 아니고, 필요해지면 (c) 를 별도 소스 드라이버로
만든다 — WDMA 는 `avsink` 의 일이 아니라 **또 하나의 소스**다. 계약이 "입력과 출력이 같은 문으로
들어온다"고 말하는 그대로다.

---

## 4. 모듈 구조

```
drv/avsink-tcc/
  src/
    avsink_main.cpp        LBX_AVSINK 계약 어댑터        ← avsink-mpp 에서 이식
    avsink_rec.h           백엔드 중립 seam              ← avsink-mpp 와 동일 유지
    avsink_rec_tcc.cpp     VideoEncoder_* 인코딩 + 세그먼트/봉인
    avsink_mux_mp4.cpp     MP4 먹서 (자체 구현)
    avsink_cast.h/.cpp     스트리밍 (P4, avsink-mpp 에서 이식)
    version.txt
  build/linux/Makefile
```

`avsink_rec.h` 는 `avsink-mpp` 의 것을 **그대로 쓴다**. dma-buf fd 대신 물리주소를 나르도록
`REC_FRAME_REF` 에 `phys` 를 tail-append 하는 것만 다르고, 나머지 계약(ref_cnt 규약, `rec_submit`
반환값 의미, 봉인 동작, `rec_do_command` 동사)은 한 글자도 바꾸지 않는다. 두 드라이버가 같은
`rec.*` 프로토콜을 말해야 호스트가 SoC 를 몰라도 된다.

### 4.1 인코더 백엔드

```c
/* avsink_enc_tcc.h — 인코더 한 채널 */
typedef struct tagENC_CHANNEL ENC_CHANNEL;

ENC_CHANNEL* enc_open(const ENC_CONFIG *cfg);   /* Create + Init + Set_Header */
const u8_t*  enc_header(ENC_CHANNEL*, u32_t *len);   /* SPS/PPS(+VPS) */
i32_t        enc_encode(ENC_CHANNEL*, u64_t phys_y, u64_t phys_uv,
                        i64_t pts_us, ENC_OUTPUT *out);
void         enc_close(ENC_CHANNEL*);
```

`video_encoder.h` 위에 얇게 덮는다. 이 한 겹을 두는 이유:

- **공개 헤더로는 못 하는 게 있다.** 런타임 비트레이트 변경과 강제 IDR 이 `venc_input_t` 에는 있으나
  `VideoEncoder_Encode_Input_T` 에는 노출돼 있지 않다. 스트리밍(P4)에서 새 시청자에게 키프레임을
  보내려면 강제 IDR 이 필요하다. 그때 `libvenc_v3_interface.so` 를 직접 부르는 우회를 이 한 겹
  **안에서만** 하면 나머지 코드는 모른다.
- **D5 는 아직 무엇을 쓸지 모른다.** 유저스페이스 스택이 없어서 `libtc_video_encoder` 를 수급하거나,
  없으면 `/dev/vpu_venc` + `VENC_V3_*` ioctl 을 직접 쳐야 한다. 이 seam 이 그 차이를 흡수한다.

### 4.2 컨테이너

**자체 MP4 먹서를 쓴다.** 텔레칩스 클로즈드 정적 라이브러리(`libTCC_ARMv8_MP4MUX_LINUX_*.a`)를
쓰지 않는 이유:

- 소스가 없어 봉인(seal)·세그먼트 경계를 우리가 통제할 수 없다. `avsink-mpp` 가 하드컷 봉인을 위해
  프래그먼트 단위로 fdatasync 하는 동작을 클로즈드 먹서에 요구할 방법이 없다.
- `flagUseOnly32BitsOffset` 전제라 4GB 상한이 걸린다.
- ARMv8/ARM32 두 벌을 저장소에 들여야 하고 SoC 마다 다시 받아야 한다.
- 인코더가 뱉는 건 **순수 Annex-B** 다. SPS/PPS → avcC 변환과 length-prefix 재작성만 하면 된다 —
  `avsink-mpp` 가 MKV 로 이미 하고 있는 일과 같은 종류다.

컨테이너를 MP4 로 할지 `avsink-mpp` 와 같은 MKV 로 할지는 **MKV 를 우선한다.** PC 재생 도구가 이미
MKV 를 읽고, `REC_` 이름 규약과 `chain.json` 이 그 위에 서 있다. SoC 가 바뀌었다고 컨테이너가
바뀌면 재생 쪽이 두 갈래가 된다.

---

## 5. 단계 계획

### P0 — 실기 판정 (코드 작성 전, 하루)

지금 설계에는 **소스가 없어 헤더와 `.ko` 심볼로만 역추정한 부분**이 있다. 코드를 쓰기 전에 보드에서
확정한다. 여기서 답이 갈리면 P1 의 방향이 바뀐다.

**D+ 보드:**
```bash
ls -l /dev/vpu* /dev/jpu*
cat /proc/modules | grep -E 'vpu|jpu'
vpu_test -e /path/in.yuv 1920 1080 -v h264 -log 2 -output
```
- `vpu_test -e` 가 돌면 → `libtc_video_encoder` 경로 확정, P1 진행.
- 안 돌면 → `vpu_drv_dev.ko` 에 `VENC_V3_*` 문자열이 없던 것이 실제 부재였다는 뜻.
  2100R 의 v1 ioctl 경로(`vpu_enc.c` + `venc_k.c` 이식)로 선회.

**avio-v4l2 물리주소 확인:** `addrs_p[]` 에 들어오는 값을 로그로 찍어, plane 별로 다른지, 0 이 아닌지,
32비트에 들어가는지 본다. `/proc/pmap` (`libtr_pmap_k.so` 가 읽는 그것)과 대조한다.

**D5 EVM:** 같은 방식으로 `/dev/vpu*` 와 모듈 목록. 특히 `vpu_hevc_enc_dev.ko` 가 D+ 처럼 빈 모듈인지
실물인지. 툴체인 두 벌 중 어느 쪽으로 부팅했는지도 함께 기록한다.

### P1 — D+ H.264 파일 녹화 (최소 경로)

계약 0.3 범프 → `avio-v4l2` 가 `phys` 를 싣는다 → `avsink-tcc` 가 1채널 H.264 로 인코딩해 MKV 로
저장. 스트리밍·다채널·봉인은 아직 없다. **끝에서 끝까지 한 줄이 돌아가는 것**이 목표다.

성공 판정: 보드에서 녹화한 파일이 PC 재생 도구에서 열리고 그림이 맞다. (stride 함정이 여기서 걸린다 —
벤더 레이어가 CbCr 을 `Y + W*H` 로 계산하므로 패딩 stride 면 화면이 밀린다.)

### P2 — 다채널 + 봉인 + `rec.*` 프로토콜 완성

`avsink-mpp` 와 동일한 동사 집합·파일명 규약·하드컷 봉인. 인스턴스 상한 확인 필요 — v1 경로 기준
`MAX_NUM_INSTANCE = 4` 였다. 8채널이 필요하면 여기서 벽에 부딪히므로 **P1 직후 caps 조회로 먼저
확인한다**(`venc_alloc_instance` 가 `max_supported_instance` 를 채워 준다).

### P3 — D5

**지금 착수할 수 없다.** §2.1 대로 서브코어에는 VPU pmap 이 0이고 커널 드라이버가 꺼져 있다.
코드 문제가 아니라 플랫폼 구성 문제이므로, 먼저 텔레칩스에 물어 (a)/(b)/(c) 중 무엇이 가능한지
답을 받아야 한다. 그 전까지 D5 는 **카메라 캡처·표시까지만** 우리 몫이다.

답이 오면 그때 갈린다. 어느 경우든 `enc_*` seam 아래만 바뀌도록 P1~P2 를 설계해 두는 것이
이 단계에 대한 준비 전부다.

- 유저스페이스 스택(`libtc_video_encoder` 상당)을 함께 받으면 → D+ 와 같은 코드가 대부분 그대로 선다.
- 못 받으면 → `/dev/vpu_venc` + `VENC_V3_*` ioctl 직접 구현. **ioctl 번호가 커널마다 다르다**
  (`GET_NEXT_RESULT_SYNC` A=217 / B=215, `vpu_drv_version_t` 등 구조체 `reserved[]` 크기도 달라
  **두 UAPI 는 바이너리 비호환**). 보드에 실제로 올라간 커널의 값을 `VPU_V3_GET_DRV_VERSION_SYNC` 로
  실측해서 맞춰야 한다.
- D5 는 HEVC(WAVE420L)가 되므로 코덱 선택을 런타임 caps 로 둔다. 최신 UAPI(A, v3.3.14)에는
  `enable_force_vpu_ip` + 독립 `vpu_hevc_enc2_dev` 가 있어 **HEVC 인코더 2 IP 동시 운용**이 가능하다.
- **32비트 물리주소 제약은 최신 SDK 에서도 그대로다.** `venc_v3_encode_in_t.pic_y_addr` 는 A/B 모두
  `unsigned int` 이고, dmabuf 필드(`enable_dma_buf_id`/`dma_buf_id`)는 **디코더 헤더에만** 있다.
  §3 의 `phys` 설계는 D5 에서도 유효하다.
- A 에만 있는 `VENC_V3_ALLOC_MEMORY_SYNC` 는 **인코더용 물리 메모리를 드라이버에서 직접 받는 경로**라,
  바운스 버퍼가 불가피해질 경우의 정공법이 된다.

### P4 — 스트리밍 (cast)

`avsink-mpp` 의 cast 를 이식. 인코더가 하나이고 파일과 소켓이 그 뒤에 둘 붙는 구조를 그대로 유지한다.
강제 IDR 이 여기서 필요해지므로 4.1 의 우회가 실제로 쓰인다.

---

## 6. 미해결 / 리스크

| 항목 | 상태 | 해소 방법 |
|---|---|---|
| D+ 에서 v3 인코더가 실제로 되는가 | **모순 있음** — `.ko` 엔 `VENC_V3_*` 문자열 0개인데 `libvenc_v3_interface` 는 `/dev/vpu_venc` 를 연다 | P0 `vpu_test -e` |
| `addrs_p[]` 가 plane 별로 맞는가 | 루프가 같은 값을 넣는다 | P0 로그 |
| 인코더 동시 인스턴스 수 | v1 기준 4, v3 는 미확인 | P1 caps 조회 |
| **D5 서브코어에 VPU 가 없다** | **확정** — pmap 0, 드라이버 off (§2.1) | 텔레칩스에 (a)/(b)/(c) 질의 |
| D5 유저스페이스 스택 수급 | 없음 | 텔레칩스에 `t-codec` 패키지 요청 |
| D5 툴체인 선택 | **A(`oecore-x86_64`) 로 확정** | — |
| 화면(GL) 녹화 | 범위 밖 | 필요해지면 WDMA 소스 드라이버 신설 |

### 재확인해야 할 함정 (조사에서 나온 것)

- **stride**: 벤더 레이어가 CbCr 주소를 `Y + W*H` 로 계산한다. NV12 vstride 1088 류 패딩을 그대로
  넣으면 화면이 밀린다.
- **`-DHAVE_ANDROID_OS`**: v1 경로로 선회할 경우 필수. 안드로이드 스위치가 아니라 "커널이 메모리를
  관리하는 모드" 플래그다.
- **`/dev/vpu_*` 늦은 생성**: 모듈 로드가 앱 시작보다 늦을 수 있다. 2100R 은 inotify 로 게이트했다.
  `avsink-tcc` 는 `AddSource` 가 아니라 **첫 `rec.start` 에서** 인코더를 열어 이 창을 자연히 피한다.
- **v2 UAPI 는 건드리지 않는다**: `vpu2_ioctl_{uapi,cmds,kern}.h` 세 파일이 같은 include guard 를 쓰고
  opcode 번호도 서로 다르다.
