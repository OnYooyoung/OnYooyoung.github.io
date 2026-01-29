---
title: "적 애니메이션 생성"
toc: true
toc_sticky: true
date: 2026-01-24 13:55 +0900
categories:
  - unity-basic
  - 2d-robot-repair
tags:
  - C#
  - Unity
  - Basic
---

## [Unit] Player character and movement > [Tutorial] Create 2D sprite animations

## 🔍 애니메이션 설정

1. 애니메이터 설정

    적 캐릭터가 애니메이션을 가질 수 있도록 '두뇌'를 달아주는 단계입니다.
    - 컴포넌트 추가: 적 프리팹에 Animator 컴포넌트를 추가합니다.
    - 컨트롤러 생성: Enemy라는 이름의 Animator Controller를 만들고, 이를 Animator 컴포넌트의 Controller 칸에 연결합니다.
2. 왼쪽 걷기 애니메이션 (EnemyLeft)

    스프라이트 여러 장을 교체하여 움직이는 효과를 만듭니다.

    - 클립 생성: 애니메이션 창에서 EnemyLeft 클립을 새로 만듭니다.
    - 스프라이트 적용: 스프라이트 시트에서 왼쪽을 보는 4개의 이미지를 선택해 타임라인에 드래그합니다.
    - 속도 조절: 기본 60으로 설정된 **Samples(초당 프레임 수)**를 4로 낮추어 자연스러운 속도로 맞춥니다.
3. 오른쪽 걷기 애니메이션 (EnemyRight)

    이미지를 반전시켜 작업 시간을 단축하는 효율적인 방법입니다.

    - 클립 생성: 새 클립 EnemyRight를 만듭니다.
    - 이미지 반전: Add Property를 통해 Sprite Renderer > Flip X 속성을 추가합니다.
    - 반전 적용: 애니메이션의 시작과 끝 프레임에서 Flip X를 체크(True) 상태로 두어, 캐릭터가 오른쪽을 보며 걷게 만듭니다.

---
> 📌 **출처:** [[Unity Learn] 2D Adventure: Robot Repair][unity-link]

[unity-link]: https://learn.unity.com/course/2D-adventure-robot-repair?version=6.3