---
title: "플레이어 이동"
toc: true
toc_sticky: true
date: 2026-01-22 19:16 +0900
categories:
  - unity-basic
  - 2d-robot-repair
tags:
  - C#
  - Unity
  - Basic
---

## [Unit] [Unit] Player character and movement > [Tutorial] Create Input Actions for player character movement

```csharp
using UnityEngine;
using UnityEngine.InputSystem; // 유니티의 새로운 입력 시스템 기능을 가져옵니다.

public class PlayerController : MonoBehaviour
{
    // [인스펙터 창에 노출됨] 이동 입력을 담당할 액션 변수입니다.
    // 여기서 키보드(WASD), 조이스틱 등의 설정을 연결합니다.
    public InputAction MoveAction;

    // 게임이 시작될 때 단 한 번 실행되는 함수
    void Start()
    {
        // MoveAction(입력 설정)을 활성화합니다. 
        // 이걸 안 하면 키보드를 눌러도 아무 반응이 없습니다.
        MoveAction.Enable();
    }

    // 매 프레임(화면이 갱신될 때마다) 실행되는 함수
    void Update()
    {
        // 1. 현재 입력된 값(방향)을 읽어옵니다.
        // 상하좌우 입력을 Vector2(x, y) 형태로 가져옵니다. (예: 오른쪽 클릭 시 x=1, y=0)
        Vector2 move = MoveAction.ReadValue<Vector2>();

        // 2. 콘솔 창에 현재 입력 값을 출력합니다. (테스트용)
        Debug.Log(move);

        // 3. 새로운 위치를 계산합니다.
        // (현재 위치) + (입력된 방향 * 속도)
        // 0.01f는 이동 속도입니다. 숫자가 커지면 더 빨리 움직입니다.
        Vector2 position = (Vector2)transform.position + move * 0.01f;

        // 4. 계산된 새로운 위치를 캐릭터의 실제 위치(transform.position)에 적용합니다.
        transform.position = position;
    }
}
```

## 🔍 3.0f (속도 조절)

역할: 이동 속도를 결정합니다.
move 값은 보통 -1에서 1 사이의 아주 작은 값입니다. 여기에 3.0을 곱해줌으로써 우리가 눈으로 볼 수 있을 만큼 빠르게 움직이게 만듭니다.  

## 🔍 Time.deltaTime (프레임 보정 - 가장 중요!)

- 문제점: 컴퓨터 사양이 좋아서 초당 144프레임이 나오는 사람과, 사양이 낮아 30프레임이 나오는 사람이 있다면? 그냥 움직일 시 성능 좋은 컴퓨터에서 캐릭터가 훨씬 빨리 움직이게 됩니다.
- 해결: Time.deltaTime은 **"이전 프레임에서 현재 프레임까지 걸린 시간"**을 의미합니다.
- 결과: 이 값을 곱해주면 컴퓨터 성능에 관계없이 모든 플레이어가 1초에 정확히 3.0만큼 이동하게 됩니다. (부드러운 이동의 필수 요소입니다.)

---
## [Unit] Player character and movement > Implement object collisions for your 2D game
## ⚠️ 캐릭터가 물체와 충돌 시 지터링(Jittering, 떨림)이 생기는 이유 (비유)

캐릭터가 벽을 뚫고 지나가려고 할 때, 1초에 수십 번씩 이런 일이 벌어지는 겁니다.

- 너(입력): "키보드 눌렀으니까 캐릭터 오른쪽으로 1cm 이동해!" (벽 속으로 캐릭터를 밀어 넣음)
- 물리 엔진: "어? 여기 벽인데? 너 들어가면 안 돼! 다시 왼쪽으로 1cm 튕겨 나가!" (캐릭터를 다시 밖으로 밀어냄)  

이 [안으로 밀기 -> 밖으로 튕기기] 과정이 화면 갱신(Frame)마다 미친 듯이 반복되니까, 우리 눈에는 캐릭터가 제자리에 못 있고 파르르 떨리는 것처럼 보이는 거예요.

 

## ✅ 해결

게임 오브젝트를 Transform 컴포넌트 대신 Rigidbody 2D 컴포넌트를 사용하여 이동해야 합니다

```csharp
using UnityEngine;
using UnityEngine.InputSystem; // 새로운 입력 시스템(Input System) 기능을 사용하기 위해 필요합니다.

public class PlayerController : MonoBehaviour
{
    // 유니티 에디터 인스펙터 창에서 설정한 입력 액션(예: WASD, 화살표)을 연결하는 변수입니다.
    public InputAction MoveAction;
    
    // 물리 연산을 담당하는 Rigidbody2D 컴포넌트를 담을 그릇입니다.
    Rigidbody2D rigidbody2d;
    
    // 입력받은 방향 값(x, y)을 저장해두었다가 FixedUpdate에서 사용하기 위한 변수입니다.
    Vector2 move;

    // 게임이 시작될 때 딱 한 번 실행됩니다.
    void Start()
    {
        // 1. 설정한 입력 액션을 활성화합니다. (이걸 안 하면 키보드를 눌러도 반응이 없어요!)
        MoveAction.Enable();
        
        // 2. 이 오브젝트에 붙어있는 Rigidbody2D 컴포넌트를 가져와서 변수에 저장합니다.
        rigidbody2d = GetComponent<Rigidbody2D>();
    }

    // 매 프레임(화면 갱신 주기에 맞춰) 실행됩니다. 입력 감지에 최적입니다.
    void Update()
    {
        // 실시간으로 입력된 방향 값을 읽어옵니다. (예: W를 누르면 y=1, D를 누르면 x=1)
        move = MoveAction.ReadValue<Vector2>();
        
        // 콘솔 창에 현재 입력 값을 표시합니다. (테스트용)
        Debug.Log(move);
    }

    // 일정한 시간 간격(기본 0.02초)으로 실행됩니다. 물리 이동 처리에 최적입니다.
    void FixedUpdate()
    {
        /* [물리 이동 공식]
           현재 위치 + (방향 * 속도 * 시간)
           - rigidbody2d.position: 물리 엔진이 관리하는 현재 캐릭터 위치
           - 3.0f: 이동 속도 (숫자가 클수록 빨라짐)
           - Time.deltaTime: 프레임 간의 시간 차이를 보정하여 어디서든 일정한 속도를 유지하게 함
        */
        Vector2 position = (Vector2)rigidbody2d.position + move * 3.0f * Time.deltaTime;
        
        // 계산된 위치로 캐릭터를 부드럽게 옮겨달라고 물리 엔진에 명령합니다.
        // 이 함수를 써야 벽을 뚫지 않고 매끄럽게 충돌 처리가 됩니다.
        rigidbody2d.MovePosition(position);
    }
}
```

## ❓ 왜 FixedUpdate 이어야 하나요?

박자 일치: 물리 엔진은 **정해진 시간(0.02초)**마다 계산하는데, 이 박자에 맞춰 명령을 내릴 수 있는 곳이 FixedUpdate뿐이기 때문입니다. (불규칙한 Update에서 하면 박자가 꼬여서 떨림 발생)
계산 순서: 유니티는 [이동 명령(FixedUpdate) → 물리 계산 → 화면 표시(Update)] 순서로 일합니다. 즉, 물리 엔진이 계산을 시작하기 직전에 이동 경로를 알려줘야 충돌 처리가 완벽해집니다.
Update: 키보드 입력 감지 (반응속도 중요)
FixedUpdate: 실제 캐릭터 이동 (물리 규칙 준수) 

## ❓ Time.deltaTime?

FixedUpdate 안에서 Time.deltaTime을 쓰면, 유니티는 똑똑하게도 Time.fixedDeltaTime 값(0.02)을 알아서 반환해 줍니다.

---
> 📌 **출처:** [[Unity Learn] 2D Adventure: Robot Repair][unity-link]

[unity-link]: https://learn.unity.com/course/2D-adventure-robot-repair?version=6.3