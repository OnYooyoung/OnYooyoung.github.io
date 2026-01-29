---
title: "발사체 구현"
toc: true
toc_sticky: true
date: 2026-01-24 21:55 +0900
categories:
  - unity-basic
  - 2d-robot-repair
tags:
  - C#
  - Unity
  - Basic
---

[Unit] Characters and interaction mechanics > [Tutorial] Implement projectiles for the player

## 발사체 스크립트

```csharp
using UnityEngine;

public class Projectile : MonoBehaviour
{
    Rigidbody2D rigidbody2d;

    // Awake is called when the Projectile GameObject is instantiated
    void Awake()
    {
        rigidbody2d = GetComponent<Rigidbody2D>();
    }

    void Update()
    {

    }

    public void Launch(Vector2 direction, float force)
    {
        rigidbody2d.AddForce(direction * force);
    }

    void OnTriggerEnter2D(Collider2D other)
    {
        Debug.Log("Projectile collision with " + other.gameObject);
        Destroy(gameObject);
    }
}
```

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
    int currentHealth;
    // get: 데이터를 요청받았을 때 실행되는 '입구(함수)'
    // return: 그 입구로 들어온 사람에게 들려보낼 '결과물'
    public int health { get { return currentHealth; } }

    // Variables related to temporary invincibility 무적
    public float timeInvincible = 2.0f;
    bool isInvincible;
    float damageCooldown; // 무적 쿨타임

    // Variables related to animation
    Animator animator;
    Vector2 moveDirection = new Vector2(1, 0); // (X, Y)

    // Variables related to projectiles
    public GameObject projectilePrefab;
    public InputAction LaunchAction;

    // Start is called once before the first execution of Update after the MonoBehaviour is created
    void Start()
    {
        MoveAction.Enable();
        LaunchAction.Enable();
        rigidbody2d = GetComponent<Rigidbody2D>();
        animator = GetComponent<Animator>();
        currentHealth = maxHealth;
    }

    // Update is called once per frame
    void Update()
    {
        move = MoveAction.ReadValue<Vector2>();

        // 플레이어가 움직이고 있다면 (0이 아니라면), 부동소수점문제 해결 위해 approximately 사용
        if (!Mathf.Approximately(move.x, 0.0f) || !Mathf.Approximately(move.y, 0.0f))
        {
            moveDirection.Set(move.x, move.y); // 현재 방향을 기억
            moveDirection.Normalize(); // 길이 1로 정규화
        }

        animator.SetFloat("Look X", moveDirection.x);
        animator.SetFloat("Look Y", moveDirection.y);
        animator.SetFloat("Speed", move.magnitude);

        if (isInvincible)
        {
            damageCooldown -= Time.deltaTime;
            if (damageCooldown < 0)
            {
                isInvincible = false;
            }
        }

        if (LaunchAction.WasPressedThisFrame()) // 발사 버튼 클릭 시
        {
            Launch();
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
            animator.SetTrigger("Hit"); // Hit(피격) 애니메이션을 딱 한 번만 실행
        }

        /* Mathf.Clamp 설명
           현재 체력에 받은 양을 더하되, 그 결과가 0보다 작아지거나 maxHealth보다 커지지 않게 '고정'합니다.
           예: 체력이 5인데 힐을 100 받아도 최대치인 5로 유지됨! 
        */
        currentHealth = Mathf.Clamp(currentHealth + amount, 0, maxHealth);
        Debug.Log(currentHealth + "/" + maxHealth);

    }

    void Launch()
    {
        // projectilePrefab 복제, 현재 캐릭터 위치에서 위로 0.5만큼 살짝 위에서 총알이 나오게, Quaternion.identity는 회전 없음
        GameObject projectileObject = Instantiate(projectilePrefab, rigidbody2d.position + Vector2.up * 0.5f, Quaternion.identity);
        Projectile projectile = projectileObject.GetComponent<Projectile>();
        projectile.Launch(moveDirection, 300);
        animator.SetTrigger("Launch");
    }
}
``` 

1. 에디터 설정 (연결 고리 만들기)
- 바인딩 추가: 플레이어 인스펙터 창에서 LaunchAction 옆의 + 버튼을 눌러 Binding을 생성합니다.
- 버튼 타입: 이 동작은 누르는 동작이므로 속성을 Button으로 맞춥니다.
- 키 할당 (C키): 'Path' 설정에서 직접 C를 찾거나, Listen 버튼을 누른 뒤 키보드 C를 눌러서 간편하게 등록합니다.
2. 레이어 정의 (이름표 만들기)
- 유니티의 Layer Manager(레이어 추가...)를 열고, 비어있는 칸에 **"Character"**와 **"Projectiles"**라는 이름을 입력합니다.
- 이제 우리 게임 세상에는 '캐릭터' 그룹과 '발사체' 그룹이라는 분류가 생긴 겁니다.
3. 플레이어에 레이어 할당
- PlayerCharacter 프리팹을 열어, 오른쪽 인스펙터 창 맨 위 Layer 항목을 방금 만든 Character로 바꿔줍니다.
4. 총알에 레이어 할당
- 프로젝트 창에 있는 Projectile(총알) 프리팹도 똑같이 선택해서 레이어를 Projectiles로 바꿔줍니다.
5. 두 레이어가 서로 충돌하지 않도록 프로젝트를 설정
- Edit > Project Settings > Physics 2D > Layer Collision Matrix tab

---

[Unit] Characters and interaction mechanics > [Tutorial] Configure projectiles to affect the enemy

## 발사체가 적에 닿았을 때 추가 구현

적 스크립트

```charp
using UnityEngine;

public class EnemyController : MonoBehaviour
{
    // Public variables
    public float speed;
    public bool vertical;
    public float changeTime = 3.0f;

    // Private variables
    Rigidbody2D rigidbody2d;
    Animator animator;
    float timer;
    int direction = 1;
    bool broken = true;

    // Start is called once before the first execution of Update after the MonoBehaviour is created
    void Start()
    {
        rigidbody2d = GetComponent<Rigidbody2D>();
        animator = GetComponent<Animator>();
        timer = changeTime;
    }

    // Update is called every frame
    void Update()
    {
        timer -= Time.deltaTime;

        if (timer < 0)
        {
            direction = -direction;
            timer = changeTime;
        }
    }

    // FixedUpdate has the same call rate as the physics system
    void FixedUpdate()
    {
        if (!broken)
        {
            return;
        }

        Vector2 position = rigidbody2d.position;

        if (vertical)
        {
            position.y = position.y + speed * direction * Time.deltaTime;
            animator.SetFloat("Move X", 0); // "Move X"라는 변수 값을 0으로 만듦
            animator.SetFloat("Move Y", direction);
        }
        else
        {
            position.x = position.x + speed * direction * Time.deltaTime;
            animator.SetFloat("Move X", direction);
            animator.SetFloat("Move Y", 0);
        }
        
        rigidbody2d.MovePosition(position);
    }

    void OnTriggerEnter2D(Collider2D other)
    {
        PlayerController player = other.gameObject.GetComponent<PlayerController>();

        if (player != null)
        {
            player.ChangeHealth(-1);
        }
    }

    public void Fix()
    {
        broken = false;
        // 적 게임 오브젝트를 물리 시스템의 충돌 시뮬레이션에서 제외.
        // 투사체가 더 이상 적과 충돌하지 않으며 적 캐릭터에게 피해를 못 주게 됨.
        rigidbody2d.simulated = false;
        animator.SetTrigger("Fixed");
    }
}
```

## 발사체 스크립트

```csharp
using UnityEngine;

public class Projectile : MonoBehaviour
{
    Rigidbody2D rigidbody2d;

    // Awake is called when the Projectile GameObject is instantiated
    void Awake()
    {
        rigidbody2d = GetComponent<Rigidbody2D>();
    }

    void Update()
    {
        // position은 원점에서 투사체 게임 오브젝트의 위치까지의 벡터
        // magnitude는 해당 벡터의 길이
        // 거리가 100 보다 크면 Projectile GameObject 가 파괴됩니다.
        if (transform.position.magnitude > 100.0f)
        {
            Destroy(gameObject);
        }
    }

    public void Launch(Vector2 direction, float force)
    {
        rigidbody2d.AddForce(direction * force);
    }

    void OnTriggerEnter2D(Collider2D other) // 트리거와의 충돌 처리
    {
        EnemyController enemy = other.GetComponent<EnemyController>();

        if (enemy != null)
        {
            enemy.Fix();
        }

        Destroy(gameObject);
    }

    void OnCollisionEnter2D(Collision2D collision)
    {
        Destroy(gameObject);
    }
}
```

---
> 📌 **출처:** [[Unity Learn] 2D Adventure: Robot Repair][unity-link]

[unity-link]: https://learn.unity.com/course/2D-adventure-robot-repair?version=6.3