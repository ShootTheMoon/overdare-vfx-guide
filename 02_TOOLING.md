# 02 TOOLING — 도구 실전 사용법

MCP 서버: [`Seungpyo1007/overdare-mcp`](https://github.com/Seungpyo1007/overdare-mcp) — `node dist/index.js`, 도구 50개.
로컬 클론이 원격보다 뒤처지면 도구가 통째로 없다. 의심되면 `git log`를 확인하고, `npm run build` 뒤 **MCP 호스트를 재시작해라** — 도구 목록은 세션 시작 때 한 번만 읽힌다.

---

## 1. ★ 안전 규칙 — 먼저 읽어라

어기면 사람이 손으로 수습해야 한다.

### 1-1. 쓰기 전엔 반드시 `overdare_stop`

플레이테스트 중에 `overdare_script_edit` / `instance_update` / `upsert`를 하면 `level.apply`가 타임아웃되고 **Studio 메인 스레드가 잠긴다. 사람이 Stop을 눌러야만 풀린다.**
`.ovdrjm`은 롤백돼서 데이터 손실은 없지만 세션은 날아간다.

**읽기는 플레이 중에도 안전하다** — `screenshot`, `ui_browse`, `observe`, `read_instance`.

### 1-2. `overdare_rc_batch`와 `overdare_rc_python`을 쓰지 마라

MCP README 원문:
> *"do not work against a shipping Studio build: `PUT /remote/batch` has **crashed Studio outright**, and `ExecutePythonCommand` returns 400 because the Python plugin is not loaded."*

1-1은 Stop으로 풀리지만 **이건 Studio를 죽인다.** 더 나쁘다.

### 1-3. 모든 프로퍼티 쓰기는 `pcall`

없는 프로퍼티 하나가 스크립트 전체를 중단시킨다. 특히 Roblox에서 가져온 코드를 처음 돌릴 때.

### 1-4. 배치 위치

- 맵 컨테이너 아래 ✗ (스왑 시 고아)
- 캐릭터 아래 ✗ (서버가 비표준 `BasePart`를 지운다)
- 런타임 스폰은 Movable 최상위 아래 — **Static 서브트리는 런타임 생성·이동 불가**

---

## 2. ★ `overdare_validate_lua` — 환각을 기계로 잡는다

Studio가 배포한 타입 정의(`overdare-types.d.lua`)로 Luau를 직접 검사한다. **이 판에서 가장 값진 도구다.**

### ⚠️ `--!strict` 없으면 아무것도 안 잡는다

실측(2026-09-08). 같은 파일, 첫 줄만 다르게:

```lua
-- --!nolint  →  "No problems found."     ← 전부 통과. 무의미하다
-- --!strict  →  네 개를 전부 잡는다:
TypeError: Key 'ZOffset' not found in external type 'ParticleEmitter'
TypeError: Key 'LightInfluence' not found in external type 'ParticleEmitter'
TypeError: Key 'BorderSizePixel' not found in external type 'Frame'
TypeError: Key 'ImageRectOffset' not found in external type 'ImageLabel'
```

같은 파일의 `FlipbookLayout = Enum.ParticleFlipbookLayout.Grid4x4`와 `Rotation = 45`는 **통과했다** — 존재한다는 뜻이다. 이 도구는 있다/없다를 양방향으로 알려준다.

**기존 프로젝트 파일은 대부분 `--!nolint`로 시작한다.** 그걸 검사해놓고 "깨끗하다"고 읽지 마라.

### 규칙

> **새로 쓰는 FX 스크립트는 `--!strict`로 시작하고, 인게임에 넣기 전에 `validate_lua`를 통과시켜라.**

### `--!strict`에서만 나오는 함정

| 쓰면 안 되는 것 | 대신 |
|---|---|
| `NumberSequence.new(0.5)` | `NumberSequence.new(0.5, 0.5)` — 1인자 오버로드가 strict에서 해결되지 않는다 |
| `ColorSequence.new(c)` | `ColorSequence.new(c, c)` |
| `obj:FindFirstChild("N")` | `obj:FindFirstChild("N", false)` — recursive 인자가 필수로 선언돼 있다 |
| `Enum.ParticleFlipbookLayout.Grid4x4` | `(Enum :: any).ParticleFlipbookLayout.Grid4x4` |
| `Instance.new("Trail")` | `(Instance :: any).new("Trail")` — 타입 목록에 없는 클래스 |

`LocalUnused` 같은 lint 잡음은 무시해도 된다. **`TypeError: Key '...' not found`가 진짜다.**

---

## 3. 확인 도구 — 무엇이 무엇을 보여주나

| 도구 | 보여주는 것 | 한계 |
|---|---|---|
| `overdare_screenshot` | **3D 뷰포트만** | **UI는 절대 안 나온다** |
| `overdare_ui_browse` | UI 트리 — dotted path, class, text, 정규화 rect, 가시성, 화면 안 여부 | **플레이테스트 중에만 동작** |
| `overdare_observe` | 캐릭터 + UI + 인스턴스 상태를 한 번에 | 값 확인용. "어떻게 보이는가"는 스크린샷 |
| `overdare_read_instance` | **저장된 프로젝트 파일**의 프로퍼티 전체 | 라이브 상태가 아니다 — 저장 안 한 변경은 안 보인다 |
| `Play.log` | 스크립트가 찍은 print | 플레이 후에만 |

**요약:** *값*은 `observe`, *모양*은 `screenshot`, *UI*는 `ui_browse`, *저장된 상태*는 `read_instance`.

### `observe` 쓰는 법

```
instances: ["Workspace.Something.Child"]            # 경로로 직접
instances: {class: "Sound", under: "...", maxDepth: 2}   # 검색
ui: true                                            # UI 전체
character: true                                     # 포즈·속도·standingOn
```

검색형으로 부르면 식별 정보(name/path/class/guid)만 돌아온다. 프로퍼티 값이 필요하면 **경로를 직접 지정**해라.
큰 씬에서 넓은 검색을 걸면 10초 타임아웃이 난다 — 범위를 좁혀라.

---

## 4. 배치 도구 — 두 벌이 있다

| | 도구 | 반영 | 언제 |
|---|---|---|---|
| 파일 | `overdare_create_instance` / `update_instance` / `delete_instance` | `overdare_apply` 리로드 필요 | 영구 구조 변경 |
| **라이브** | `overdare_instance_create` / `instance_update` / `instance_delete` | 즉시 | **FX 튜닝은 이쪽** |

`instance_update`는 **없는 프로퍼티 이름을 넣어도 호출이 실패하지 않는다** — 반환 `message`에 `[WARNING]`으로 오고 나머지는 적용된다. **반환값을 읽어라.**

복합 값은 배열 축약형을 받는다:
`Position`/`Size` `[xScale,xOffset,yScale,yOffset]` · `*Color3` `[r,g,b]` 0-255 · `AnchorPoint` `[x,y]` · `SliceCenter` `[minX,minY,maxX,maxY]` · `FontFace` `{Family,Style,Weight}`

MCP 경유 시퀀스는 생성자가 아니라 JSON이다:
`Color` = `[{Time, Color}]` · `Size`/`Transparency`/`Squash` = `[{Time, Value, Envelope?}]` · `Lifetime`/`Speed` = `{Min, Max}`

⚠️ **열거형 철자가 경로마다 다르다.** `ParticleEmitterShapeInOut`이 Luau에선 `Outward`/`Inward`, **MCP 스키마에선 `OutWard`/`InWard`**(대문자 W).

---

## 5. 에셋 반입

```
overdare_image_import(절대경로)  →  "ovdrassetid://<번호>"
```

이 id를 꽂는 곳: `ParticleEmitter.Texture` · `Beam/Trail.Texture` · `ImageLabel.Image` · `MeshPart.TextureId` · `MaterialVariant.*Map` · `Atmosphere.CloudTexture`

제약: **.png/.tga, 15 MB 이하, 권장 512×512** (저사양 256). *"권장"이라고만 적혀 있다* — 1024가 되는지는 실측해봐야 안다.
`asset_drawer_import`는 `assetType`이 `["MODEL"]`뿐이라 **텍스처를 못 넣는다.** 이미지는 위 경로가 유일하다.

메시는 `overdare_mesh_prepare`가 헤드리스 블렌더로 임포트 가능하게 만들어준다 — **메시당 30,000 삼각형 상한**, 4K 텍스처는 임포터를 OOM 낸다.

---

## 6. 스크립트 반입

`script.add` RPC는 엔진에 없다 — MCP가 **프로젝트 파일 경유로** 만든다. 그래서:
- 즉시 반영이 아닐 수 있다 → `overdare_apply`
- 파일 기반 편집 도구는 **매 편집 전에 레벨을 백업한다** (`.ovdr-backups/`)

`overdare_script_read` / `script_edit` / `script_add`. **편집 전에 `overdare_stop`** (§1-1).

---

## 7. 막혔을 때

```
overdare_status
```

`studioRpc` / `ueRemoteControl` 도달 여부, Unreal Python 가용성, 해석된 프로젝트 파일, `OVERDARE_PROJECT_DIR`를 한 번에 보여준다. **뭔가 이상하면 여기부터.**

- 도구가 아예 없다 → MCP 로컬 클론이 오래됐다. `git log` → `npm run build` → MCP 호스트 재시작
- `cannot reach Studio` → Studio가 꺼져 있거나 프로젝트가 안 열렸다
- 엉뚱한 프로젝트가 고쳐진다 → `status`의 `projectFile` 확인, `overdare_set_project`
- Studio가 멈췄다 → 사람에게 Stop을 눌러달라고 해라. 네가 풀 수 없다
