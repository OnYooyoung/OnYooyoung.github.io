---
title: "raycasting과 캐릭터 대화 구현"
toc: true
toc_sticky: true
date: 2026-01-25 17:31 +0900
categories:
  - unity-basic
  - 2d-robot-repair
tags:
  - C#
  - Unity
  - Basic
---

## [Unit] Game UI and game loop > [Tutorial] Display character dialogue using raycasting

1. 기본 배경(Background) 만들기

- 에셋 생성: 프로젝트 창에서 새 **UI Document(UXML)**를 만들고 이름을 NPCDialogue로 지정한 뒤 더블클릭합니다.
- 요소 추가: Hierarchy(계층 구조) 창에 VisualElement를 추가하고 이름을 Background로 바꿉니다.
- 이미지 넣기: Inspector 창의 Background > Image에서 UIDialogueBox 스프라이트를 선택합니다.
- 투명도 조절: Background > Color의 A(Alpha) 값을 0으로 설정합니다. (스프라이트 자체 이미지만 보이게 함)
- 꽉 채우기: Size 섹션에서 너비(Width)와 높이(Height)를 모두 **100%**로 설정합니다.
2. 대화 텍스트(Label) 넣기

- 텍스트 추가: Library 창의 Controls 탭에서 Label을 드래그하여 Background의 자식으로 넣습니다.
- 크기 맞춤: Label의 너비와 높이도 **100%**로 설정합니다.
- 여백 주기 (Padding):
- Spacing 섹션에서 Margin은 모두 0, Padding은 모두 25px로 설정합니다. (글자가 상자 테두리에 딱 붙지 않게 함)
- 텍스트 스타일:
    - Text 섹션에서 글꼴, 크기, 색상을 원하는 대로 바꿉니다.
    - Wrap(줄 바꿈) 속성을 Normal로 설정합니다. (글이 길어지면 자동으로 다음 줄로 넘어가게 함)
- 내용 입력: Attributes > Text 칸에 플레이어에게 보여줄 대화 내용을 입력합니다.
 
---

1. UI 빌더 설정 (화면 배치)

- 에셋 열기: GameUI 에셋을 더블클릭하여 UI Builder 창을 엽니다.
- 드래그: Library 창의 Project 탭에서 NPCDialogue 에셋을 찾은 뒤, 왼쪽 Hierarchy 창으로 끌어다 놓습니다.
- 이름 변경: Hierarchy에 들어온 요소의 이름을 "NPCDialogue"로 바꿉니다.
- 위치 고정: Inspector 창의 Position 섹션에서 Position Mode를 Absolute로 선택합니다.
- 좌표 입력: Top: 70% / Right: 10% / Bottom: 0% / Left: 10% 로 설정하여 화면 하단에 맞춥니다.
2. NPC 머리 위 말풍선(X키 아이콘) 설정

- 스프라이트 배치: DialogueBox 스프라이트를 Scene 뷰로 끌어와서 NPC 머리 위 적절한 위치에 둡니다.
- 계층 구조: 이 오브젝트를 **NPC 오브젝트의 자식(Child)**으로 드래그하여 종속시킵니다.
- 레이어 순서: Inspector의 Sprite Renderer에서 Order in Layer를 2로 수정합니다. (캐릭터 뒤로 가려지는 것 방지)
- 이름: 구분을 위해 이 오브젝트 이름을 **"DialogueBubble"**로 바꿉니다.
3. 인스펙터 컴포넌트 연결

- Player 오브젝트:
  - PlayerController 컴포넌트의 Talk Action 칸에 X 키를 바인딩합니다.
  - Projectile Prefab 칸에 만들어둔 투사체 프리팹을 넣습니다.
- NPC 오브젝트:
  - NonPlayerCharacter 스크립트의 Dialogue Bubble 빈칸에, 방금 만든 자식 오브젝트(DialogueBubble)를 드래그해서 넣습니다.

---

## 플레이어

```charp
using Beginner2D;
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

    // Variables related to NPC
    private NonPlayerCharacter lastNonPlayerCharacter;
    public InputAction TalkAction; // 대화 키

    // Start is called once before the first execution of Update after the MonoBehaviour is created
    void Start()
    {
        MoveAction.Enable();
        LaunchAction.Enable();
        TalkAction.Enable(); // 대화키 가능
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
            UIHandler.instance.DisplayDialogue();
        }
    }

}
```

## UI

```charp
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
}
```

## NPC

```charp
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

---
> 📌 **출처:** [[Unity Learn] 2D Adventure: Robot Repair][unity-link]

[unity-link]: https://learn.unity.com/course/2D-adventure-robot-repair?version=6.3