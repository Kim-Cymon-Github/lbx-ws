# lit workspace (L:\)

`lit` 멀티레포 워크스페이스 루트. 각 모듈(lbx-core, lbx-intf, lbx-cal, lbx-geo, lbx-gfx,
lbx-gui, drv/*, cal/*, lbsvm-core, eyel2sdk ...)은 **독립 git 레포**이고,
빌드 멤버십과 릴리스 번들 규칙은 [.lit](.lit) 매니페스트가 정의한다.

이 레포는 **얇은 루트 레포**다 — 추적 대상은 워크스페이스 매니페스트(`.lit`)와
워크스페이스급(크로스-레포) 설계 문서(`doc/`)뿐이며, 모듈 폴더는 전부 무시한다
(화이트리스트 .gitignore). 서브모듈로 모듈을 묶지 않는다: 멤버십/조합 스냅샷은
`lit` 와 `[release.bundle]` 태그가 담당하므로 서브모듈 포인터 관리는 중복 잡일이다.

- 빌드: `lit-build` (conda env `ltk`)
- 배포: 각 모듈에서 `lit deploy` / `lit-build <module>`
- 단일 모듈 설계 문서는 해당 모듈 레포의 `doc/` 에 둔다.
  여기 `doc/` 에는 여러 레포에 걸치는 문서만 둔다.