---
title: "Rule Tile 만들기"
toc: true
toc_sticky: true
date: 2026-01-26 19:27 +0900
categories:
  - unity-basic
  - 2d-robot-repair
tags:
  - C#
  - Unity
  - Basic
---

[Unit] Enhance your game > [Tutorial] Extra things to add 

## 🔍 고장난 로봇 수 표시 UI 구현

1. UI 에셋 생성 및 구성

- 파일 생성: Project 창 > Assets 우클릭 > Create > UI Toolkit > UI Document 선택 (RobotCounter로 이름 변경).
- 컨테이너 설정: RobotCounter.uxml 열기 > VisualElement 추가 > Background에 UICounterBar 스프라이트 할당 > Size 속성(Width/Height)을 100%로 설정.
- 텍스트 레이블 추가: VisualElement의 자식으로 Label 생성 > 이름을 CounterLabel로 변경 > 폰트 및 스타일 커스텀.
2. 메인 UI 통합 및 배치

- UI 병합: GameUI.uxml 열기 > RobotCounter를 자식 요소로 드래그하여 추가.
- 레이아웃 조정: RobotCounter 요소를 화면 오른쪽 상단으로 배치 및 정렬.
- [자세한 내용 4번 부터](https://learn.unity.com/course/2D-adventure-robot-repair/unit/game-ui-and-game-loop/tutorial/display-character-dialogue-using-raycasting?version=6.3)
 

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
    }

    void Update()
    {
        // 1. 패배 조건: 플레이어 체력이 0 이하인가?
        if (player.health <= 0)
        {
            uiHandler.DisplayLoseScreen(); // 패배 UI 출력 (아까 만든 페이드 효과 발동)
            Invoke(nameof(ReloadScene), 3f); // 3초 뒤에 ReloadScene 함수 실행
        }

        // 2. 승리 조건: 모든 적이 고쳐졌는가?
        if (AllEnemiesFixed())
        {
            uiHandler.DisplayWinScreen();  // 승리 UI 출력
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
}
```

---
> 📌 **출처:** [[Unity Learn] 2D Adventure: Robot Repair][unity-link]

[unity-link]: https://learn.unity.com/course/2D-adventure-robot-repair?version=6.3