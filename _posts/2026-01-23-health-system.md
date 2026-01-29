---
title: "체력 시스템 구현"
toc: true
toc_sticky: true
date: 2026-01-23 22:01 +0900
categories:
  - unity-basic
  - 2d-robot-repair
tags:
  - C#
  - Unity
  - Basic
---

## [Unit] Health system

## 플레이어 스크립트

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerController : MonoBehaviour
{
    // Variables related to player character movement
    public InputAction MoveAction;
    Rigidbody2D rigidbody2d;
    Vector2 move;
    public float speed = 3.0f;

    // Variables related to the health system
    public int maxHealth = 5;
    // get: 데이터를 요청받았을 때 실행되는 '입구(함수)'
    // return: 그 입구로 들어온 사람에게 들려보낼 '결과물'
    public int health { get { return currentHealth; } }
    int currentHealth;
    
    // Variables related to temporary invincibility 무적
    public float timeInvincible = 2.0f;
    bool isInvincible;
    float damageCooldown; // 무적 쿨타임

    // Start is called once before the first execution of Update after the MonoBehaviour is created
    void Start()
    {
        MoveAction.Enable();
        rigidbody2d = GetComponent<Rigidbody2D>();
        currentHealth = maxHealth;
    }

    // Update is called once per frame
    void Update()
    {
        move = MoveAction.ReadValue<Vector2>();
        //Debug.Log(move);
        if (isInvincible)
        {
            damageCooldown -= Time.deltaTime;
            if (damageCooldown < 0)
            {
                isInvincible = false;
            }
        }
    }

    // FixedUpdate has the same call rate as the physics system
    void FixedUpdate()
    {
        Vector2 position = (Vector2)rigidbody2d.position + move * speed * Time.deltaTime;
        rigidbody2d.MovePosition(position);
    }

    // 외부에서 데미지를 주거나(amount가 음수), 힐을 줄 때(amount가 양수) 호출하는 함수
    public void ChangeHealth(int amount)
    {
    	if (amount < 0) // 데미지 줄 때
        {
            if (isInvincible)
            {
                return;
            }
            isInvincible = true;
            damageCooldown = timeInvincible;
        }
        
        /* Mathf.Clamp 설명
           현재 체력에 받은 양을 더하되, 그 결과가 0보다 작아지거나 maxHealth보다 커지지 않게 '고정'합니다.
           예: 체력이 5인데 힐을 100 받아도 최대치인 5로 유지됨! 
        */
        currentHealth = Mathf.Clamp(currentHealth + amount, 0, maxHealth);

        // 현재 체력 상태를 콘솔창에 출력합니다. (예: 3/5)
        Debug.Log(currentHealth + "/" + maxHealth);
    }
}
```

## 🔎 코드 흐름

- 데미지 발생 (ChangeHealth 호출):
  - 맨처음 isInvincible이 true가 됩니다.
  - damageCooldown이 2.0초(timeInvincible)로 꽉 채워집니다.
- 매 프레임 실행 (Update 함수):
  - 유니티는 매 프레임마다 Update를 실행합니다.
  - 지금 isInvincible이 true니까, if (isInvincible) 블록 안으로 들어갑니다.
  - damageCooldown -= Time.deltaTime; 이 코드가 실행되면서 2.0 → 1.9 → 1.8... 순식간에 숫자가 줄어듭니다.
- 무적 해제:
  - 약 2초 뒤에 damageCooldown이 0보다 작아지면, isInvincible = false;가 실행됩니다.
  - 이제 다시 데미지를 입을 수 있는 상태가 됩니다!
 

## ❓get은 왜 사용하나요?

get 안에서 계산을 한 뒤에 return을 한다고 생각해보면 왜 이게 필요한지 더 명확해집니다.

```csharp
private float rawDamage = 100f;
private float defense = 20f;

// 실제 최종 데미지는 계산해서 알려줘야 함!
public float finalDamage 
{
    get 
    { 
        // 1. 여기서 계산(로직)을 하고
        float calculated = rawDamage - defense; 

        // 2. 최종 결과물만 딱 들여보냄!
        return calculated; 
    }
}
 ```

## ❓damageCooldown을 왜 FixedUpdate가 아니라 Update에서 처리 하나요?

- 시간의 정확성: Update는 우리가 눈으로 보는 화면의 주사율과 동기화됩니다. 무적 시간처럼 "플레이어가 느끼는 실제 시간"은 Time.deltaTime을 사용하는 Update에서 깎는 것이 훨씬 정확합니다.
- 입력과의 반응성: 플레이어의 피격이나 상태 변화는 매 프레임 체크되어야 합니다. FixedUpdate는 물리 주기에 따라 Update보다 적게 혹은 더 많이 실행될 수 있어, 타이머가 미세하게 어긋날 수 있습니다.

| 구분 | Update (권장) | FixedUpdate |
| :--- | :--- | :--- |
| **기준** | 실제 흐름 시간 (프레임 기반) | 물리 연산 주기 (고정 시간 기반) |
| **적합한 작업** | 타이머, 입력 체크, 상태 변경 | 힘(Force), 속도(Velocity) 변경 |
| **이유** | 실제 체감 시간과 가장 일치함 | 물리 엔진의 계산 효율을 위함 |

## 체력 회복 아이템 스크립트

```csharp
using UnityEngine;

// HealthCollectible 클래스: 아이템(체력 회복 아이템 등)에 부착하는 스크립트입니다.
public class HealthCollectible : MonoBehaviour
{
    // OnTriggerEnter2D: 이 오브젝트의 Trigger Collider에 다른 오브젝트가 들어왔을 때 호출됩니다.
    // 'other' 변수는 방금 부딪힌(겹쳐진) 상대방의 Collider 정보입니다.
    void OnTriggerEnter2D(Collider2D other)
    {
        // 1. 부딪힌 상대방(other)에게 'PlayerController'라는 스크립트가 있는지 확인합니다.
        PlayerController controller = other.GetComponent<PlayerController>();

        // 2. 만약 상대방에게 PlayerController가 있고 최대 체력이 아니면
        if (controller != null && controller.health < controller.maxHealth)
        {
            // 3. 여기서는 플레이어 스크립트의 ChangeHealth 함수를 호출해 체력을 1 증가시킵니다.
            controller.ChangeHealth(1);
            
            // 4. 아이템을 먹었으므로, 게임 화면에서 아이템 오브젝트를 삭제합니다.
            Destroy(gameObject);
        }
    }
}
```

---

## 데미지 구역 스크립트

🔎 OnTriggerStay2D 함수는 Rigidbody 2D 컴포넌트를 가진 게임 오브젝트가 데미지 구역 안에 있는 동안 매 프레임마다 호출됩니다.

PlayerCharacter의 Rigidbody 2D component에서 Sleeping Mode 속성을 Never Sleep으로 변경.

- 원래 동작 (Sleep): 유니티는 물리 연산을 아끼려고 움직임이 멈춘 물체는 계산을 안 합니다. (잠자는 상태)
- 플레이어가 가만히 서 있으면, 그 자리에 있는 데미지 구역(불길, 가시 등)에 닿아도 충돌 계산이 안 되어 데미지를 안 입을 수 있어 플레이어가 멈춰 있어도 충돌 계산(Collision Detection)이 계속 일어나도록 설정이 필요합니다.

```csharp
using UnityEngine;

public class DamageZone : MonoBehaviour
{
    void OnTriggerStay2D(Collider2D other)
    {
        PlayerController controller = other.GetComponent<PlayerController>();

        if (controller != null)
        {
            controller.ChangeHealth(-1);
        }
    }
}
```

---
> 📌 **출처:** [[Unity Learn] 2D Adventure: Robot Repair][unity-link]

[unity-link]: https://learn.unity.com/course/2D-adventure-robot-repair?version=6.3