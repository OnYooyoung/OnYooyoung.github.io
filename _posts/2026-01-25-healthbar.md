---
title: "체력 바 구현"
toc: true
toc_sticky: true
date: 2026-01-25 14:20 +0900
categories:
  - unity-basic
  - 2d-robot-repair
tags:
  - C#
  - Unity
  - Basic
---

## [Unit] Sidebar node imageunitGame UI and game loop > [Tutorial] Create Input Actions for player character movement, Display character health on the UI


1. UI 문서 생성 및 환경 설정
- 파일 생성: Project 창 > Assets 폴더 우클릭 > Create > UI Toolkit > UI Document (UXML) 선택 > 이름을 **"GameUI"**로 변경.
- 툴 열기: GameUI 파일 더블 클릭 (UI Builder 창 열림).
- 캔버스 설정: Hierarchy 패널에서 GameUI.uxml 선택 > Inspector 창 > Canvas Size 섹션 > Match Game View 체크 > 상단 메뉴 Save 클릭.

2. 체력 바 배경(Frame) 만들기
- 요소 추가: Library 패널 > Containers > VisualElement를 Hierarchy로 드래그.
- 이름 변경: Inspector 상단 Name에 "HealthBarBackground" 입력.
- 이미지 설정: Background 섹션 > Image를 Sprite로 변경 > UIHealthFrame 스프라이트 선택.
- 유연성 해제: Flex 섹션 > Grow를 0으로 설정.
- 크기 설정: Size 섹션 > Width와 Height를 **50%**로 설정.
- 모양 보정: Background 섹션 > Scale Mode를 'scale-to-fit' (가장 오른쪽 아이콘)으로 설정.

3. 실제 게임 화면에 연결
- 오브젝트 생성: 유니티 메인 Hierarchy 우클릭 > UI Toolkit > UI Document 선택.
- 에셋 연결: 생성된 오브젝트의 Inspector > Source Asset에 아까 만든 GameUI 연결.
- 해상도 대응: PanelSettings 에셋 클릭 > Scale Mode를 **'Scale With Screen Size'**로 변경 > Reference Resolution을 1920 x 1080으로 입력.

4. 초상화 및 체력 게이지 구성 (계층 구조 중요)
- 초상화 추가: Library > VisualElement를 HealthBarBackground 자식으로 드래그 > 이름 "CharacterPortrait" > 스프라이트 적용 > Position을 Absolute로 변경 후 위치 잡기.
- 게이지 컨테이너: Library > VisualElement를 HealthBarBackground 자식으로 드래그 > 이름 "HealthBarContainer" > Position을 Absolute로 변경 후 체력바 영역에 맞게 크기 조절. ( *주의 HealthBar 크기 변경 아님* )
- 실제 게이지(HealthBar): Library > VisualElement를 HealthBarContainer의 자식으로 드래그 > 이름 "HealthBar" > UIHealthBar 스프라이트 적용.

---

## UI 스크립트

```csharp
using UnityEngine;
using UnityEngine.UIElements; // UI Toolkit 기능을 사용하기 위해 필요한 네임스페이스

public class UIHandler : MonoBehaviour
{
    
    private VisualElement m_Healthbar; // UI의 개별 요소(VisualElement)를 담을 변수
    public static UIHandler instance { get; private set; }

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
    }

    // 외부(예: Player 스크립트)에서 체력 수치를 변경할 때 호출하는 함수
    public void SetHealthValue(float percentage)
    {
        // m_Healthbar의 가로 길이(width) 스타일을 퍼센트 단위로 변경합니다.
        // 0.0 ~ 1.0 사이의 값을 받아 0% ~ 100%로 변환하여 적용합니다.
        // 퍼센트는 부모 너비 기준으로 작동합니다.
        m_Healthbar.style.width = Length.Percent(100 * percentage);
    }
}
```

## 🔍 public static UIHandler instance { get; private set; }

1. get (가져오기)
- 의미: 외부에서 이 변수의 값을 읽을 수 있는지 결정합니다.
- 코드상의 효과: public get 상태(기본값)이므로, 게임 내 어떤 스크립트에서든 UIHandler.instance라고 쳐서 이 변수 안에 담긴 정보를 가져갈 수 있습니다.

2. private set (설정하기)

- 의미: 이 변수에 값을 집어넣거나 수정할 수 있는 권한을 결정합니다.
- 코드상의 효과: 앞의 private 때문에, 오직 UIHandler 클래스 내부에서만 값을 수정할 수 있습니다. 다른 스크립트에서 UIHandler.instance = null; 같은 식으로 값을 조작하려고 하면 에러가 납니다.

---

## 플레이어 스크립트

```charp
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
  	public int health { get { return currentHealth; }}

  	// Variables related to temporary invincibility
  	public float timeInvincible = 2.0f;
  	bool isInvincible;
	float damageCooldown;

  	// Variables related to animation
   	Animator animator;
   	Vector2 moveDirection = new Vector2(1, 0);

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

		if (!Mathf.Approximately(move.x, 0.0f) || !Mathf.Approximately(move.y, 0.0f))
       	{
           		moveDirection.Set(move.x, move.y);
           		moveDirection.Normalize();
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

       	if (LaunchAction.WasPressedThisFrame())
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

  	public void ChangeHealth (int amount)
  	{
     	if (amount < 0)
       	{
       		if (isInvincible)
			{
           		return;
 			}        
            isInvincible = true;
           	damageCooldown = timeInvincible;
    		animator.SetTrigger("Hit");
       	}

     	currentHealth = Mathf.Clamp(currentHealth + amount, 0, maxHealth);
     	UIHandler.instance.SetHealthValue(currentHealth / (float)maxHealth); // 추가한 부분
  	}

void Launch()
   	{
       	GameObject projectileObject = Instantiate(projectilePrefab, rigidbody2d.position + Vector2.up * 0.5f, Quaternion.identity);
       	Projectile projectile = projectileObject.GetComponent<Projectile>();
       	projectile.Launch(moveDirection, 300);

       	animator.SetTrigger("Launch");
   	}
}
```

---
> 📌 **출처:** [[Unity Learn] 2D Adventure: Robot Repair][unity-link]

[unity-link]: https://learn.unity.com/course/2D-adventure-robot-repair?version=6.3