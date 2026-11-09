# 07 WIRING — 만든 이펙트를 언제 터뜨릴 것인가

01~06은 **이펙트를 만드는 법**이다. 이 문서는 **그게 알맞은 순간에 재생되게 배선하는 법**이다.
둘은 다른 문제고, 여기가 비어 있으면 잘 구운 아틀라스가 레벨 안에 가만히 서 있게 된다.

조사일 2026-11-09 · 런타임 v0.11.0 · 출처는 전부 로컬 스킬 파일(아래 §0)

---

## 0. 이 문서의 출처

| 무엇 | 경로 (`%USERPROFILE%` 기준) |
|---|---|
| 피격 FX 필드·구조 | `.overdare\skills\tpa\references\02_Weapon_Skill_Guide.md` §Target Hit VFX |
| 풀링 런타임 동작 | `.overdare\skills\tpa\references\06_Extension_Guide.md` §Target Hit VFX |
| 트랙 종류·이름 규약 | `.overdare\skills\actionsequence\SKILL.md` |
| 컨트롤 트랙 지원 프로퍼티 | `.overdare\skills\actionsequence\references\guide.md` |
| 레벨 트리 위치 | `.overdare\skills\tpa\references\04_Level_Browser_Structure.md` |

**이건 런타임이 기본으로 깔아주는 액션게임 템플릿(`tpa`)의 배선이다.** 그 템플릿을 쓰지 않는 프로젝트라면 §1·§2는 "이런 모양으로 돼 있다"는 참고고, §3(컨트롤 트랙)과 §4(함정)는 엔진 기능이라 그대로 적용된다.

**전부 문서 기반이고 라이브 실측이 아니다.** 아래 어느 줄도 스크린샷으로 확인하지 않았다 — 이 판의 상수 3번(`README`)이 여기도 적용된다.

---

## 1. 세 가지 트리거 경로

| 경로 | 언제 터지나 | 누가 재생하나 | 코드 작업량 |
|---|---|---|---|
| **W-1** `TargetHitVFX` | **공격이 맞은 순간, 맞은 쪽에서** | 서버 `HitVFXView` (풀링) | **한 줄** |
| **W-2** 컨트롤 트랙 키프레임 | 시퀀스 타임라인의 특정 프레임 | Action Sequence 런타임 | 에디터 작업, 코드 0 |
| **W-3** `ActiveTrigger` + `ActiveHandler` | 타임라인 마커 시점에 임의 로직 | 직접 쓴 핸들러 | 핸들러 하나 |

**싼 것부터 시도해라.** 피격 반응이면 W-1 에서 끝난다. "칼이 가장 벌어진 프레임에 궤적이 떠야 한다" 같은 타이밍 요구면 W-2. 투사체·텔레포트·장판처럼 **이펙트 말고 다른 것도 같이 벌어져야** 하면 W-3.

---

## 2. W-1 — `TargetHitVFX` 한 줄

스킬 정의에 필드 하나를 더한다.

```lua
Sword_Attack1 = {
    Damage = 12,
    TargetHitVFX = "SwordSlash",
}
```

값은 **이 Part 의 이름**과 일치해야 한다:

```text
ReplicatedStorage
└─ Model
   └─ VFX
      ├─ FlashHit (Part)
      │  └─ VFXPreset
      └─ SwordSlash (Part)
         └─ ParticleEmitter
```

### Part 권장 설정

```
Anchored = true      CanCollide = false    CanQuery = false
CanTouch = false     Transparency = 1
```

그리고 **템플릿의 이펙트 자식은 `Enabled = false` 로 둔다.** 안 그러면 창고에서 계속 재생된다.

`Transparency = 1` 은 이 엔진에서 못 믿는 값이라는 기록이 있다(`onlyoneshot-vfx` Q-1). 캐리어 Part 는 크기가 작아 눈에 안 띌 가능성이 높지만, **빈 캐리어가 화면에 보이면 1순위 용의자가 이것이다.**

### 재생 규칙 — 클래스마다 다르다

| 자식 클래스 | 재생 | 해제 |
|---|---|---|
| `VFXPreset` | `Enabled = true` | `Enabled = false` |
| `ParticleEmitter` | `Enabled = true` → **잠깐 기다린 뒤** `Emit(1)` | `Enabled = false` |

**`ParticleEmitter` 의 "잠깐 기다린다"를 빼지 마라.** 공식 문서가 *"OVERDARE Studio 안정성을 위해"* 라고 명시한 대기다. 클론 직후·획득 직후에도 한 틱 기다리고, 해제 후 풀로 되돌릴 때도 한 틱 기다린다.

`HitVFXView`(`ServerStorage > Module > View`)가 Part 를 클론해 `workspace > FXRuntime` 아래에 붙이고, 끝나면 `ReplicatedStorage > Pooling` 으로 돌려보낸다. **서버 쪽 재생이다** — 클라이언트에서 같은 걸 또 만들면 두 번 터진다.

---

## 3. W-2 — 컨트롤 트랙으로 타이밍 잡기

Action Sequence 에 제어 대상을 넣어두고(`Part`, `VFXPreset` 등) 키프레임으로 프로퍼티를 찍는다.

컨트롤 트랙이 만질 수 있는 것:

| 클래스 | 프로퍼티 |
|---|---|
| `Part` | `Shape` `Transparency` `CanCollide` `CanTouch` `Size` `OriginPosition` `OriginRotation` |
| `MeshPart` | `Transparency` `CanCollide` `CanTouch` `Size` `OriginPosition` `OriginRotation` |
| `ParticleEmitter` | `Enabled` `Rate` `Emit` |
| `VFXPreset` | `Enable` |

**`VFXPreset` 은 `Enable` 하나뿐이다.** 색·크기를 타임라인에서 흔들 수 없다 — 그게 필요하면 `VFXRecipe`(경로 B)나 `ParticleEmitter`(경로 C)로 내려가야 한다. 프리셋의 "통제권" 비용이 여기서 한 번 더 나온다(`03_ROUTES.md` §1).

**`VFXPreset` 을 `Part` 자식으로 두고 부모 `Part` 를 컨트롤 트랙에 걸면 자식도 같이 제어된다.** 캐리어 Part 를 움직이면서 그 위의 프리셋을 켜고 끄는 구성이 이걸로 된다.

`ParticleEmitter` 에 `Rate` 와 `Emit` 이 둘 다 있다 — 지속 분출을 타임라인에서 조절하는 것과, 특정 프레임에 한 번 터뜨리는 것이 모두 가능하다.

### 트랙 이름은 규약이다

`SequencerController.Bind(script)` 가 **이름으로** 트랙을 찾아 묶는다. 있으면 묶이고 없으면 건너뛴다.

| 트랙 | 이름 | 역할 |
|---|---|---|
| Trigger | `Sequence` | 시퀀스 수명. 필수 |
| Trigger | `KeyInput` | 콤보 입력 접수 창 |
| Trigger | `CancelWindow` | 콤보 즉시 전환 + 취소. 꼬리에 둔다 |
| Collision | `HitTrigger` / `HitTrigger{N}` | 범위 판정. 다단히트는 여러 개 |
| Event | `ActiveTrigger` / `ActiveTrigger{1~10}` | W-3 의 마커 |

**이름을 틀리면 에러가 아니라 조용히 안 걸린다.** 이 엔진의 기본 실패 양식(`README` 상수 1)이 트랙 이름에도 적용된다.

---

## 4. 함정

**시퀀스가 끝나면 클론이 파괴되고 자식 이벤트 연결이 전부 끊어진다.** 이펙트가 시퀀스보다 오래 살아야 하면(지면 장판, 그을음, 연기 잔재) 시퀀스 밖으로 복제해야 하고, 그건 `PersistentEffectManager` 가 관리한다. **그냥 `workspace` 에 복제해두면 아무도 안 치운다.**

**CollisionTrack 콜백은 대상마다 따로 불린다** — 배열로 한 번 오지 않는다. 다섯 명이 맞으면 다섯 번 불린다. 피격 FX 를 여기서 직접 만들면 **다섯 개가 동시에 생긴다.** W-1 의 풀링을 쓰는 이유가 이것이다.

**TriggerTrack 은 "적용 → 복원" 짝이다.** 시퀀스가 교체될 때 **이전 시퀀스의 End 가 새 시퀀스의 Start 보다 먼저** 불린다. 켠 것을 End 에서 끄는 설계라면 이 순서를 믿어도 된다.

**맵 컨테이너 아래에 FX 를 두지 마라** — 맵 스왑이 있는 프로젝트에서 고아가 된다. 캐리어 Part 는 `ReplicatedStorage` 에, 실행 중인 것은 `workspace > FXRuntime` 에 둔다.

---

## 5. 체크리스트에 붙이기

`06_CHECKLIST.md` 를 다 돌려 이펙트가 레벨에 서 있게 됐다면, 그 뒤에 이걸 더한다.

1. 이 이펙트는 **무엇에 반응해서** 터져야 하나 → §1 에서 경로를 고른다
2. W-1 이면 캐리어 Part 를 `ReplicatedStorage > Model > VFX > <이름>` 에 만들고 §2 설정을 걸었나
3. 자식 클래스에 맞는 재생 규칙을 썼나 (`ParticleEmitter` 는 대기 + `Emit(1)`)
4. 템플릿 자식이 `Enabled = false` 인가
5. W-2 면 트랙 이름이 규약과 **글자까지** 같은가
6. 시퀀스보다 오래 사는 이펙트가 있나 → `PersistentEffectManager`
7. 터뜨린 뒤 fps 를 다시 쟀나 (`PerfProbe` 같은 상시 계측이 있으면 그 줄을 비교)
