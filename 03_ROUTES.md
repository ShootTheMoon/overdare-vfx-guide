# 03 ROUTES — 어느 경로로 만들 것인가

상세 결정 트리와 경로별 프로퍼티 표는 `onlyoneshot-vfx`에 있다:
[월드 FX 6경로](https://github.com/ShootTheMoon/onlyoneshot-vfx/blob/main/03_ROUTES_WORLD.md) · [UI FX 5경로](https://github.com/ShootTheMoon/onlyoneshot-vfx/blob/main/04_ROUTES_UI.md)

**여기서는 "어떻게 고를 것인가"만 다룬다.**

---

## 1. 경로 한눈에

### 3D 월드

| | 경로 | 커스텀 아트 | 통제권 | 비용 |
|---|---|---|---|---|
| A | `VFXPreset` — 기성 146개 | ✗ | `Color`/`Size`/`Transparency`만 | **최저** |
| B | `VFXRecipe` — Niagara 39소스 3레이어 | 간접 | 소스 교체·색·강도·딜레이 | 낮음 |
| C | `ParticleEmitter` + 우리 아틀라스 | ✓ | NumberSequence 커브 전부 | 중 (굽기 필요) |
| D | `Beam` / `Trail` | ✓ | 폭·색·**UV 스크롤** | 중 |
| E | `EmptySprite` (B 하위) | ✓ | 임의 행×열 플립북, 비정사각 | 중 |
| F | 절차적 `Part` | — | 무한 | **높음 — 파츠 수가 곧 fps** |

### 2D UI

| | 경로 | 업로드 |
|---|---|---|
| U-a | `ImageLabel` + 트윈 | 1 |
| U-b | `ClipsDescendants` 창 + 아틀라스 | **1** |
| U-c | 프레임 시퀀스 (`Visible` 토글) | **N** |
| U-d | 절차적 `Frame` | 0 |
| U-e | `BillboardGui` (월드 공간 2D) | 1~N |

---

## 2. 고르는 순서 — 싼 것부터

```
1. 프리셋에 있나?           →  grep presets.md.  있으면 끝.
2. 레시피로 조합되나?        →  템플릿 7종 중 비슷한 걸 복사, 색·강도만
3. 선/궤적/리본인가?         →  Beam / Trail  (UV 스크롤은 여기만 된다)
4. 우리만의 그림이 필요한가?  →  아틀라스를 굽는다
5. 파티클로 안 되는 기하인가? →  절차적 Part
```

**F(절차적 Part)로 바로 가지 마라.** 가장 자유롭지만 가장 비싸고, 이 프로젝트가 지금까지 유일하게 써온 방법이라 익숙해서 손이 먼저 간다.

---

## 3. 판단이 갈릴 때의 기준

### "프리셋이 있는데 톤이 안 맞는다"

먼저 `VFXPreset`의 **`Color` / `Size` / `Transparency` 셋으로 눌러봐라.** 채도를 낮추고 크기를 줄이고 투명도를 올리면 상당수가 쓸 만해진다.
그래도 안 되면 **B(레시피)로 한 칸 내려가서** 비슷한 템플릿의 Color/Alpha 키포인트만 갈고 `SpawnCount`를 낮춘다.

**공짜라는 이유로 톤이 안 맞는 프리셋을 쓰지 마라.** 카탈로그 146개는 `Adventure`/`Cartoon`/`Fantasy`/`Sci-Fi` 태그가 붙은 남의 게임 톤이다.

### "아틀라스를 구울 가치가 있나"

**프레임당 픽셀부터 계산해라.**

| 512 아틀라스 | 프레임 | 프레임당 |
|---|---|---|
| `Grid2x2` | 4 | 256 px |
| **`Grid4x4`** | **16** | **128 px** ← 기본값 |
| `Grid8x8` | 64 | 64 px |

**128 px에서 안 읽히는 그림은 굽는 의미가 없다.** 가는 선, 잔글씨, 섬세한 실루엣은 여기서 죽는다.
프레임을 줄이면(4x4 → 2x2) 해상도를 벌 수 있고, 체감 속도는 `FlipbookFramerate`와 `Lifetime`으로 맞출 수 있다. **하지만 해상도는 만들어낼 수 없다.**

### "UI에서 프레임 애니메이션이 필요하다"

`ImageRectOffset`이 **없다.** 그래서 스프라이트시트를 프로퍼티로 재생할 수 없고, `ScaleType`도 `Stretch`/`Slice`뿐이라 `Crop` 트릭도 안 된다.

**우회: `ClipsDescendants` 부모(창) + 아틀라스 전체 크기 자식(필름), 자식의 `Position`을 셀 단위로 옮긴다.** 업로드 1장으로 끝나고 3D 아틀라스와 자산을 공유할 수 있다. [상세](https://github.com/ShootTheMoon/onlyoneshot-vfx/blob/main/04_ROUTES_UI.md#2--imagerectoffset--imagerectsize는-없다)

안 되면 프레임을 6장 이하로 줄이고 **사이를 트윈으로 메워라.** N장 업로드는 이펙트 종류마다 곱해져서 금방 감당이 안 된다.

---

## 4. 죽는 기법 — 판단 대상이 아니다

전부 커스텀 셰이더가 필요하고 **이 엔진엔 셰이더도 머티리얼 그래프도 없다.**

모션벡터 플립북 · 소프트/뎁스 페이드 파티클 · 버텍스 애니메이션 텍스처(VAT) · 런타임 디졸브 · SDF 왜곡 · 열 아지랑이 · 6-way lighting 연기 · **포스트 프로세싱 전부**(Bloom/DOF/ColorCorrection/Blur/SunRays) · 표면 UV 스크롤(`Decal`/`Texture` 클래스 부재)

**대신:**
- 프레임 보간이 필요하면 → 프레임 수를 늘리거나 `FlipbookFramerate`를 낮춰 끊김을 감춘다
- 디졸브·왜곡이 필요하면 → **플립북으로 미리 구워라**
- 깊이 페이드가 필요하면 → 스프라이트 하단을 지면에 살짝 묻어 교차선을 숨긴다
- UV 스크롤이 필요하면 → **`Beam`/`Trail`의 `TextureSpeed`가 유일한 수단이다**
- 블룸이 필요하면 → **글로우를 텍스처에 구워 넣어라** (아래)

---

## 5. 포스트FX 없이 빛나 보이게

전역 손잡이는 `Lighting.Brightness` / `Contrast` / `Saturation` 셋뿐이다. 발광은 소재 레벨에서 만든다:

| 수단 | 비고 |
|---|---|
| `Material.Neon` | 검증된 유일한 발광. 기존 코드가 전부 이걸 쓴다 |
| `Material.Unlit` | 조명 무시 |
| `MaterialVariant.Emissive` + `EmissiveIntensity` + `EmissiveMap` | **세기 조절이 되는 유일한 발광.** Roblox엔 없는 기능 |
| `ParticleEmitter.LightEmission` 0→1 | 0 = 알파블렌드, 1 = **가산합성**. 회색조 시트를 가산으로 쓰면 알파 채널 없이 빛난다 |
| **글로우를 텍스처에 굽기** | 블룸이 없으니 이게 제일 확실하다 |
| `PointLight` | 비싸다. 히어로 순간에만 |

⚠️ **`ZOffset`이 없어서 불을 연기 앞에 강제로 둘 수 없다.** 겹침 순서에 의존하는 설계를 피하고, 필요하면 이미터를 카메라 쪽으로 물리적으로 오프셋해라.
