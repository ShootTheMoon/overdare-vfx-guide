# 06 CHECKLIST — 이펙트 하나를 처음부터 끝까지

급하면 이 문서만 보고 시작해도 된다. 막히는 항목에서 링크로 들어가라.

---

## 시작 전 (프로젝트당 한 번)

- [ ] `overdare_status` — `studioRpc` / `ueRemoteControl` 둘 다 `reachable`, `projectFile`이 맞는 프로젝트인가
- [ ] [환각 차단 목록](https://github.com/ShootTheMoon/onlyoneshot-vfx/blob/main/02_DO_NOT.md)을 읽었나
- [ ] 능력 프로브를 돌렸나 — 특히 **`VFXRecipe` 배치 가능 여부**와 **런타임 `Instance.new`** ([01_WORKFLOW §0](01_WORKFLOW.md))
- [ ] 프로젝트에 이미 있는 것을 확인했나 — 전역 헬퍼(`_G.HUD.flash` 같은), 공용 모션 모듈, 이미 올라간 `ovdrassetid`

---

## 1. 정의

- [ ] 한 문장으로 썼나 — **무엇이 / 언제 / 얼마나 오래 / 얼마나 멀리서 보이나**
- [ ] 트리거가 뭔지 정했나 (기존 이벤트에 얹을 수 있나? 새 RemoteEvent를 만들 필요는 거의 없다)

## 2. 경로

- [ ] `presets.md`를 영어 키워드로 grep 했나 (동의어 2~3개까지)
- [ ] [결정 트리](03_ROUTES.md)를 탔나
- [ ] **절차적 `Part`로 바로 가지 않았나** — 가장 자유롭고 가장 비싸다
- [ ] 아틀라스를 굽기로 했다면 **프레임당 픽셀을 계산했나** (512 ÷ 4×4 = 128 px)

## 3. 재료

- [ ] (프리셋) `Resource` 값을 정확히 복사했나
- [ ] (레시피) 소스가 **맞는 레이어**에 있나 — 같은 이름이 레이어마다 다른 자산이다
- [ ] (레시피) `_R` 접미사면 `Duration`+`SpawnRate`, 아니면 `SpawnCount` — 섞지 않았나
- [ ] (아틀라스) [굽기 함정](04_BAKING.md)을 확인했나 — Render Method `Blended`, 프레임 패딩, 알파
- [ ] `overdare_image_import` 반환 id를 기록했나

## 4. 코드

- [ ] 스크립트가 **`--!strict`로 시작**하나
- [ ] **`overdare_validate_lua`를 통과**했나 — `TypeError: Key '...' not found`가 0건
- [ ] **×28 환산**을 했나 (또는 cm 값을 중복 환산하지 않았나)
- [ ] 모든 프로퍼티 쓰기가 `pcall` 안에 있나
- [ ] 사전 할당 후 재배치인가 (매 프레임 `Instance.new` 아님)
- [ ] 캡이 있나 — 넘으면 조용히 버리거나 오래된 것부터 제거
- [ ] 거리 컬링 반경이 **맵보다 크게** 잡혔나

## 5. 배치

- [ ] **`overdare_stop`을 먼저 했나** ← 플레이 중 쓰면 Studio가 잠긴다
- [ ] 라이브 RPC 도구(`overdare_instance_create/update`)를 쓰고 있나 (파일 경로 아님)
- [ ] 맵 컨테이너 아래가 아닌가 / 캐릭터 아래가 아닌가
- [ ] Movable 최상위 아래인가 (런타임 스폰이면)
- [ ] 이름에 전역 유일 접두사가 붙었나
- [ ] `instance_update` 반환 `message`에 `[WARNING]`이 없나 — 없는 프로퍼티는 조용히 무시된다

## 6. 튜닝

- [ ] `Size` / `Transparency` **커브**부터 만졌나 (`Rate`는 마지막)
- [ ] 생애 앞뒤 페이드가 있나 (팝 없음)
- [ ] `Transparency`와 함께 **`Size`도 줄어드나**
- [ ] `Lifetime` / `Rotation` / `RotSpeed`가 **범위값**인가
- [ ] `FlipbookStartRandom = true`인가
- [ ] 싼 이미터 여러 개로 나눴나 (비싼 것 하나 아님)
- [ ] 밀도를 한 번 더 줄여봤나

## 7. 검증

- [ ] **3D는 `overdare_screenshot`으로 봤나**
- [ ] **UI는 `overdare_ui_browse`로 봤나** — 스크린샷엔 UI가 안 나온다
- [ ] **실제 전투 거리**에서 확인했나
- [ ] 카메라가 뚫고 지나가도 화면이 안 막히나
- [ ] 양쪽 스폰 방향에서 다 봤나
- [ ] 색이 게임플레이 정보(팀색·경고색)와 안 겹치나
- [ ] **fps를 넣기 전과 비교**했나

## 8. 마무리

- [ ] `overdare_stop` → `overdare_save`
- [ ] **프로브·디버그 스크립트를 지우거나 `Enabled = false`** 했나
  (`ZZ_` 같은 임시 스크립트 하나가 로비 음악을 몇 주간 죽여놓은 적이 있다)
- [ ] 임시로 만든 Part/GUI를 정리했나
- [ ] **새로 알아낸 엔진 사실을 문서에 반영했나** — 다음 사람이 같은 실측을 반복하지 않게

---

## 막혔을 때 3초 진단

| 증상 | 먼저 볼 것 |
|---|---|
| 안 보인다 | ×28 안 함 · `Transparency = 1` · 카메라 뒤 · 맵 밖 좌표 |
| 화면을 덮는다 | ×28을 두 번 함 |
| 생애 끝에 안 사라진다 | `Size`를 같이 안 줄였다 |
| 코드가 조용히 아무 일도 안 한다 | 없는 프로퍼티/클래스 → `--!strict` + `validate_lua` |
| 한 덩어리로 맥동한다 | 랜덤 범위 없음 · `FlipbookStartRandom` 꺼짐 |
| UI가 화면에 안 보인다 | 스크린샷엔 원래 안 나온다 → `ui_browse` |
| Studio가 멈췄다 | 플레이 중에 썼다. **사람이 Stop을 눌러야 한다** |
| Studio가 죽었다 | `rc_batch` / `rc_python`을 썼다 |
| 맵 바꾸니 다 사라졌다 | 맵 컨테이너 아래 · 컬링 반경이 맵보다 작음 |
| 도구가 아예 없다 | MCP 로컬 클론이 오래됐다 → `npm run build` + 재시작 |
