# 영웅 지휘관 프로토타입 설계 (hero 모드)

- 작성일: 2026-10-02
- 브랜치: `hero-prototype`
- 상태: 승인됨 (구현 계획 작성 전)

## 1. 목적과 가설

**목적:** 사내 검증용 프로토타입. 판매/배포 목적 아님. 기존 RA 에셋을 임시로 사용한다.

**검증할 가설:** "건설 가능 영역이 고정된 건설소가 아니라 움직이는 영웅 유닛을 따라다니면,
기지 확장이 영웅의 위치 선택과 보호 문제로 바뀌어 RTS의 의사결정이 더 흥미로워진다."

**범위에 포함:** 영웅 주변에서만 건설, 영웅 사망 시 병영에서 재훈련, AI 스커미시 대응.

**범위 밖 (명시적 제외):** 영웅 레벨업/스킬 선택, 오라 버프, 자동 부활, AI의 영웅 이동 확장,
캠페인/미션, 자체 아트/사운드, 엔진 C# 수정.

## 2. 구조

엔진 C# 코드는 수정하지 않는다. RA 모드를 그대로 불러오고 그 위에 덮어쓰기 규칙만 얹는
새 모드 `hero`를 만든다. 원본 `mods/ra`는 건드리지 않으므로 런처에서 RA와 hero를 나란히 비교할 수 있다.

```
mods/hero/
  mod.yaml            ← mods/ra/mod.yaml 복사 후 수정
  rules/hero.yaml     ← 영웅 액터 정의 + 건설 규칙 덮어쓰기
  rules/hero-ai.yaml  ← AI 모듈 조정
  fluent/hero.ftl     ← 모드 제목, 영웅 이름/설명 문자열
```

`mod.yaml` 변경점:
- `Metadata` 제목/창 제목을 hero 전용 문자열로 교체.
- `FileSystem.SystemPackages`에 `$hero: hero` 마운트 추가, `mods/ra`를 `ra` 패키지로 마운트해
  기존 `ra|...` 경로를 그대로 사용.
- 콘텐츠 경로 `~^SupportDir|Content/ra/v2/`와 `ContentInstallerMod: ra-content`는 RA와 공유.
  RA 콘텐츠를 한 번 설치하면 두 모드 모두 사용 가능.
- `Rules:` 목록 끝에 `hero|rules/hero.yaml`, `hero|rules/hero-ai.yaml` 추가 (RA 규칙 뒤에 로드되어 덮어씀).
- `FluentMessages:`에 `hero|fluent/hero.ftl` 추가.
- `Missions:` 섹션 제거 (캠페인 미션은 MCV/건설소 전제라 지원하지 않음).
- `SupportsMapsFrom: ra` 유지 → RA 스커미시 맵 사용.

## 3. 영웅 유닛 `HERO` (지휘관)

| 항목 | 값 | 비고 |
|---|---|---|
| 기반 | `^Soldier` 상속 | 보병 공통 동작 |
| 외형 | `RenderSprites: Image: e1` + 상시 엘리트 계급장(`WithDecoration`, `rank`/`rank-elite`) | 일반 소총병 외형을 임시로 쓰고 계급장으로 구분. 추후 자체 아트로 교체 |
| 체력 | 20000 | 소총병(5000)의 4배 |
| 무기 | `M1Carbine` | 소총병과 동일. 전투 유닛이 아니라 지휘 유닛 |
| 건설 반경 | `BaseProvider: Range: 10c0` | 건설소는 16c0 |
| 건설 인접 기준 | `GivesBuildableArea: AreaTypes: building` | |
| 건물 생산 | `Production: Produces: Building, Defense` | 영웅이 살아 있는 동안 건물/방어 탭 활성 |
| 재훈련 | `Buildable: Queue: Infantry, Prerequisites: ~barracks, BuildLimit: 1` | 진영 무관, 동시에 1명 |
| 비용 | 1500 | |
| 승리 조건 | `MustBeDestroyed: RequiredForShortGame: true` | MCV와 같은 역할 |
| AI 자세 | `AutoTarget: InitialStanceAI: Defend` | AI 영웅이 적을 쫓아 본진을 비우지 않게 |

수치(반경, 체력, 비용)는 첫 시험값이며 플레이테스트로 조정한다.

## 4. 건설 규칙

엔진 동작 (`OpenRA.Mods.Common/Traits/Buildings/Building.cs` `IsCloseEnoughToBase`):
건물 배치는 (1) 아군 `BaseProvider` 반경 안이고 **그리고** (2) `GivesBuildableArea`를 가진 액터로부터
`RequiresBuildableArea.Adjacent` 칸 이내여야 한다. (1)은 로비 옵션 `buildradius`가 켜져 있을 때만 검사된다.

변경:
1. **시작 유닛:** 모든 `StartingUnits@*`의 `BaseActor: mcv` → `hero`. 지원 유닛 구성은 유지.
2. **인접 조건 무력화:** 모든 `RequiresBuildableArea`의 `Adjacent`를 10으로 (기본 2, 방어시설 7, 일부 8). 영웅 자신이 `GivesBuildableArea`이므로
   영웅 반경(10c0) 안의 위치는 항상 인접 조건을 만족 → 실질 조건은 "영웅 반경 10칸 이내" 하나.
3. **건설 반경 옵션 고정:** `MapBuildRadius`의 `BuildRadiusCheckboxEnabled: true`, `BuildRadiusCheckboxLocked: true`.
   꺼지면 (1)이 생략되어 메커닉이 깨지기 때문.
4. **MCV/건설소 무력화:** MCV는 생산 불가(`Buildable.Prerequisites: ~disabled`), FACT는 `-BaseProvider`.
   맵에 미리 배치된 경우를 위해 액터 정의는 유지.

## 5. 영웅 사망 처리

- 건물 생산자(`Production: Building, Defense`)가 사라지므로 `ClassicProductionQueue`가 비활성화되고
  진행 중 항목은 `ClearQueue()`로 취소·환불된다 (`ClassicProductionQueue.cs` Tick).
- 병영에서 영웅을 재훈련하면 큐가 다시 활성화된다.
- 병영도 없으면 새 건물을 지을 수 없다 (사실상 종료). 남은 건물이 있는 한 패배는 아님.

## 6. AI (`rules/hero-ai.yaml`)

RA의 4개 AI(rush, normal, turtle, naval) 모두에 적용:

| 모듈 | 변경 | 이유 |
|---|---|---|
| `SquadManagerBotModule@*` | `ExcludeFromSquadsTypes`에 `hero` 추가 | 영웅이 공격 부대로 차출되지 않게 |
| `UnitBuilderBotModule@*` | `UnitsToBuild: hero: 1000`, `UnitLimits: hero: 1` | 사망 시 재훈련. 1000은 다른 유닛 가중치 합(약 600)보다 커서 병영이 비면 거의 확실히 영웅을 고름. 살아 있으면 `BuildLimit`으로 후보에서 빠짐 |
| `McvManagerBotModule@*` | 제거 | MCV가 없으므로 무의미한 시도 방지 |
| `BaseBuilderBotModule@*` | `MaxBaseRadius: 10` | 영웅 반경과 맞춰 배치 실패 반복 감소 |

`BaseBuilderBotModule`은 건설소(`ConstructionYardTypes: fact`)가 없으면 시작 위치를 기지 중심으로 쓴다.
AI 영웅은 `Defend` 자세로 시작 위치 근처에 머물므로 AI는 본진 주변에 건설한다.

## 7. 검증

자동 (구현 완료 조건):
1. 빌드: `./make.ps1 all` 성공.
2. 규칙 검사: `utility.cmd hero --check-yaml` 오류 0건.
3. 실행: `launch-game.cmd Game.Mod=hero` 로 기동 후 콘텐츠 설치 화면 또는 메인 메뉴 도달, 로그에 예외 없음.

수동 플레이 체크리스트 (사용자 확인, RA 콘텐츠 설치 필요):
- [ ] 스커미시 시작 시 MCV 대신 영웅이 있고 건물 탭이 활성화되어 있다.
- [ ] 건물 배치 시 영웅 반경 원이 보이고, 원 밖에는 배치할 수 없다.
- [ ] 영웅을 이동시키면 새 위치 주변에 건설할 수 있다.
- [ ] 영웅이 죽으면 건물 탭이 사라지고 진행 중이던 건설이 환불된다.
- [ ] 병영에서 영웅을 재훈련하면 건물 탭이 복구되고, 두 번째 영웅은 훈련할 수 없다.
- [ ] AI가 기지를 짓고 공격해 오며, AI 영웅이 죽으면 재훈련한다.
- [ ] 원본 RA 모드는 기존과 똑같이 동작한다.

## 8. 알려진 한계

- AI는 영웅을 움직여 확장하지 않는다 (본진 주변 건설만). 필요하면 후속 작업으로 C# 봇 모듈 추가.
- 캠페인/미션 미지원.
- 콘텐츠 설치기는 설치 후 RA 모드로 돌아간다 (`ra-content`가 `Mod: ra`로 고정). 설치 후 hero 모드로 다시 실행.
- 외형·음성은 RA 일반 보병 에셋 임시 사용.
