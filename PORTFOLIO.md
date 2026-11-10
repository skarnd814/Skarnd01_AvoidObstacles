# Avoid Obstacles — 포트폴리오

> **유니티의 기능 체험을 목적으로 만든 2D 아케이드 게임**입니다.
> 처음 만든 유니티 2D 프로젝트로, 물리 엔진·프리팹·UI·오디오·파티클·데이터 저장 같은
> 유니티의 기본 기능을 하나씩 직접 써 보며 게임 하나로 엮었습니다.

| 항목 | 내용 |
|---|---|
| 장르 | 2D 아케이드 / 장애물 피하기 |
| 엔진 | Unity 6 (6000.3.25f1), 2D URP |
| 언어 | C# |
| 사용 기능 | Rigidbody2D, Physics Material 2D, Prefab, TextMeshPro, AudioSource, Particle System, PlayerPrefs, SceneManager |

---

## 1. 게임 설명

하늘에서 끝없이 떨어지는 장애물을 좌우로 움직이며 피하는 게임입니다.
플레이어는 통통 튀는 공이며, 장애물에 한 번이라도 닿으면 게임이 끝납니다.
오래 버틸수록 장애물이 더 자주, 더 빠르게 떨어지고 점수도 더 빨리 오릅니다.

### 조작법

| 키 | 동작 |
|---|---|
| `A` / `D` 또는 `←` / `→` | 좌우 이동 |
| `Space` | 바라보는 방향으로 짧게 대시 (쿨타임 3초) |
| `R` | 게임 오버 후 재시작 |

### 게임 규칙

- 버틴 시간만큼 점수가 오르며, 시간이 지날수록 초당 점수가 커집니다.
- 최고 점수(BEST)는 게임을 껐다 켜도 남아 있습니다.
- 대시 UI가 **회색**이면 쿨타임 중, **흰색**이면 사용 가능입니다.

---

## 2. 작동 원리

### 2-1. 전체 구조

게임은 다섯 개의 스크립트가 각자 역할을 나눠 맡습니다.

| 스크립트 | 역할 |
|---|---|
| `PlayerController` | 이동·대시 입력, 충돌 감지, 재시작 |
| `ObstacleSpawner` | 장애물 생성과 난이도 상승 |
| `Obstacle` | 바닥에 닿거나 화면 밖으로 나간 장애물 삭제 |
| `GameManager` | 점수 계산, 최고 점수 저장, GAME OVER UI |
| `ExplosionEffect` | 충돌 시 파티클과 효과음 재생 |

```
[ObstacleSpawner] --생성--> [Obstacle] --충돌--> [Player]
                                                   |
                         +-------------------------+------------------+
                         v                         v                  v
                  [GameManager]            [ObstacleSpawner]   [ExplosionEffect]
                  점수 고정·저장             생성 중지            파티클·효과음
                  GAME OVER 표시
                                     Time.timeScale = 0 (정지)
```

### 2-2. 플레이어 이동과 대시 — `Rigidbody2D`

- 입력은 `Update`에서 `Input.GetAxisRaw("Horizontal")`로 읽고, 실제 이동은 물리 주기에 맞춘 `FixedUpdate`에서
  `Rigidbody2D.linearVelocity`를 바꿔 처리합니다.
- **X축 속도만 바꾸고 Y축 속도는 그대로 둡니다.** 그래서 바닥의 `Physics Material 2D`(탄성) 덕분에 생기는
  통통 튀는 움직임이 이동 중에도 유지됩니다.
- 대시는 `dashDuration`(0.15초) 동안 X축 속도를 `dashSpeed`(15)로 올리는 방식입니다.
  마지막 대시 시각(`lastDashTime`)과 `Time.time`을 비교해 쿨타임을 판정하고, 같은 값으로 UI 색을 바꿉니다.

### 2-3. 장애물 생성과 난이도 상승 — `Prefab` · `Instantiate`

- 장애물은 프리팹으로 만들어 두고, `ObstacleSpawner`가 정해진 간격마다 무작위 X 위치에 `Instantiate`로 생성합니다.
- 경과 시간에 따라 두 가지가 함께 어려워집니다.
  - **생성 간격**: `1.0초 - (경과 시간 × 0.02)`, 최소 0.2초까지 줄어듦
  - **낙하 속도**: 생성할 때 `gravityScale = 1.0 + (경과 시간 × 0.03)`로 중력을 키움
- 바닥에 닿거나 Y 좌표가 -6 아래로 내려간 장애물은 `Destroy`로 지워 메모리가 쌓이지 않게 합니다.

### 2-4. 점수와 최고 점수 — `TextMeshPro` · `PlayerPrefs`

- 매 프레임 `(기본 초당 점수 + 경과 시간 × 가속도) × Time.deltaTime`을 더해, 오래 버틸수록 점수가 빨리 오릅니다.
- 점수는 `TextMeshProUGUI`로 화면에 표시합니다.
- 현재 점수가 최고 점수를 넘으면 `PlayerPrefs`에 바로 저장하고, 게임 오버 때 `PlayerPrefs.Save()`로 확정합니다.

### 2-5. 충돌과 게임 오버 — `OnCollisionEnter2D` · `Time.timeScale`

- 플레이어가 `Obstacle` 컴포넌트를 가진 물체와 부딪히면 게임 오버로 처리합니다.
- `GameManager`에 게임 오버를 알리고, `ObstacleSpawner`의 생성을 멈춘 뒤 `Time.timeScale = 0`으로 게임 전체를 정지합니다.
- `R` 키를 누르면 `timeScale`을 1로 되돌리고 `SceneManager.LoadScene`으로 현재 씬을 다시 불러옵니다.

### 2-6. 연출 — `Particle System` · `AudioSource`

- 충돌 순간 폭발 파티클 프리팹을 플레이어 위치에 생성합니다.
- 충돌 효과음은 `AudioSource.ignoreListenerPause`를 켜 두어, 게임이 정지된 뒤에도 끊기지 않고 재생됩니다.
- 대시 효과음은 `AudioSource.PlayClipAtPoint`로 카메라 위치에서 재생해 2D 게임에서도 소리가 고르게 들립니다.
- 배경음악은 루프로 계속 재생됩니다.

---

## 3. 이 프로젝트로 익힌 것

- **물리 기반 이동**: `Update`(입력)와 `FixedUpdate`(물리)를 나누는 이유와, 속도의 한 축만 바꿔 물리 효과를 살리는 방법
- **프리팹 활용**: 같은 물체를 반복 생성하고 필요 없어지면 지우는 흐름
- **난이도 설계**: 시간 값 하나로 생성 간격·낙하 속도·점수 가속을 함께 조절하는 방법
- **시간 정지 처리**: `Time.timeScale = 0` 상태에서도 소리와 입력이 동작하게 만드는 방법
- **데이터 저장**: `PlayerPrefs`로 간단한 기록을 영구 저장하는 방법
