# 박태훈 | 게임 클라이언트 프로그래머

> 1998년생, 군필  
> 📧 hunobas.dev@gmail.com

즉각적인 재미를 만들기 위해 게임 흐름, 판정, 피드백의 기술적 기반을 탄탄하게 다지는 엔지니어입니다.  
특정 언어나 플랫폼에 종속되지 않고, Claude와 같은 AI를 개발 파이프라인의 표준 도구로 활용하여 **스펙 정의 → 초안 생성 → 직접 검증·리팩터링 → 릴리스** 사이클을 실천합니다.

---

## 📌 프로젝트 1 — 목성의 노래 (Unity/C# | 5인 팀 | 2025.07~12)

내러티브 1인칭 3D 퍼즐 게임 | 담당: FSM 아키텍처, 렌더링 최적화, UI 인터랙션, 사운드 시스템

- ▶️ [플레이 데모](https://www.youtube.com/watch?v=UEz0kvJCfAg) | 📂 [GitHub](https://github.com/Hunobas/Song-Of-Jupitor)

---

### 1-1. FSM 기반 플레이 모드 아키텍처 설계

**문제:** 5가지 플레이 모드(Normal/Panel/Cinema/Dialog/Pause)가 상호배타적이어야 하는데, 패널을 여는 도중 컷씬이 재생되면 컷씬이 끝나도 **조작 불가 상태**가 되는 치명적 버그 발생. 상태 관련 코드가 여러 파일에 분산되어 디버깅 평균 **60분 소요**.

![GameState 버그 영상](https://github.com/user-attachments/assets/fa973d2f-df58-483d-ae3b-05d5104e9bc6)

*패널 모드 진입 중 시네마 모드가 끼어들면 발생하는 조작 불가 문제*

**해결:** 중앙 집중식 FSM으로 모든 플레이 모드를 단일 책임 관리.
```csharp
public void ChangePlayMode(IPlayMode next)
{
    if (next == null || ReferenceEquals(_activeMode, next)) return;

    // 시네마 모드는 일시정지 이외 모든 전환 요청 무시
    if (IsPlayingCinema && !ReferenceEquals(next, PauseMode)) return;

    // 패널 모드 강제 종료 후 전환
    if (IsOperatingPanel && PanelMode.Controller != null)
        PanelMode.Controller.EndPanelForcely();

    var prev = _activeMode;
    prev?.OnExit(next);
    _activeMode = next;
    _activeMode.OnEnter(prev);
    InputManager.Instance?.UpdateCursorLock();
}
```

각 모드의 `OnExit` 훅이 UI 상태·입력 바인딩·커서 잠금을 자동으로 정리하므로, 호출부에서 정리 로직을 신경 쓸 필요 없습니다.

| 개선 항목 | Before | After |
|---------|--------|-------|
| 상태 충돌 버그 | 주 2~3건 | **0건** |
| 디버깅 소요 시간 | 평균 60분 | 평균 30분 |
| 신규 모드 추가 | - | `IPlayMode` 구현만으로 20분 이내 |

📂 [GameState.cs 전체 코드](https://github.com/Hunobas/Song-Of-Jupitor/blob/7386ab978fc3115a13a700758c7a618567bc168a/Scripts/System/GameState.cs#L15)

<details>
<summary><b>엣지 케이스 처리: 일시정지 해제 시 이전 모드 복구 / 시네마 중 다이얼로그 전환 방지</b></summary>

- `PauseMode`가 `prevMode`를 저장하고 자체 `Resume()` 메서드로 복구 — [코드 보기](https://github.com/Hunobas/Song-Of-Jupitor/blob/7386ab978fc3115a13a700758c7a618567bc168a/Scripts/System/PauseMode.cs#L34)
- 시네마 모드는 `TimelineController`에서 자체적으로 종료, `ChangePlayMode`의 가드 조건이 외부 중단 차단 — [코드 보기](https://github.com/Hunobas/Song-Of-Jupitor/blob/7386ab978fc3115a13a700758c7a618567bc168a/Scripts/System/CinemaMode.cs#L27)

</details>

---

### 1-2. 렌더링 성능 최적화

**문제:** 400만 버텍스 + 300개 머터리얼 구성의 우주 정거장 씬에서 FPS 30~60으로 불안정.

**해결 과정:**

| 단계 | Batches | FPS |
|------|---------|-----|
| 기준선 | 2,650 | 30~60 |
| MeshBaker 텍스처 아틀라스 + 메쉬 콤바인 | 750 | 80~100 |
| 오클루전 컬링 추가 | 601 | **120+** |

단순 텍스처 아틀라스만 적용했을 때는 Verts가 72% 감소했으나 Draw Call이 74% 증가해 전체 성능이 오히려 하락했습니다. **가설 → 측정 → 반증 → 재설계** 사이클로 콤바인 메쉬 + 오클루전 컬링의 조합이 이 씬에 최적임을 확인했습니다.

<img width="1548" height="591" alt="최적화 전후" src="https://github.com/user-attachments/assets/6b35a453-6a45-4258-9635-3bcff6062e97" />

📜 [최적화 개발일지 전문](https://velog.io/@po127992/목성의-노래-MeshBaker-최적화-삽질기-텍스처-아틀라스만-vs-콤바인-메쉬까지)

---

### 1-3. 어드벤처 게임을 위한 사운드 아키텍처

**문제:** Unity 기본 AudioSource로는 페이드인/아웃, 크로스페이드, Duck(중요 사운드 재생 시 나머지 볼륨 자동 감소)을 매번 코루틴으로 직접 작성해야 했고, 기획자가 Inspector에서 설정하기 불편했습니다.

**해결:** 영상 편집의 트랙/클립 개념을 차용. `SoundSource`(트랙, Inspector 설정)와 `SoundEntry`(클립 엔티티, FluentAPI 제어)를 분리.

```csharp
// 프로그래머: 한 줄 체이닝으로 복잡한 연출 완성
_soundSource.PlayByNameOrNull("GeneratorStartUp")
    .WithFadeIn(0.5f)
    .WithFadeOut(1.0f)
    .WithPriority(SoundPriority.High)
    .WithOptions(PlayOptions.DuckOthers)
    .OnFinish(() => _soundSource.PlayByNameOrNull("GeneratorLoop")
        .WithLoop().WithOptions(PlayOptions.DuckOthers));
```

크로스페이드 시 페이드아웃되는 클립은 오브젝트 풀에서 꺼낸 `AudioSource`로 이관하여, 메모리 누수 없이 무한 크로스페이드를 지원합니다.

📂 [SoundManager](https://github.com/Hunobas/Song-Of-Jupitor/blob/main/Scripts/Sound/SoundManager.cs) | [SoundSource](https://github.com/Hunobas/Song-Of-Jupitor/blob/main/Scripts/Sound/SoundSource.cs) | [SoundEntry](https://github.com/Hunobas/Song-Of-Jupitor/blob/main/Scripts/Sound/SoundEntry.cs)

---

## 📌 프로젝트 2 — THE RATTUS (Node.js/Socket.IO | 5인 팀 | 크래프톤 정글)

실시간 멀티플레이 웹게임 | 담당: 아키텍처 설계, 실시간 상태 동기화

📂 [GitHub](https://github.com/younggun339/jungleTwo)

---

**낯선 기술을 빠르게 습득해 결과물을 만든 사례입니다.**

Next.js 및 Socket.IO를 처음 접한 상태에서 크래프톤 정글 합숙 일정 내에 실시간 멀티플레이 웹게임을 출시했습니다. AI 도구로 이벤트 기반 소켓 통신 구조를 빠르게 파악한 뒤, AI가 생성한 코드를 그대로 붙여넣지 않고 직접 읽으며 클라이언트-서버 간 상태 동기화 구조를 재설계했습니다. 짧은 개발 싸이클 안에 스펙 정의 → 구현 → 검증을 반복하기 위해 마이크로 회의를 직접 주도했습니다.

- 제약된 일정 안에 플레이 가능한 멀티플레이 빌드 출시
- 스프라이트 아틀라스로 묶은 인스턴싱을 유도하여 웹기반 이미지 콜을 최소화
- 낯선 스택에서도 아키텍처 설계 역할을 맡아 팀 전체 개발 속도에 기여

---

## 📋 추가 이력

| 항목 | 내용 |
|------|------|
| **현업 경험** | Dreamotion 인턴 3개월 — My Little Puppy 스팀 데모 출시, 루트모션 FSM 버그 3건 해결 |
| **CS 기초** | 크래프톤 정글 6개월 합숙, PintOS 137/141 통과, RB트리·malloc 직접 구현 |
| **알고리즘** | [![Solved.ac](http://mazassumnida.wtf/api/v2/generate_badge?boj=po1279)](https://solved.ac/po1279) |

---

## 📞 Contact

- 휴대폰: 010-3702-1279
- 이메일: hunobas.dev@gmail.com
- 블로그: [Velog](https://velog.io/@po127992/posts)
- 깃허브: [GitHub](https://github.com/hunobas)
