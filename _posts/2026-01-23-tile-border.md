---
title: "Box collider 타일 위 이동 제한"
date: 2026-01-23 18:04 +0900
categories:
  - unity-basic
  - 2d-robot-repair
tags:
  - C#
  - Unity
  - Basic
---

### [Unit] Game environment and physics > [Tutorial] Set up tilemap collision

1단계: 타일맵에 "벽" 설치하기 (전체 차단)
- Hierarchy 창 → Grid 안의 Tilemap 선택 → Add Component 버튼 클릭 → Tilemap Collider 2D 추가

2단계: 지나갈 수 있는 "길" 뚫기 (선택 해제) 
- 바닥, 풀밭 등 캐릭터가 밟고 다닐 타일들을 모두 선택 (Ctrl 또는 Shift 활용) → Inspector 창에서 Collider Type 항목 찾아 설정을 Sprite에서 **None**으로 변경

3단계:
- 타일맵의 성능을 높이고 캐릭터가 타일 틈에 끼는 버그를 막아주는 '콜라이더 합치기(Composite)'
- Tilemap 오브젝트에 Composite Collider 2D를 추가 → Tilemap Collider 2D 컴포넌트의 Composite Operation을 Merge로 설정( 수많은 타일 사각형 콜라이더가 하나의 큰 덩어리로 합쳐짐) → 추가된 Rigidbody 2D의 Body Type을 Static으로 바꿈

---
> 📌 **출처:** [[Unity Learn] 2D Adventure: Robot Repair][unity-link]

[unity-link]: https://learn.unity.com/course/2D-adventure-robot-repair?version=6.3