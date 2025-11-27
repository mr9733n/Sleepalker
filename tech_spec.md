# SLEEPWALKER — ТЕХНИЧЕСКАЯ СПЕЦИФИКАЦИЯ

**Версия:** 1.0  
**Для:** Программисты, технические лиды  
**Движок:** Unity 2022 LTS+  
**Язык:** C#

---

## 1. ОБЗОР АРХИТЕКТУРЫ

### 1.1 Высокоуровневая схема

```
┌─────────────────────────────────────────────────────────────┐
│                      GAME MANAGER                            │
│  (синглтон, точка входа, управление состояниями игры)       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │    LEVEL     │  │    PLAYER    │  │     NPC      │       │
│  │   MANAGER    │  │   SYSTEMS    │  │   SYSTEMS    │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                 │                 │                │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐       │
│  │ Level Loader │  │ Movement     │  │ NPC Manager  │       │
│  │ Spawn System │  │ Inventory    │  │ AI Director  │       │
│  │ Objective    │  │ Interaction  │  │ Perception   │       │
│  │ Manager      │  │ Shame System │  │ Behavior     │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  DETECTION   │  │   CLOTHING   │  │     UI       │       │
│  │   SYSTEM     │  │   SYSTEM     │  │   SYSTEM     │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                 │                 │                │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐       │
│  │ Vision Cones │  │ Item Manager │  │ HUD          │       │
│  │ Alert System │  │ Outfit Calc  │  │ Menus        │       │
│  │ Camera Detect│  │ Disguise     │  │ Notifications│       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                     DATA LAYER                               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│  │ Save System │ │ Config/     │ │ Analytics   │            │
│  │             │ │ ScriptableO │ │ (optional)  │            │
│  └─────────────┘ └─────────────┘ └─────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Принципы архитектуры

- **Composition over Inheritance** — компоненты вместо глубокой иерархии
- **ScriptableObjects для данных** — конфигурации NPC, предметов, уровней
- **Event-driven communication** — системы общаются через события, минимум прямых зависимостей
- **Separation of Concerns** — каждая система отвечает за одну область

### 1.3 Ключевые паттерны

| Паттерн | Применение |
|---------|------------|
| Singleton | GameManager, AudioManager |
| Observer | События (обнаружение, смена одежды) |
| State Machine | AI NPC, состояния игры |
| Object Pool | NPC, эффекты, UI-элементы |
| Strategy | Типы реакций NPC |
| Command | Система взаимодействий |

---

## 2. ОСНОВНЫЕ СИСТЕМЫ

### 2.1 Game Manager

```csharp
public enum GameState
{
    MainMenu,
    Loading,
    Sleeping,      // Переход между уровнями
    Waking,        // Кат-сцена пробуждения
    Playing,       // Основной геймплей
    Paused,
    LevelComplete,
    GameOver
}

public class GameManager : MonoBehaviour
{
    public static GameManager Instance { get; private set; }
    
    public GameState CurrentState { get; private set; }
    
    // События
    public event Action<GameState> OnStateChanged;
    public event Action OnLevelStart;
    public event Action<LevelResult> OnLevelEnd;
    
    // Ссылки на менеджеры
    public LevelManager LevelManager { get; private set; }
    public PlayerController Player { get; private set; }
    public NPCManager NPCManager { get; private set; }
    public UIManager UIManager { get; private set; }
    
    public void ChangeState(GameState newState);
    public void StartLevel(LevelData level);
    public void EndLevel(LevelResult result);
    public void PauseGame();
    public void ResumeGame();
}
```

### 2.2 Player Systems

#### 2.2.1 Player Controller

```csharp
public class PlayerController : MonoBehaviour
{
    [Header("Movement")]
    [SerializeField] private float walkSpeed = 3f;
    [SerializeField] private float runSpeed = 6f;
    [SerializeField] private float sneakSpeed = 1.5f;
    
    [Header("State")]
    public PlayerMovementState MovementState { get; private set; }
    public bool IsHidden { get; private set; }
    public bool IsInteracting { get; private set; }
    
    // Компоненты
    private CharacterController controller;
    private PlayerInventory inventory;
    private PlayerInteraction interaction;
    private ShameSystem shame;
    private ClothingSystem clothing;
    
    // События
    public event Action<PlayerMovementState> OnMovementStateChanged;
    public event Action OnHidingStateChanged;
}

public enum PlayerMovementState
{
    Idle,
    Walking,
    Running,
    Sneaking,
    Hiding,
    Interacting
}
```

#### 2.2.2 Shame System (Ключевая механика)

```csharp
[System.Serializable]
public class ShameConfig
{
    public float maxShame = 100f;
    public float decayRate = 1f;              // В секунду, когда не видят
    public float decayDelay = 3f;             // Задержка перед началом восстановления
    
    [Header("Modifiers")]
    public float seenNakedBase = 20f;
    public float caughtOnCameraBase = 15f;
    public float caughtOnPhoneBase = 25f;
    public float failedLieBase = 10f;
    public float witnessScreamBase = 5f;
}

public class ShameSystem : MonoBehaviour
{
    [SerializeField] private ShameConfig config;
    
    public float CurrentShame { get; private set; }
    public float ShamePercent => CurrentShame / config.maxShame;
    public ShameLevel CurrentLevel { get; private set; }
    
    // События
    public event Action<float> OnShameChanged;
    public event Action<ShameLevel> OnShameLevelChanged;
    public event Action OnShameMaxed;  // Game Over
    
    public void AddShame(float amount, ShameSource source);
    public void ReduceShame(float amount);
    public void ApplyClothingBonus(ClothingData clothing);
    
    private void Update()
    {
        // Естественное восстановление, если не видят
        if (!isBeingWatched && timeSinceLastSeen > config.decayDelay)
        {
            ReduceShame(config.decayRate * Time.deltaTime);
        }
    }
}

public enum ShameLevel
{
    Calm,        // 0-25%
    Nervous,     // 25-50%
    Panicking,   // 50-75%
    Critical     // 75-100%
}

public enum ShameSource
{
    SeenByNPC,
    CaughtOnCamera,
    CaughtOnPhone,
    FailedSocialCheck,
    WitnessScream,
    Other
}
```

#### 2.2.3 Clothing System

```csharp
[CreateAssetMenu(fileName = "ClothingItem", menuName = "Sleepwalker/Clothing")]
public class ClothingData : ScriptableObject
{
    public string itemName;
    public ClothingCategory category;
    public Sprite icon;
    public GameObject prefab;
    
    [Header("Stats")]
    [Range(0f, 1f)] public float coverageLevel;     // 0 = голый, 1 = полностью одет
    [Range(0f, 1f)] public float visibilityReduction;
    public float shameReduction;
    
    [Header("Durability")]
    public bool hasLimitedUses;
    public int maxUses;
    public float duration;  // Для временных предметов
    
    [Header("Special")]
    public bool isDisguise;
    public NPCType[] disguiseEffectiveAgainst;
    public string[] specialAccess;  // "staff_only", "vip", etc.
}

public enum ClothingCategory
{
    None,
    Improvised,    // Газета, пакет
    Temporary,     // Полотенце, скатерть
    Partial,       // Шорты, футболка
    Full,          // Полный комплект
    Disguise       // Униформа
}

public class ClothingSystem : MonoBehaviour
{
    public ClothingData CurrentClothing { get; private set; }
    public float TotalCoverage { get; private set; }
    
    public event Action<ClothingData> OnClothingChanged;
    public event Action OnClothingBroken;
    
    public void EquipClothing(ClothingData clothing);
    public void RemoveClothing();
    public void UseClothing();  // Для предметов с ограниченным использованием
    
    public float GetVisibilityModifier();
    public bool HasAccessTo(string accessType);
}
```

#### 2.2.4 Player Inventory

```csharp
public class PlayerInventory : MonoBehaviour
{
    [SerializeField] private int maxSlots = 3;
    
    private List<InventoryItem> items = new();
    
    public event Action<InventoryItem> OnItemAdded;
    public event Action<InventoryItem> OnItemRemoved;
    public event Action<InventoryItem> OnItemUsed;
    
    public bool TryAddItem(InventoryItem item);
    public void RemoveItem(InventoryItem item);
    public void UseItem(int slotIndex);
    public bool HasItem(ItemType type);
}

[System.Serializable]
public class InventoryItem
{
    public ItemData data;
    public int currentUses;
}

[CreateAssetMenu(fileName = "Item", menuName = "Sleepwalker/Item")]
public class ItemData : ScriptableObject
{
    public string itemName;
    public ItemType type;
    public Sprite icon;
    public GameObject worldPrefab;
    
    public bool isConsumable;
    public int maxUses;
    
    public ItemEffect effect;
}

public enum ItemType
{
    Clothing,
    Distraction,   // Монетка, камень
    Tool,          // Ключ-карта, отмычка
    Special
}
```

### 2.3 NPC Systems

#### 2.3.1 NPC Data Structure

```csharp
[CreateAssetMenu(fileName = "NPCType", menuName = "Sleepwalker/NPC Type")]
public class NPCTypeData : ScriptableObject
{
    public string typeName;
    public NPCCategory category;
    
    [Header("Appearance")]
    public GameObject[] prefabVariants;
    public RuntimeAnimatorController animator;
    
    [Header("Perception")]
    [Range(5f, 25f)] public float viewDistance = 15f;
    [Range(30f, 180f)] public float viewAngle = 90f;
    [Range(0.5f, 3f)] public float detectionSpeed = 1f;
    
    [Header("Personality")]
    [Range(0f, 1f)] public float empathy = 0.5f;
    [Range(0f, 1f)] public float anxiety = 0.5f;
    [Range(0f, 1f)] public float tolerance = 0.5f;
    
    [Header("Behavior")]
    public NPCReactionType primaryReaction;
    public NPCReactionType[] possibleReactions;
    public float reactionDelay;
    
    [Header("Threat Level")]
    public ThreatType threatType;
    public float threatSeverity;
}

public enum NPCCategory
{
    Civilian,
    Staff,
    Security,
    Authority,
    Child,
    Helper
}

public enum NPCReactionType
{
    Ignore,
    Stare,
    Laugh,
    Scream,
    Flee,
    Help,
    CallSecurity,
    RecordOnPhone,
    Confront,
    Attack
}

public enum ThreatType
{
    None,
    Social,      // Смущение
    Recording,   // Мем-угроза
    Security,    // Вызов охраны
    Arrest       // Game over
}
```

#### 2.3.2 NPC Controller

```csharp
public class NPCController : MonoBehaviour
{
    [SerializeField] private NPCTypeData typeData;
    
    // Компоненты
    private NPCPerception perception;
    private NPCMovement movement;
    private NPCStateMachine stateMachine;
    private NPCAnimator animator;
    
    // Состояние
    public NPCState CurrentState => stateMachine.CurrentState;
    public bool HasSeenPlayer { get; private set; }
    public float Alertness { get; private set; }
    
    // События
    public event Action<NPCState> OnStateChanged;
    public event Action OnPlayerDetected;
    public event Action OnPlayerLost;
    
    public void Initialize(NPCSpawnData spawnData);
    public void SetRoute(PatrolRoute route);
    public void ReactToPlayer(PlayerController player);
    public void ForceState(NPCState state);
}
```

#### 2.3.3 NPC Perception System

```csharp
public class NPCPerception : MonoBehaviour
{
    [SerializeField] private NPCTypeData typeData;
    
    // Кэш
    private PlayerController player;
    private float currentDetection = 0f;
    
    public bool CanSeePlayer { get; private set; }
    public bool IsPlayerDetected => currentDetection >= 1f;
    public float DetectionProgress => currentDetection;
    
    public event Action<float> OnDetectionChanged;
    public event Action OnPlayerFullyDetected;
    
    private void Update()
    {
        UpdateVision();
        UpdateDetection();
    }
    
    private void UpdateVision()
    {
        if (player == null) return;
        
        Vector3 dirToPlayer = player.transform.position - transform.position;
        float distance = dirToPlayer.magnitude;
        
        // Проверка дистанции
        if (distance > typeData.viewDistance)
        {
            CanSeePlayer = false;
            return;
        }
        
        // Проверка угла
        float angle = Vector3.Angle(transform.forward, dirToPlayer);
        if (angle > typeData.viewAngle / 2f)
        {
            CanSeePlayer = false;
            return;
        }
        
        // Raycast для препятствий
        if (Physics.Raycast(transform.position + Vector3.up * 1.5f, 
                           dirToPlayer.normalized, 
                           out RaycastHit hit, 
                           distance,
                           obstacleMask))
        {
            CanSeePlayer = hit.collider.CompareTag("Player");
        }
        else
        {
            CanSeePlayer = true;
        }
    }
    
    private void UpdateDetection()
    {
        if (CanSeePlayer)
        {
            float visibilityMod = player.GetComponent<ClothingSystem>().GetVisibilityModifier();
            float distanceMod = 1f - (GetDistanceToPlayer() / typeData.viewDistance);
            float movementMod = player.MovementState == PlayerMovementState.Running ? 1.5f : 1f;
            
            float detectionRate = typeData.detectionSpeed * visibilityMod * distanceMod * movementMod;
            currentDetection += detectionRate * Time.deltaTime;
            
            if (currentDetection >= 1f && !wasDetected)
            {
                wasDetected = true;
                OnPlayerFullyDetected?.Invoke();
            }
        }
        else
        {
            // Медленное забывание
            currentDetection = Mathf.Max(0, currentDetection - forgetRate * Time.deltaTime);
            if (currentDetection < 0.5f) wasDetected = false;
        }
        
        OnDetectionChanged?.Invoke(currentDetection);
    }
}
```

#### 2.3.4 NPC State Machine

```csharp
public enum NPCState
{
    Idle,
    Patrolling,
    Suspicious,
    Investigating,
    Alerted,
    Reacting,
    Fleeing,
    Helping,
    Recording,
    CallingBackup,
    Returning
}

public class NPCStateMachine : MonoBehaviour
{
    public NPCState CurrentState { get; private set; }
    
    private Dictionary<NPCState, INPCState> states;
    private INPCState currentStateHandler;
    
    public event Action<NPCState, NPCState> OnStateChanged;
    
    private void Awake()
    {
        states = new Dictionary<NPCState, INPCState>
        {
            { NPCState.Idle, new NPCIdleState(this) },
            { NPCState.Patrolling, new NPCPatrolState(this) },
            { NPCState.Suspicious, new NPCSuspiciousState(this) },
            { NPCState.Investigating, new NPCInvestigateState(this) },
            { NPCState.Alerted, new NPCAlertedState(this) },
            { NPCState.Reacting, new NPCReactingState(this) },
            { NPCState.Fleeing, new NPCFleeingState(this) },
            { NPCState.Helping, new NPCHelpingState(this) },
            { NPCState.Recording, new NPCRecordingState(this) },
            { NPCState.CallingBackup, new NPCCallingBackupState(this) },
            { NPCState.Returning, new NPCReturningState(this) }
        };
    }
    
    public void TransitionTo(NPCState newState)
    {
        if (CurrentState == newState) return;
        
        currentStateHandler?.Exit();
        var oldState = CurrentState;
        CurrentState = newState;
        currentStateHandler = states[newState];
        currentStateHandler.Enter();
        
        OnStateChanged?.Invoke(oldState, newState);
    }
}

public interface INPCState
{
    void Enter();
    void Update();
    void Exit();
}
```

### 2.4 Detection System

```csharp
public class DetectionSystem : MonoBehaviour
{
    public static DetectionSystem Instance { get; private set; }
    
    [SerializeField] private DetectionConfig config;
    
    // Глобальный уровень тревоги
    public AlertLevel CurrentAlertLevel { get; private set; }
    public int ActiveWitnesses { get; private set; }
    
    public event Action<AlertLevel> OnAlertLevelChanged;
    public event Action<NPCController> OnNewWitness;
    public event Action OnAllClear;
    
    public void RegisterDetection(NPCController npc, DetectionType type);
    public void RegisterCameraDetection(SecurityCamera camera);
    public void ClearWitness(NPCController npc);
    
    public float GetGlobalAlertModifier();
}

public enum AlertLevel
{
    None,        // Никто не видел
    Low,         // 1-2 свидетеля, не уверены
    Medium,      // Несколько свидетелей, ищут
    High,        // Охрана активна
    Critical     // Полиция вызвана
}
```

### 2.5 Interaction System

```csharp
public class InteractionSystem : MonoBehaviour
{
    [SerializeField] private float interactionRadius = 2f;
    [SerializeField] private LayerMask interactableMask;
    
    private IInteractable currentTarget;
    
    public IInteractable CurrentTarget => currentTarget;
    public bool CanInteract => currentTarget != null;
    
    public event Action<IInteractable> OnTargetChanged;
    public event Action<IInteractable, InteractionResult> OnInteractionComplete;
    
    private void Update()
    {
        ScanForInteractables();
        HandleInput();
    }
    
    public void Interact()
    {
        if (currentTarget == null) return;
        
        var result = currentTarget.Interact(GetComponent<PlayerController>());
        OnInteractionComplete?.Invoke(currentTarget, result);
    }
}

public interface IInteractable
{
    string InteractionPrompt { get; }
    bool CanInteract(PlayerController player);
    InteractionResult Interact(PlayerController player);
    void Highlight(bool active);
}

public class InteractionResult
{
    public bool success;
    public string message;
    public InteractionOutcome outcome;
}

public enum InteractionOutcome
{
    Success,
    Failed,
    PartialSuccess,
    Interrupted
}
```

### 2.6 Social Interaction System

```csharp
[CreateAssetMenu(fileName = "SocialOption", menuName = "Sleepwalker/Social Option")]
public class SocialOptionData : ScriptableObject
{
    public string optionText;
    public SocialOptionType type;
    public Sprite icon;
    
    [Header("Requirements")]
    public int requiredCharismaLevel;
    public ClothingCategory minClothing;
    
    [Header("Checks")]
    public int baseDifficulty;
    public StatType checkStat;
    
    [Header("Outcomes")]
    public SocialOutcome successOutcome;
    public SocialOutcome failureOutcome;
}

public enum SocialOptionType
{
    Lie,           // "Это костюм для косплея!"
    AskHelp,       // "Пожалуйста, помогите..."
    Distract,      // "Смотрите, там!"
    Intimidate,    // "Вы ничего не видели."
    Charm,         // "Прекрасная погода, не правда ли?"
    Shock,         // *Молча смотреть*
    Explain        // "У меня заболевание..."
}

public class SocialInteractionManager : MonoBehaviour
{
    public void StartSocialEncounter(NPCController npc)
    {
        // Получить доступные опции на основе:
        // - Типа NPC
        // - Навыков игрока
        // - Текущей одежды
        // - Контекста ситуации
        
        var options = GetAvailableOptions(npc);
        UIManager.Instance.ShowSocialOptions(options, OnOptionSelected);
    }
    
    private void OnOptionSelected(SocialOptionData option, NPCController npc)
    {
        // Рассчитать успех
        int roll = Random.Range(1, 21);  // d20
        int playerBonus = GetPlayerBonus(option.checkStat);
        int clothingBonus = GetClothingBonus();
        int npcModifier = GetNPCModifier(npc, option.type);
        
        int total = roll + playerBonus + clothingBonus;
        bool success = total >= option.baseDifficulty + npcModifier;
        
        ApplyOutcome(success ? option.successOutcome : option.failureOutcome, npc);
    }
}
```

---

## 3. LEVEL SYSTEMS

### 3.1 Level Data Structure

```csharp
[CreateAssetMenu(fileName = "Level", menuName = "Sleepwalker/Level")]
public class LevelData : ScriptableObject
{
    public string levelName;
    public string sceneName;
    public Sprite thumbnail;
    
    [Header("Setup")]
    public SpawnPointData[] playerSpawnPoints;
    public NPCSpawnData[] npcSpawns;
    public ItemSpawnData[] itemSpawns;
    
    [Header("Objectives")]
    public ObjectiveData primaryObjective;
    public ObjectiveData[] secondaryObjectives;
    public ObjectiveData[] bonusObjectives;
    
    [Header("Settings")]
    public float parTime;
    public int maxShameAllowed = 100;
    public bool hasTimeLimit;
    public float timeLimit;
    
    [Header("Difficulty")]
    public int baseDifficulty;
    public DifficultyModifiers modifiers;
}

[System.Serializable]
public class SpawnPointData
{
    public Vector3 position;
    public Quaternion rotation;
    public float weight = 1f;  // Для рандомного выбора
    public string description;  // "В лифте", "На балконе"
}

[System.Serializable]
public class NPCSpawnData
{
    public NPCTypeData type;
    public Vector3 position;
    public PatrolRoute route;
    public NPCSchedule schedule;
}
```

### 3.2 Objective System

```csharp
public class ObjectiveManager : MonoBehaviour
{
    private List<Objective> activeObjectives = new();
    
    public event Action<Objective> OnObjectiveAdded;
    public event Action<Objective> OnObjectiveCompleted;
    public event Action<Objective> OnObjectiveFailed;
    public event Action OnAllObjectivesComplete;
    
    public void Initialize(LevelData levelData)
    {
        AddObjective(new Objective(levelData.primaryObjective));
        
        foreach (var secondary in levelData.secondaryObjectives)
        {
            AddObjective(new Objective(secondary));
        }
    }
    
    public void CheckObjectives()
    {
        foreach (var objective in activeObjectives)
        {
            if (objective.CheckCompletion())
            {
                CompleteObjective(objective);
            }
        }
    }
}

[System.Serializable]
public class ObjectiveData
{
    public string title;
    public string description;
    public ObjectiveType type;
    public bool isRequired;
    
    [Header("Conditions")]
    public ObjectiveCondition[] conditions;
    
    [Header("Rewards")]
    public int xpReward;
    public int shameReduction;
}

public enum ObjectiveType
{
    FindClothing,      // Найти любую одежду
    FindSpecificItem,  // Найти конкретный предмет
    ReachLocation,     // Добраться до точки
    AvoidDetection,    // Не быть замеченным
    SocialSuccess,     // Успешно пройти соц. проверку
    TimeLimit,         // Успеть за время
    NoPanic            // Не вызвать панику
}
```

---

## 4. SAVE SYSTEM

```csharp
[System.Serializable]
public class SaveData
{
    public string version;
    public DateTime saveTime;
    
    [Header("Progress")]
    public int currentLevel;
    public List<LevelProgress> levelProgress;
    
    [Header("Player")]
    public PlayerProgressData playerProgress;
    public List<string> unlockedSkills;
    public List<string> unlockedClothing;
    
    [Header("Stats")]
    public GameStats stats;
}

[System.Serializable]
public class PlayerProgressData
{
    public int totalXP;
    public int currentLevel;
    public int skillPoints;
    public Dictionary<string, int> skillLevels;
}

[System.Serializable]
public class LevelProgress
{
    public string levelId;
    public bool completed;
    public int bestStars;
    public float bestTime;
    public bool perfectRun;  // Никто не видел
}

public class SaveManager : MonoBehaviour
{
    private const string SAVE_KEY = "sleepwalker_save";
    
    public void Save()
    {
        var data = CollectSaveData();
        string json = JsonUtility.ToJson(data);
        PlayerPrefs.SetString(SAVE_KEY, json);
        PlayerPrefs.Save();
    }
    
    public SaveData Load()
    {
        if (!PlayerPrefs.HasKey(SAVE_KEY)) return null;
        
        string json = PlayerPrefs.GetString(SAVE_KEY);
        return JsonUtility.FromJson<SaveData>(json);
    }
}
```

---

## 5. UI SYSTEM

### 5.1 HUD Components

```csharp
public class HUDManager : MonoBehaviour
{
    [Header("References")]
    [SerializeField] private ShameBar shameBar;
    [SerializeField] private DetectionIndicator detectionIndicator;
    [SerializeField] private ObjectiveDisplay objectiveDisplay;
    [SerializeField] private InventoryHUD inventoryHUD;
    [SerializeField] private InteractionPrompt interactionPrompt;
    [SerializeField] private NPCIndicatorManager npcIndicators;
    
    public void UpdateShame(float current, float max);
    public void ShowDetection(NPCController npc, float amount);
    public void HideDetection(NPCController npc);
    public void UpdateObjective(Objective objective);
    public void ShowInteractionPrompt(string prompt);
    public void HideInteractionPrompt();
}

public class ShameBar : MonoBehaviour
{
    [SerializeField] private Image fillImage;
    [SerializeField] private Gradient colorGradient;
    [SerializeField] private Animator animator;
    
    public void SetValue(float normalized)
    {
        fillImage.fillAmount = normalized;
        fillImage.color = colorGradient.Evaluate(normalized);
        
        if (normalized > 0.75f)
        {
            animator.SetBool("Critical", true);
        }
    }
}
```

### 5.2 NPC Indicators

```csharp
public class NPCIndicatorManager : MonoBehaviour
{
    [SerializeField] private NPCIndicator indicatorPrefab;
    
    private Dictionary<NPCController, NPCIndicator> indicators = new();
    
    public void ShowIndicator(NPCController npc, NPCIndicatorType type)
    {
        if (!indicators.TryGetValue(npc, out var indicator))
        {
            indicator = Instantiate(indicatorPrefab, transform);
            indicators[npc] = indicator;
        }
        
        indicator.Show(npc, type);
    }
    
    public void UpdateDetection(NPCController npc, float progress)
    {
        if (indicators.TryGetValue(npc, out var indicator))
        {
            indicator.UpdateDetection(progress);
        }
    }
}

public enum NPCIndicatorType
{
    None,
    Question,      // ?
    Suspicious,    // ? (жёлтый)
    Alert,         // !
    Phone,         // 📱
    Help,          // 🤝
    Flee           // 😱
}
```

---

## 6. AUDIO SYSTEM

```csharp
public class AudioManager : MonoBehaviour
{
    public static AudioManager Instance { get; private set; }
    
    [Header("Mixers")]
    [SerializeField] private AudioMixer masterMixer;
    
    [Header("Music")]
    [SerializeField] private AudioSource musicSource;
    [SerializeField] private MusicData[] levelMusic;
    
    [Header("Ambient")]
    [SerializeField] private AudioSource ambientSource;
    
    // SFX через пул
    private AudioSourcePool sfxPool;
    
    public void PlaySFX(AudioClip clip, Vector3 position, float volume = 1f);
    public void PlayMusic(MusicTrack track, float fadeTime = 1f);
    public void SetTension(float level);  // Динамическая музыка
    
    public void PlayFootstep(SurfaceType surface, MovementType movement);
    public void PlayNPCReaction(NPCReactionType reaction, Vector3 position);
    public void PlayUISound(UISoundType type);
}
```

---

## 7. КОНФИГУРАЦИЯ И БАЛАНС

### 7.1 Game Config

```csharp
[CreateAssetMenu(fileName = "GameConfig", menuName = "Sleepwalker/Game Config")]
public class GameConfig : ScriptableObject
{
    [Header("Player")]
    public float baseWalkSpeed = 3f;
    public float baseRunSpeed = 6f;
    public float baseSneakSpeed = 1.5f;
    
    [Header("Detection")]
    public float baseDetectionDistance = 15f;
    public float baseDetectionAngle = 90f;
    public float detectionDecayRate = 0.5f;
    
    [Header("Shame")]
    public float maxShame = 100f;
    public float shameDecayRate = 1f;
    public float shameDecayDelay = 3f;
    
    [Header("Difficulty Scaling")]
    public AnimationCurve difficultyCurve;
    public float npcCountMultiplier = 1f;
    public float detectionSpeedMultiplier = 1f;
}
```

---

## 8. MVP SCOPE — ТЕХНИЧЕСКИЕ ТРЕБОВАНИЯ

### 8.1 Что должно работать в MVP

| Система | Статус | Приоритет |
|---------|--------|-----------|
| Player Movement | Обязательно | P0 |
| Shame System | Обязательно | P0 |
| Basic Clothing | Обязательно | P0 |
| NPC Perception | Обязательно | P0 |
| NPC State Machine (базовый) | Обязательно | P0 |
| Interaction System | Обязательно | P0 |
| HUD | Обязательно | P0 |
| 1 Level (Отель) | Обязательно | P0 |
| Save System | Важно | P1 |
| Social Interactions | Важно | P1 |
| Audio (базовый) | Важно | P1 |
| Progression | Можно позже | P2 |
| Multiple Levels | Можно позже | P2 |

### 8.2 Технические метрики

- **Target FPS:** 60 (PC), 30 (Switch)
- **Max NPC on screen:** 20
- **Level load time:** < 5 sec
- **Memory budget:** < 2GB RAM

### 8.3 Инструменты разработки

- **Version Control:** Git (GitHub/GitLab)
- **Project Management:** Notion / Trello
- **Communication:** Discord
- **CI/CD:** Unity Cloud Build или GitHub Actions
- **Analytics:** Unity Analytics (опционально)

---

## 9. CODING STANDARDS

### 9.1 Naming Conventions

```csharp
// Классы — PascalCase
public class PlayerController { }

// Методы — PascalCase
public void MovePlayer() { }

// Публичные поля — PascalCase
public float MoveSpeed;

// Приватные поля — camelCase с underscore
private float _moveSpeed;

// Параметры — camelCase
public void SetSpeed(float newSpeed) { }

// Константы — UPPER_SNAKE_CASE
private const float MAX_SPEED = 10f;

// События — On + Action
public event Action OnPlayerDied;
public event Action<float> OnHealthChanged;
```

### 9.2 File Organization

```
Assets/
├── _Project/
│   ├── Scripts/
│   │   ├── Core/
│   │   ├── Player/
│   │   ├── NPC/
│   │   ├── Systems/
│   │   ├── UI/
│   │   └── Utilities/
│   ├── Prefabs/
│   ├── ScriptableObjects/
│   ├── Scenes/
│   ├── Art/
│   ├── Audio/
│   └── Resources/
├── Plugins/
└── ThirdParty/
```