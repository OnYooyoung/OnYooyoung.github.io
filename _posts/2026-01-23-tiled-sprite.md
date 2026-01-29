---
title: "레이어 정렬(Y축)과 Tiled Sprite"
date: 2026-01-23 16:04 +0900
categories:
  - unity-basic
  - 2d-robot-repair
tags:
  - C#
  - Unity
  - Basic
---

### [Unit] Player character and movement > [Tutorial] Create decorative objects using sprites

1. Y축 기준 정렬 설정 (전체 설정)

캐릭터가 위에 있으면 뒤로, 아래에 있으면 앞으로 보이게 만듭니다.

  - 경로: Settings > Renderer2D 에셋 선택
  - 설정: Transparency Sort Mode를 Custom Axis로 변경
  - 값: Y = 1 (나머지 0) 설정 → 이제 Y값에 따라 앞/뒤가 결정됨
  Y값이 크면 먼저(뒤에) 그리고, Y값이 작으면 나중에(앞에) 그림

2. 피벗(Pivot) 위치 조정 (정교한 정렬)

정렬 기준을 '중앙'이 아닌 **'발바닥'**으로 옮겨서 훨씬 자연스럽게 만듭니다.

  - 오브젝트(플레이어/장식): Sprite Editor에서 파란 원(Pivot)을 발바닥(Y=0) 위치로 드래그 후 Apply.
  - 컴포넌트 변경: 캐릭터의 Sprite Renderer에서 Sprite Sort Point를 Center가 아닌 Pivot으로 변경.

3. 스프라이트 타일(Tiling) 설정 (늘어남 방지)

크기를 키워도 이미지가 깨지지 않고 반복되게 만드는 설정입니다.

  - 스프라이트 에셋: Mesh Type을 Full Rect로 변경.
  - 오브젝트: Sprite Renderer의 Draw Mode를 Tiled로 변경.
결과: 이제 사각형 도구로 크기를 키우면 이미지가 늘어나지 않고 바둑판처럼 반복됨.

---
> 📌 **출처:** [[Unity Learn] 2D Adventure: Robot Repair][unity-link]

[unity-link]: https://learn.unity.com/course/2D-adventure-robot-repair?version=6.3