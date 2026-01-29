---
title: "NPC와 대화 후 게임 승리 구현 및 최종 코드"
toc: true
toc_sticky: true
date: 2026-01-26 20:06 +0900
categories:
  - unity-basic
  - 2d-robot-repair
tags:
  - C#
  - Unity
  - Basic
---

[Unit] Enhance your game > [Tutorial] Extra things to add

## 🔍 수정한 스크립트

### 플레이어

```csharp
using UnityEngine;
using UnityEngine.InputSystem;
using System;

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

    // Variables related to NPC
    private NonPlayerCharacter lastNonPlayerCharacter;
    public InputAction TalkAction; // 대화 키

    // Variables related to audio
    AudioSource audioSource;

    // 액션 이벤트
    public event Action OnTalkedToNPC;

    // Start is called once before the first execution of Update after the MonoBehaviour is created
    void Start()
    {
        MoveAction.Enable();
        LaunchAction.Enable();
        TalkAction.Enable(); // 대화키 가능
        rigidbody2d = GetComponent<Rigidbody2D>();
        animator = GetComponent<Animator>();
        currentHealth = maxHealth;
        audioSource = GetComponent<AudioSource>();

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

        // NPC 레이캐스트 감지 로직
        // Physics2D.Raycast(시작위치, 방향, 거리, 감지할 레이어)
        RaycastHit2D hit = Physics2D.Raycast(
            rigidbody2d.position + Vector2.up * 0.2f, // 시작점: 캐릭터 위치에서 위로 0.2 유닛 (발밑 감지 방지)
            moveDirection,                             // 방향: 현재 캐릭터가 움직이는(바라보는) 방향
            1.5f,                                      // 거리: 앞방향으로 1.5 유닛만큼만 레이저를 쏨
            LayerMask.GetMask("NPC")                   // 필터: "NPC" 레이어가 설정된 오브젝트만 충돌 처리
        );

        // 레이캐스트에 무언가 감지되었다면
        if (hit.collider != null)
        {
            // 충돌한 오브젝트에서 NonPlayerCharacter 스크립트 컴포넌트를 가져옴
            NonPlayerCharacter npc = hit.collider.GetComponent<NonPlayerCharacter>();

            npc.dialogueBubble.SetActive(true); // 해당 NPC의 대화 키 표시 말풍선을 화면에 표시
            lastNonPlayerCharacter = npc;       // 나중에 말풍선을 끄기 위해 현재 NPC 정보를 변수에 저장
            FindFriend(); // 친구를 찾는 추가 로직 실행
        }
        // 레이캐스트에 아무것도 감지되지 않았다면 (NPC 앞을 벗어났다면)
        else
        {
            // 이전에 감지했던 NPC 정보가 변수에 남아있는지 확인
            if (lastNonPlayerCharacter != null)
            {
                lastNonPlayerCharacter.dialogueBubble.SetActive(false); // 켜져 있던 대화 키 말풍선을 다시 끔
                lastNonPlayerCharacter = null; // NPC 저장 변수를 비움
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
            animator.SetTrigger("Hit"); // Hit(피격) 애니메이션을 딱 한 번만 실행
        }

        /* Mathf.Clamp 설명
           현재 체력에 받은 양을 더하되, 그 결과가 0보다 작아지거나 maxHealth보다 커지지 않게 '고정'합니다.
           예: 체력이 5인데 힐을 100 받아도 최대치인 5로 유지됨! 
        */
        currentHealth = Mathf.Clamp(currentHealth + amount, 0, maxHealth);
        UIHandler.instance.SetHealthValue(currentHealth / (float)maxHealth);

    }

    void Launch()
    {
        // projectilePrefab 복제, 현재 캐릭터 위치에서 위로 0.5만큼 살짝 위에서 총알이 나오게, Quaternion.identity는 회전 없음
        GameObject projectileObject = Instantiate(projectilePrefab, rigidbody2d.position + Vector2.up * 0.5f, Quaternion.identity);
        Projectile projectile = projectileObject.GetComponent<Projectile>();
        projectile.Launch(moveDirection, 300);
        animator.SetTrigger("Launch");
    }

    void FindFriend()
    {
        if (TalkAction.WasPressedThisFrame()) // 대화 키 누르면
        {
            OnTalkedToNPC?.Invoke(); // 알림
        }
    }

    public void PlaySound(AudioClip clip)
    {
        audioSource.PlayOneShot(clip); // 한 번만 소리 재생하는 함수
    }

}
``` 

### 게임 매니저

```csharp
using Beginner2D;
using UnityEngine;
using UnityEngine.SceneManagement; // 씬(레벨) 재시작을 위해 필요한 네임스페이스

public class GameManager : MonoBehaviour
{
    public PlayerController player; // 플레이어 참조 (체력 확인용)
    EnemyController[] enemies;      // 맵에 있는 모든 적을 저장할 배열
    public UIHandler uiHandler;     // UI 제어 스크립트 참조
    int enemiesFixed = 0; // 수정된 적의 수

    void Start()
    {
        // 게임 시작 시 씬에 배치된 모든 EnemyController를 찾아 배열에 저장
        // FindObjectsSortMode.None는 정렬 옵션 없음
        enemies = FindObjectsByType<EnemyController>(FindObjectsSortMode.None);
        
        foreach (var enemy in enemies)
        {
            enemy.OnFixed += HandleEnemyFixed; // 적이 가진 OnFixed라는 알림 벨에 HandleEnemyFixed라는 함수를 연결
        }
        uiHandler.SetCounter(0, enemies.Length);
        player.OnTalkedToNPC += HandlePlayerTalkedToNPC; // 함수 연결
    }

    void Update()
    {
        // 1. 패배 조건: 플레이어 체력이 0 이하인가?
        if (player.health <= 0)
        {
            uiHandler.DisplayLoseScreen(); // 패배 UI 출력 (아까 만든 페이드 효과 발동)
            Invoke(nameof(ReloadScene), 3f); // 3초 뒤에 ReloadScene 함수 실행
        }

    }

    // 모든 적의 상태를 체크하는 함수
    bool AllEnemiesFixed()
    {
        foreach (EnemyController enemy in enemies)
        {
            // 한 명이라도 여전히 고장(isBroken) 상태라면 false 반환
            if (enemy.isBroken) return false;
        }
        // 모든 적을 확인했는데 고장 난 적이 없다면 true 반환
        return true;
    }

    // 현재 씬을 다시 불러오는 함수 (게임 재시작)
    // SceneManager.GetActiveScene().name 현재 내가 플레이 중인 씬의 이름을 알아냄
    void ReloadScene()
    {
        SceneManager.LoadScene(SceneManager.GetActiveScene().name);
    }

    void HandleEnemyFixed()
    {
        enemiesFixed++;
        uiHandler.SetCounter(enemiesFixed, enemies.Length);
    }

    void HandlePlayerTalkedToNPC()
    {
        if (AllEnemiesFixed())
        {
            uiHandler.DisplayWinScreen();
            Invoke(nameof(ReloadScene), 3f);
        }
        else
        {
            UIHandler.instance.DisplayDialogue();
        }
    }

}using Beginner2D;
using UnityEngine;
using UnityEngine.SceneManagement; // 씬(레벨) 재시작을 위해 필요한 네임스페이스

public class GameManager : MonoBehaviour
{
    public PlayerController player; // 플레이어 참조 (체력 확인용)
    EnemyController[] enemies;      // 맵에 있는 모든 적을 저장할 배열
    public UIHandler uiHandler;     // UI 제어 스크립트 참조
    int enemiesFixed = 0; // 수정된 적의 수

    void Start()
    {
        // 게임 시작 시 씬에 배치된 모든 EnemyController를 찾아 배열에 저장
        // FindObjectsSortMode.None는 정렬 옵션 없음
        enemies = FindObjectsByType<EnemyController>(FindObjectsSortMode.None);
        
        foreach (var enemy in enemies)
        {
            enemy.OnFixed += HandleEnemyFixed; // 적이 가진 OnFixed라는 알림 벨에 HandleEnemyFixed라는 함수를 연결
        }
        uiHandler.SetCounter(0, enemies.Length);
        player.OnTalkedToNPC += HandlePlayerTalkedToNPC; // 함수 연결
    }

    void Update()
    {
        // 1. 패배 조건: 플레이어 체력이 0 이하인가?
        if (player.health <= 0)
        {
            uiHandler.DisplayLoseScreen(); // 패배 UI 출력 (아까 만든 페이드 효과 발동)
            Invoke(nameof(ReloadScene), 3f); // 3초 뒤에 ReloadScene 함수 실행
        }

    }

    // 모든 적의 상태를 체크하는 함수
    bool AllEnemiesFixed()
    {
        foreach (EnemyController enemy in enemies)
        {
            // 한 명이라도 여전히 고장(isBroken) 상태라면 false 반환
            if (enemy.isBroken) return false;
        }
        // 모든 적을 확인했는데 고장 난 적이 없다면 true 반환
        return true;
    }

    // 현재 씬을 다시 불러오는 함수 (게임 재시작)
    // SceneManager.GetActiveScene().name 현재 내가 플레이 중인 씬의 이름을 알아냄
    void ReloadScene()
    {
        SceneManager.LoadScene(SceneManager.GetActiveScene().name);
    }

    void HandleEnemyFixed()
    {
        enemiesFixed++;
        uiHandler.SetCounter(enemiesFixed, enemies.Length);
    }

    void HandlePlayerTalkedToNPC()
    {
        if (AllEnemiesFixed())
        {
            uiHandler.DisplayWinScreen();
            Invoke(nameof(ReloadScene), 3f);
        }
        else
        {
            UIHandler.instance.DisplayDialogue();
        }
    }

}
```

### UI

```csharp
using UnityEngine;
using UnityEngine.UIElements; // UI Toolkit 기능을 사용하기 위해 필요한 네임스페이스

public class UIHandler : MonoBehaviour
{
    
    private VisualElement m_Healthbar; // UI의 개별 요소(VisualElement)를 담을 변수
    public static UIHandler instance { get; private set; }

    // UI dialogue window variables
    public float displayTime = 4.0f; // 대화창이 떠 있을 시간
    private VisualElement m_NonPlayerDialogue; // NPC 대화창 UI 요소
    private float m_TimerDisplay; // 남은 표시 시간을 체크할 타이머

    // 승패 장면
    private VisualElement m_WinScreen;
    private VisualElement m_LoseScreen;

    // 로봇 카운터
    private Label m_RobotCounter;

    // Awake is called when the script instance is being loaded (in this situation, when the game scene loads)
    private void Awake()
    {
        instance = this;
    }

    // 객체가 생성된 후 첫 번째 Update 직전에 호출되는 함수
    void Start()
    {
        // 1. 현재 오브젝트에 붙어있는 UIDocument 컴포넌트를 가져옵니다.
        UIDocument uiDocument = GetComponent<UIDocument>();

        // 2. UI 레이아웃(UXML)에서 이름이 "HealthBar"인 요소를 찾아 변수에 할당합니다.
        // Q는 'Query'의 약자로, 특정 요소를 찾는 기능을 합니다.
        m_Healthbar = uiDocument.rootVisualElement.Q<VisualElement>("HealthBar");

        // 3. 시작할 때 체력바를 100%(1.0)로 초기화합니다.
        SetHealthValue(1.0f);

        // 이름이 "NPCDialogue"인 요소를 찾고, 처음에는 화면에서 숨김
        m_NonPlayerDialogue = uiDocument.rootVisualElement.Q<VisualElement>("NPCDialogue");
        m_NonPlayerDialogue.style.display = DisplayStyle.None;
        m_TimerDisplay = -1.0f; // 타이머 초기화 (-1은 작동하지 않는 상태를 의미)

        m_LoseScreen = uiDocument.rootVisualElement.Q<VisualElement>("LoseScreenContainer");
        m_WinScreen = uiDocument.rootVisualElement.Q<VisualElement>("WinScreenContainer");

        m_RobotCounter = uiDocument.rootVisualElement.Q<Label>("CounterLabel");

    }

    // 외부(예: Player 스크립트)에서 체력 수치를 변경할 때 호출하는 함수
    public void SetHealthValue(float percentage)
    {
        // m_Healthbar의 가로 길이(width) 스타일을 퍼센트 단위로 변경합니다.
        // 0.0 ~ 1.0 사이의 값을 받아 0% ~ 100%로 변환하여 적용합니다.
        // 퍼센트는 부모 너비 기준으로 작동합니다.
        m_Healthbar.style.width = Length.Percent(100 * percentage);
    }

    private void Update()
    {
        if (m_TimerDisplay > 0)
        {
            m_TimerDisplay -= Time.deltaTime;

            // 시간이 다 되면 대화창을 다시 숨김
            if (m_TimerDisplay < 0)
            {
                m_NonPlayerDialogue.style.display = DisplayStyle.None;
            }
        }
    }

    public void DisplayDialogue()
    {
        m_NonPlayerDialogue.style.display = DisplayStyle.Flex; // 대화창 보이기
        m_TimerDisplay = displayTime; // 타이머 리셋
    }

    public void DisplayWinScreen()
    {
        m_WinScreen.style.opacity = 1.0f;
    }

    public void DisplayLoseScreen()
    {
        m_LoseScreen.style.opacity = 1.0f;
    }

    // 로봇 숫자를 업데이트하는 함수
    public void SetCounter(int current, int enemies)
    {
        m_RobotCounter.text = $"{current} / {enemies}";
    }

}
```

### 적

```csharp
using UnityEngine;
using System;

public class EnemyController : MonoBehaviour
{
  
    // Public variables
    public float speed;
    public bool vertical;
    public float changeTime = 3.0f;
    public bool isBroken { get { return broken; } }
    public ParticleSystem smokeParticleEffect;
    public event Action OnFixed; // 적이 수정될 때 다른 스크립트에 알림을 보내는 데 사용

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
        smokeParticleEffect.Stop(); // 고친 후 연기 안 남
        OnFixed?.Invoke(); // 알림 실행
    }
}
```

### 체력 회복 아이템

```csharp
using UnityEngine;

// HealthCollectible 클래스: 아이템(체력 회복 아이템 등)에 부착하는 스크립트입니다.
public class HealthCollectible : MonoBehaviour
{

    public AudioClip collectedClip;

    // OnTriggerEnter2D: 이 오브젝트의 Trigger Collider에 다른 오브젝트가 들어왔을 때 호출됩니다.
    // 'other' 변수는 방금 부딪힌(겹쳐진) 상대방의 Collider 정보입니다.
    void OnTriggerEnter2D(Collider2D other)
    {
        // 1. 부딪힌 상대방(other)에게 'PlayerController'라는 스크립트가 있는지 확인합니다.
        PlayerController controller = other.GetComponent<PlayerController>();

        // 2. 만약 상대방에게 PlayerController가 있고 최대 체력이 아니면
        if (controller != null && controller.health < controller.maxHealth)
        {
            controller.PlaySound(collectedClip);
            controller.ChangeHealth(1);

            Destroy(gameObject);
        }
    }
}
``` 

### NPC

```csharp
using UnityEngine;

public class NonPlayerCharacter : MonoBehaviour
{
    public GameObject dialogueBubble;

    void Start()
    {
        dialogueBubble.SetActive(false); // 대화 키 말풍선 안 보이게
    }
}
```

### 수집물

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

### 데미지 구역

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