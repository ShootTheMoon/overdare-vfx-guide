# overdare-vfx-guide

**OVERDARE에서 VFX를 어떻게 만드는가** — 작업 방법론. 프로젝트에 안 묶인다.

OVERDARE는 Roblox처럼 생겼고 Unreal 위에서 돈다. 그래서 **Roblox 지식이 문법은 맞는데 의미가 틀리는** 상태가 계속 생기고, 그게 이 판에서 시간을 가장 많이 잡아먹는다. 이 가이드는 그걸 피하는 순서와 도구 사용법이다.

작성 2026-09-08 · 런타임 v0.10.0 기준

---

## 두 리포의 관계

| | |
|---|---|
| **이 리포 (`overdare-vfx-guide`)** | **어떻게 하는가.** 작업 루프 · 도구 사용법 · 굽기 · 크래프트 · 체크리스트. 어느 프로젝트에나 적용된다 |
| [`onlyoneshot-vfx`](https://github.com/ShootTheMoon/onlyoneshot-vfx) | **무엇이 있는가.** 검증된 API 표면(선언문 원문) · 환각 차단 목록 · ONLY ONE SHOT의 실제 지형·UI 부착점 · 미해결 질문 · 실측 프로브 |

**API가 뭐가 있는지 궁금하면 저쪽을 봐라.** 여기서 표를 복붙하지 않는다 — 두 군데서 갈라지면 둘 다 못 믿게 된다.

자주 쓸 링크:
- [엔진 API 표면 (선언문 원문)](https://github.com/ShootTheMoon/onlyoneshot-vfx/blob/main/01_ENGINE_TRUTH.md)
- [환각 차단 목록](https://github.com/ShootTheMoon/onlyoneshot-vfx/blob/main/02_DO_NOT.md) ← **작업 전 필독**
- [프리셋·레시피 카탈로그](https://github.com/ShootTheMoon/onlyoneshot-vfx/blob/main/05_PRESET_MAP.md)

---

## 읽을 순서

| | | |
|---|---|---|
| 1 | [01_WORKFLOW.md](01_WORKFLOW.md) | 작업 루프 전체 — 어디서 시작해서 어디서 끝나나 |
| 2 | [02_TOOLING.md](02_TOOLING.md) | MCP 도구 실전 사용법 + **안전 규칙** |
| 3 | [03_ROUTES.md](03_ROUTES.md) | 이 이펙트를 어느 경로로 만들 것인가 (결정 트리) |
| 4 | [04_BAKING.md](04_BAKING.md) | 블렌더에서 스프라이트·아틀라스를 굽는 법 |
| 5 | [05_CRAFT.md](05_CRAFT.md) | 좋아 보이게 만드는 원칙 |
| 6 | [06_CHECKLIST.md](06_CHECKLIST.md) | 이펙트 하나를 처음부터 끝까지 |

급하면 **[06_CHECKLIST.md](06_CHECKLIST.md)만 읽고 시작해도 된다.** 막히는 항목에서 해당 문서로 들어가라.

---

## 이 판의 세 가지 상수

이 가이드 전체가 이 셋 위에 서 있다.

### 1. Roblox 지식은 기본값이 아니라 가설이다

`Sparkles` `Fire` `Smoke` `Explosion` `Highlight` `Decal` `ZOffset` `Debris` `UICorner` `CanvasGroup` `BindToRenderStep` `LightInfluence` — **전부 이 엔진에 없다.** 그런데 문법이 같아서 그냥 써진다. 없는 프로퍼티에 쓰면 에러가 아니라 **조용한 실패**다.

→ **`--!strict` + `overdare_validate_lua`가 이걸 기계로 잡는다.** `02_TOOLING.md` §2

### 2. 1 stud = 28 units

`1 unit = 1 cm`, 따라서 `1 stud = 28 units`. **튜토리얼에서 베낀 모든 `Size`/`Speed`/`Acceleration`/`Range`에 ×28이 필요하다.** 안 하면 안 보이거나 화면을 덮는다.

이미 cm로 적힌 값(예: *"Speed 5~12 cm/s"*)은 **환산하지 마라.** 두 번 곱하는 게 흔한 실수다.

### 3. 판정 근거는 스크린샷이지 코드가 아니다

프로퍼티 쓰기가 성공해도 화면에 안 나올 수 있다. 오프라인 렌더는 인게임을 예측하지 못한다.
**3D는 `overdare_screenshot`, UI는 `overdare_ui_browse`.** 그 외엔 아무것도 증거가 아니다.

---

## 시작하기 전 30초

```
overdare_status
```

`studioRpc` (13377) 와 `ueRemoteControl` (30010) 이 둘 다 `reachable` 이어야 한다.
`projectFile` 이 네가 작업하려는 프로젝트인지도 확인해라 — Studio에 다른 프로젝트가 열려 있으면 엉뚱한 곳을 고친다.

그리고 **쓰기 전엔 반드시 `overdare_stop`.** 이유는 `02_TOOLING.md` §1.
