# Unity 상점 시스템 구현 가이드

**Author**: whals21
**Date**: 2025-11-18
**Target Engine**: Unity 2021.3 LTS 이상

---

## 📁 프로젝트 초기 설정

### 1. Unity 프로젝트 생성

1. Unity Hub에서 새 프로젝트 생성
2. 템플릿: **3D (URP)** 또는 **3D Core**
3. 프로젝트 이름: `StoreSystem`

### 2. 폴더 구조 생성

```
Assets/
├── Scripts/
│   ├── Core/
│   │   ├── GameManager.cs
│   │   ├── SaveSystem.cs
│   │   └── Singleton.cs
│   ├── Shop/
│   │   ├── ShopManager.cs
│   │   ├── ShopUI.cs
│   │   └── ShopItemUI.cs
│   ├── Inventory/
│   │   ├── InventoryManager.cs
│   │   ├── InventoryUI.cs
│   │   └── InventoryItemUI.cs
│   ├── Data/
│   │   ├── ItemData.cs (ScriptableObject)
│   │   └── PlayerData.cs
│   └── Player/
│       └── PlayerController.cs
├── Prefabs/
│   ├── UI/
│   │   ├── ShopItemSlot.prefab
│   │   └── InventoryItemSlot.prefab
│   └── Player/
│       └── Player.prefab
├── Resources/
│   └── Items/
│       ├── Potion.asset
│       ├── Sword.asset
│       └── Shield.asset
├── Sprites/
│   └── Icons/
│       ├── potion_icon.png
│       ├── sword_icon.png
│       └── shield_icon.png
└── Scenes/
    └── MainScene.unity
```

---

## 🔧 단계별 구현 가이드

## Phase 1: 데이터 구조 설정

### 1.1 ItemData.cs (ScriptableObject) 생성

**경로**: `Assets/Scripts/Data/ItemData.cs`

```csharp
using UnityEngine;

[CreateAssetMenu(fileName = "New Item", menuName = "Shop System/Item Data")]
public class ItemData : ScriptableObject
{
    [Header("기본 정보")]
    public string itemName;
    public string itemID;
    [TextArea(3, 5)]
    public string description;

    [Header("가격")]
    public int buyPrice;
    public int sellPrice; // buyPrice의 50%로 자동 계산됨

    [Header("비주얼")]
    public Sprite icon;

    [Header("추가 정보 (선택)")]
    public ItemRarity rarity = ItemRarity.Common;

    private void OnValidate()
    {
        // 판매 가격은 구매 가격의 50%
        sellPrice = Mathf.FloorToInt(buyPrice * 0.5f);
    }
}

public enum ItemRarity
{
    Common,
    Rare,
    Epic,
    Legendary
}
```

**작업 순서**:
1. 위 코드를 `ItemData.cs` 파일로 저장
2. Unity로 돌아가서 컴파일 완료 대기
3. `Assets/Resources/Items` 폴더에서 우클릭 → Create → Shop System → Item Data
4. 테스트용 아이템 3개 생성:
   - **Potion** (체력 포션): 가격 50, 아이콘 설정
   - **Sword** (검): 가격 200, 아이콘 설정
   - **Shield** (방패): 가격 150, 아이콘 설정

### 1.2 PlayerData.cs 생성

**경로**: `Assets/Scripts/Data/PlayerData.cs`

```csharp
using System;
using System.Collections.Generic;

[Serializable]
public class PlayerData
{
    public int gold = 1000; // 시작 골드
    public List<InventoryItem> inventory = new List<InventoryItem>();

    public PlayerData()
    {
        gold = 1000;
        inventory = new List<InventoryItem>();
    }
}

[Serializable]
public class InventoryItem
{
    public string itemID;
    public int quantity;

    public InventoryItem(string id, int qty)
    {
        itemID = id;
        quantity = qty;
    }
}
```

---

## Phase 2: 핵심 시스템 구현

### 2.1 Singleton.cs (유틸리티)

**경로**: `Assets/Scripts/Core/Singleton.cs`

```csharp
using UnityEngine;

public class Singleton<T> : MonoBehaviour where T : MonoBehaviour
{
    private static T instance;

    public static T Instance
    {
        get
        {
            if (instance == null)
            {
                instance = FindObjectOfType<T>();

                if (instance == null)
                {
                    GameObject go = new GameObject(typeof(T).Name);
                    instance = go.AddComponent<T>();
                }
            }
            return instance;
        }
    }

    protected virtual void Awake()
    {
        if (instance == null)
        {
            instance = this as T;
            DontDestroyOnLoad(gameObject);
        }
        else if (instance != this)
        {
            Destroy(gameObject);
        }
    }
}
```

### 2.2 SaveSystem.cs

**경로**: `Assets/Scripts/Core/SaveSystem.cs`

```csharp
using UnityEngine;
using System.IO;

public static class SaveSystem
{
    private static readonly string SAVE_FOLDER = Application.persistentDataPath + "/Saves/";
    private static readonly string SAVE_FILE = "playerData.json";

    public static void Init()
    {
        if (!Directory.Exists(SAVE_FOLDER))
        {
            Directory.CreateDirectory(SAVE_FOLDER);
        }
    }

    public static void SavePlayerData(PlayerData data)
    {
        Init();
        string json = JsonUtility.ToJson(data, true);
        File.WriteAllText(SAVE_FOLDER + SAVE_FILE, json);
        Debug.Log("데이터 저장 완료: " + SAVE_FOLDER + SAVE_FILE);
    }

    public static PlayerData LoadPlayerData()
    {
        Init();
        string filePath = SAVE_FOLDER + SAVE_FILE;

        if (File.Exists(filePath))
        {
            string json = File.ReadAllText(filePath);
            PlayerData data = JsonUtility.FromJson<PlayerData>(json);
            Debug.Log("데이터 로드 완료");
            return data;
        }
        else
        {
            Debug.Log("저장 파일이 없습니다. 새 데이터 생성");
            return new PlayerData();
        }
    }

    public static void DeleteSaveData()
    {
        string filePath = SAVE_FOLDER + SAVE_FILE;
        if (File.Exists(filePath))
        {
            File.Delete(filePath);
            Debug.Log("저장 데이터 삭제됨");
        }
    }
}
```

### 2.3 GameManager.cs

**경로**: `Assets/Scripts/Core/GameManager.cs`

```csharp
using UnityEngine;
using System.Collections.Generic;
using System.Linq;

public class GameManager : Singleton<GameManager>
{
    public PlayerData playerData;

    // 모든 아이템 데이터 로드 (Resources 폴더)
    private Dictionary<string, ItemData> allItems = new Dictionary<string, ItemData>();

    protected override void Awake()
    {
        base.Awake();
        LoadAllItems();
        LoadGame();
    }

    private void LoadAllItems()
    {
        ItemData[] items = Resources.LoadAll<ItemData>("Items");
        foreach (ItemData item in items)
        {
            if (!allItems.ContainsKey(item.itemID))
            {
                allItems.Add(item.itemID, item);
            }
        }
        Debug.Log($"아이템 로드 완료: {allItems.Count}개");
    }

    public ItemData GetItemData(string itemID)
    {
        if (allItems.TryGetValue(itemID, out ItemData item))
        {
            return item;
        }
        Debug.LogWarning($"아이템을 찾을 수 없습니다: {itemID}");
        return null;
    }

    // 골드 관련
    public int GetGold() => playerData.gold;

    public void AddGold(int amount)
    {
        playerData.gold += amount;
        SaveGame();
    }

    public bool SpendGold(int amount)
    {
        if (playerData.gold >= amount)
        {
            playerData.gold -= amount;
            SaveGame();
            return true;
        }
        return false;
    }

    // 인벤토리 관련
    public void AddItemToInventory(string itemID, int quantity = 1)
    {
        InventoryItem existingItem = playerData.inventory.Find(x => x.itemID == itemID);

        if (existingItem != null)
        {
            existingItem.quantity += quantity;
        }
        else
        {
            playerData.inventory.Add(new InventoryItem(itemID, quantity));
        }
        SaveGame();
    }

    public bool RemoveItemFromInventory(string itemID, int quantity = 1)
    {
        InventoryItem existingItem = playerData.inventory.Find(x => x.itemID == itemID);

        if (existingItem != null && existingItem.quantity >= quantity)
        {
            existingItem.quantity -= quantity;

            if (existingItem.quantity <= 0)
            {
                playerData.inventory.Remove(existingItem);
            }

            SaveGame();
            return true;
        }
        return false;
    }

    public int GetItemQuantity(string itemID)
    {
        InventoryItem item = playerData.inventory.Find(x => x.itemID == itemID);
        return item != null ? item.quantity : 0;
    }

    public List<InventoryItem> GetInventory()
    {
        return playerData.inventory;
    }

    // 저장/로드
    public void SaveGame()
    {
        SaveSystem.SavePlayerData(playerData);
    }

    public void LoadGame()
    {
        playerData = SaveSystem.LoadPlayerData();
    }
}
```

### 2.4 ShopManager.cs

**경로**: `Assets/Scripts/Shop/ShopManager.cs`

```csharp
using UnityEngine;
using System.Collections.Generic;

public class ShopManager : MonoBehaviour
{
    [Header("상점 상품 목록")]
    public List<ItemData> shopItems = new List<ItemData>();

    private void Start()
    {
        // Resources 폴더에서 자동으로 상품 로드 (선택)
        if (shopItems.Count == 0)
        {
            shopItems.AddRange(Resources.LoadAll<ItemData>("Items"));
        }
    }

    public bool BuyItem(ItemData item)
    {
        if (GameManager.Instance.SpendGold(item.buyPrice))
        {
            GameManager.Instance.AddItemToInventory(item.itemID, 1);
            Debug.Log($"{item.itemName} 구매 성공!");
            return true;
        }
        else
        {
            Debug.Log("골드가 부족합니다!");
            return false;
        }
    }

    public bool SellItem(string itemID)
    {
        ItemData itemData = GameManager.Instance.GetItemData(itemID);

        if (itemData != null && GameManager.Instance.GetItemQuantity(itemID) > 0)
        {
            GameManager.Instance.RemoveItemFromInventory(itemID, 1);
            GameManager.Instance.AddGold(itemData.sellPrice);
            Debug.Log($"{itemData.itemName} 판매 성공! +{itemData.sellPrice} 골드");
            return true;
        }

        Debug.Log("판매 실패!");
        return false;
    }

    public List<ItemData> GetShopItems()
    {
        return shopItems;
    }
}
```

---

## Phase 3: UI 구현

### 3.1 Unity UI 설정

**Scene 설정**:
1. **Canvas 생성**: Hierarchy → 우클릭 → UI → Canvas
   - Canvas Scaler → UI Scale Mode: **Scale With Screen Size**
   - Reference Resolution: **1920 x 1080**

2. **EventSystem** 자동 생성 확인

### 3.2 상점 UI 구조 생성

**Hierarchy 구조**:
```
Canvas
├── ShopPanel (전체 상점 창)
│   ├── Background (반투명 검은 배경)
│   ├── ShopWindow
│   │   ├── Header
│   │   │   ├── Title (Text: "상점")
│   │   │   ├── GoldText (Text: "골드: 1000")
│   │   │   └── CloseButton
│   │   ├── Content
│   │   │   ├── LeftPanel (상점 상품)
│   │   │   │   ├── Title (Text: "상점 상품")
│   │   │   │   └── ShopItemsContainer (Vertical Layout Group)
│   │   │   │       └── (ShopItemSlot 프리팹들이 여기 생성됨)
│   │   │   └── RightPanel (플레이어 인벤토리)
│   │   │       ├── Title (Text: "인벤토리")
│   │   │       └── InventoryItemsContainer (Vertical Layout Group)
│   │   │           └── (InventoryItemSlot 프리팹들이 여기 생성됨)
│   │   └── FeedbackPanel
│   │       └── FeedbackText (Text: "구매 성공!")
└── InventoryPanel (인벤토리 전용 창, 선택 과제)
```

**상세 설정**:

#### ShopPanel
- RectTransform: Stretch both (앵커 전체)
- Image: 색상 (0, 0, 0, 200) - 반투명 검은색
- 초기 상태: **비활성화**

#### ShopWindow
- RectTransform: 중앙 정렬, Width: 1400, Height: 800
- Image: 흰색 배경
- Layout: 수직 배치

#### LeftPanel / RightPanel
- Horizontal Layout Group으로 좌우 분할
- 각각 Width: 50%

#### ShopItemsContainer / InventoryItemsContainer
- Vertical Layout Group
- Child Force Expand: Width ✓, Height ✗
- Spacing: 10
- Padding: 10

### 3.3 ShopItemSlot Prefab 생성

**구조**:
```
ShopItemSlot
├── Icon (Image)
├── NameText (Text)
├── PriceText (Text)
└── BuyButton
    └── ButtonText (Text: "구매")
```

**설정**:
- Layout Element: Min Height: 80
- Horizontal Layout Group 추가

**스크립트**: `Assets/Scripts/Shop/ShopItemUI.cs`

```csharp
using UnityEngine;
using UnityEngine.UI;
using TMPro;

public class ShopItemUI : MonoBehaviour
{
    [Header("UI 참조")]
    public Image icon;
    public TextMeshProUGUI nameText;
    public TextMeshProUGUI priceText;
    public Button buyButton;

    private ItemData itemData;
    private ShopUI shopUI;

    public void Setup(ItemData item, ShopUI ui)
    {
        itemData = item;
        shopUI = ui;

        icon.sprite = item.icon;
        nameText.text = item.itemName;
        priceText.text = $"{item.buyPrice} G";

        buyButton.onClick.RemoveAllListeners();
        buyButton.onClick.AddListener(OnBuyButtonClicked);
    }

    private void OnBuyButtonClicked()
    {
        shopUI.OnBuyItem(itemData);
    }
}
```

### 3.4 InventoryItemSlot Prefab 생성

**구조**:
```
InventoryItemSlot
├── Icon (Image)
├── NameText (Text)
├── QuantityText (Text)
└── SellButton
    └── ButtonText (Text: "판매")
```

**스크립트**: `Assets/Scripts/Inventory/InventoryItemUI.cs`

```csharp
using UnityEngine;
using UnityEngine.UI;
using TMPro;

public class InventoryItemUI : MonoBehaviour
{
    [Header("UI 참조")]
    public Image icon;
    public TextMeshProUGUI nameText;
    public TextMeshProUGUI quantityText;
    public Button sellButton;

    private ItemData itemData;
    private ShopUI shopUI;

    public void Setup(ItemData item, int quantity, ShopUI ui)
    {
        itemData = item;
        shopUI = ui;

        icon.sprite = item.icon;
        nameText.text = item.itemName;
        quantityText.text = $"x{quantity}";

        sellButton.onClick.RemoveAllListeners();
        sellButton.onClick.AddListener(OnSellButtonClicked);
    }

    private void OnSellButtonClicked()
    {
        shopUI.OnSellItem(itemData);
    }
}
```

### 3.5 ShopUI.cs (메인 UI 컨트롤러)

**경로**: `Assets/Scripts/Shop/ShopUI.cs`

```csharp
using UnityEngine;
using UnityEngine.UI;
using TMPro;
using System.Collections;

public class ShopUI : MonoBehaviour
{
    [Header("UI 패널")]
    public GameObject shopPanel;

    [Header("UI 요소")]
    public TextMeshProUGUI goldText;
    public Transform shopItemsContainer;
    public Transform inventoryItemsContainer;
    public TextMeshProUGUI feedbackText;
    public GameObject feedbackPanel;

    [Header("프리팹")]
    public GameObject shopItemSlotPrefab;
    public GameObject inventoryItemSlotPrefab;

    [Header("참조")]
    public ShopManager shopManager;

    private bool isShopOpen = false;

    private void Start()
    {
        shopPanel.SetActive(false);
        feedbackPanel.SetActive(false);
    }

    private void Update()
    {
        // Tab 또는 E 키로 상점 토글
        if (Input.GetKeyDown(KeyCode.Tab) || Input.GetKeyDown(KeyCode.E))
        {
            ToggleShop();
        }

        // ESC 키로 상점 닫기
        if (Input.GetKeyDown(KeyCode.Escape) && isShopOpen)
        {
            CloseShop();
        }
    }

    public void ToggleShop()
    {
        if (isShopOpen)
        {
            CloseShop();
        }
        else
        {
            OpenShop();
        }
    }

    public void OpenShop()
    {
        isShopOpen = true;
        shopPanel.SetActive(true);

        // 커서 활성화
        Cursor.lockState = CursorLockMode.None;
        Cursor.visible = true;

        // 플레이어 조작 정지 (PlayerController 있다면)
        // PlayerController.Instance?.SetControlEnabled(false);

        RefreshUI();
    }

    public void CloseShop()
    {
        isShopOpen = false;
        shopPanel.SetActive(false);

        // 커서 비활성화
        Cursor.lockState = CursorLockMode.Locked;
        Cursor.visible = false;

        // 플레이어 조작 재개
        // PlayerController.Instance?.SetControlEnabled(true);
    }

    public void RefreshUI()
    {
        UpdateGoldDisplay();
        UpdateShopItems();
        UpdateInventory();
    }

    private void UpdateGoldDisplay()
    {
        goldText.text = $"골드: {GameManager.Instance.GetGold()} G";
    }

    private void UpdateShopItems()
    {
        // 기존 아이템 UI 삭제
        foreach (Transform child in shopItemsContainer)
        {
            Destroy(child.gameObject);
        }

        // 상점 아이템 생성
        foreach (ItemData item in shopManager.GetShopItems())
        {
            GameObject slotObj = Instantiate(shopItemSlotPrefab, shopItemsContainer);
            ShopItemUI itemUI = slotObj.GetComponent<ShopItemUI>();
            itemUI.Setup(item, this);
        }
    }

    private void UpdateInventory()
    {
        // 기존 인벤토리 UI 삭제
        foreach (Transform child in inventoryItemsContainer)
        {
            Destroy(child.gameObject);
        }

        // 인벤토리 아이템 생성
        foreach (InventoryItem invItem in GameManager.Instance.GetInventory())
        {
            ItemData itemData = GameManager.Instance.GetItemData(invItem.itemID);
            if (itemData != null)
            {
                GameObject slotObj = Instantiate(inventoryItemSlotPrefab, inventoryItemsContainer);
                InventoryItemUI itemUI = slotObj.GetComponent<InventoryItemUI>();
                itemUI.Setup(itemData, invItem.quantity, this);
            }
        }
    }

    // 구매 처리
    public void OnBuyItem(ItemData item)
    {
        bool success = shopManager.BuyItem(item);

        if (success)
        {
            ShowFeedback($"{item.itemName} 구매 성공!", Color.green);
            RefreshUI();
        }
        else
        {
            ShowFeedback("골드가 부족합니다!", Color.red);
        }
    }

    // 판매 처리
    public void OnSellItem(ItemData item)
    {
        bool success = shopManager.SellItem(item.itemID);

        if (success)
        {
            ShowFeedback($"{item.itemName} 판매 성공! +{item.sellPrice}G", Color.green);
            RefreshUI();
        }
        else
        {
            ShowFeedback("판매 실패!", Color.red);
        }
    }

    // 피드백 표시
    private void ShowFeedback(string message, Color color)
    {
        StopAllCoroutines();
        StartCoroutine(FeedbackCoroutine(message, color));
    }

    private IEnumerator FeedbackCoroutine(string message, Color color)
    {
        feedbackText.text = message;
        feedbackText.color = color;
        feedbackPanel.SetActive(true);

        yield return new WaitForSeconds(2f);

        feedbackPanel.SetActive(false);
    }
}
```

---

## Phase 4: Scene 설정 및 연결

### 4.1 Hierarchy 설정

1. **GameManager 오브젝트 생성**:
   - 빈 GameObject 생성 → 이름: `GameManager`
   - `GameManager.cs` 스크립트 추가

2. **ShopManager 오브젝트 생성**:
   - 빈 GameObject 생성 → 이름: `ShopManager`
   - `ShopManager.cs` 스크립트 추가
   - Inspector에서 `Shop Items` 리스트에 아이템 추가 (Resources/Items 폴더의 아이템들)

3. **ShopUI 연결**:
   - Canvas 선택
   - `ShopUI.cs` 스크립트 추가
   - Inspector에서 모든 참조 연결:
     - Shop Panel → ShopPanel 오브젝트
     - Gold Text → GoldText
     - Shop Items Container → ShopItemsContainer
     - Inventory Items Container → InventoryItemsContainer
     - Feedback Text, Feedback Panel 연결
     - Shop Item Slot Prefab, Inventory Item Slot Prefab 연결
     - Shop Manager → ShopManager 오브젝트

### 4.2 Prefab 저장

1. `ShopItemSlot` 오브젝트를 `Assets/Prefabs/UI/` 폴더로 드래그하여 Prefab 생성
2. `InventoryItemSlot` 오브젝트를 `Assets/Prefabs/UI/` 폴더로 드래그하여 Prefab 생성
3. Hierarchy에서 원본 삭제 (Prefab만 남김)

---

## 🎨 UI 스타일링 가이드

### 권장 색상 팔레트

```
배경: #2C2C2C (어두운 회색)
패널: #FFFFFF (흰색)
강조: #4CAF50 (초록) - 성공
경고: #F44336 (빨강) - 실패
텍스트: #212121 (검정)
버튼: #2196F3 (파랑)
```

### 폰트 설정

- **TextMeshPro** 사용 권장
- Window → TextMeshPro → Import TMP Essential Resources

---

## 🧪 테스트 체크리스트

### 기본 기능 테스트

- [ ] **상점 열기**: Tab/E 키로 상점이 열리는가?
- [ ] **상점 닫기**: Tab/E/ESC 키로 상점이 닫히는가?
- [ ] **커서**: 상점 열렸을 때 커서가 보이는가?
- [ ] **골드 표시**: 현재 골드가 올바르게 표시되는가?
- [ ] **아이템 구매 (성공)**: 골드가 충분할 때 구매되는가?
- [ ] **아이템 구매 (실패)**: 골드 부족 시 실패 메시지가 나오는가?
- [ ] **인벤토리 추가**: 구매한 아이템이 인벤토리에 표시되는가?
- [ ] **아이템 판매**: 인벤토리 아이템이 판매되는가?
- [ ] **골드 증가**: 판매 시 골드가 50% 환급되는가?
- [ ] **수량 감소**: 판매 시 인벤토리 수량이 감소하는가?
- [ ] **아이템 삭제**: 수량이 0이 되면 인벤토리에서 사라지는가?
- [ ] **피드백**: 성공/실패 메시지가 표시되는가?

### 저장/로드 테스트

- [ ] 아이템 구매 후 게임 재시작 → 인벤토리 유지되는가?
- [ ] 골드 사용 후 게임 재시작 → 골드 유지되는가?

---

## 🚀 빌드 가이드

### 1. 빌드 설정

1. File → Build Settings
2. **Scenes in Build**에 MainScene 추가
3. Platform 선택 (Windows, Mac, Linux)
4. Build 클릭

### 2. 빌드 최적화

**Player Settings**:
- Company Name: whals21
- Product Name: StoreSystem
- Default Icon 설정 (선택)
- Resolution:
  - Fullscreen Mode: Windowed
  - Default Screen Width: 1920
  - Default Screen Height: 1080

---

## 📹 시연 영상 촬영 가이드

### 촬영 순서

1. **게임 시작** (5초)
   - 타이틀/메인 화면 표시

2. **상점 열기** (5초)
   - Tab 또는 E 키 눌러서 상점 열기
   - UI 전체 화면 보여주기

3. **아이템 구매** (15초)
   - 상점 상품 목록에서 아이템 선택
   - [구매] 버튼 클릭
   - 골드 감소 확인
   - 인벤토리에 아이템 추가 확인
   - 피드백 메시지 확인

4. **골드 부족 시연** (10초)
   - 비싼 아이템 구매 시도
   - "골드가 부족합니다" 메시지 확인

5. **아이템 판매** (15초)
   - 인벤토리에서 아이템 선택
   - [판매] 버튼 클릭
   - 수량 감소 확인
   - 골드 증가 확인
   - 피드백 메시지 확인

6. **상점 닫기** (5초)
   - ESC 또는 Tab 키로 상점 닫기

### 촬영 도구

- **OBS Studio** (무료): https://obsproject.com/
- **Bandicam** (유료)
- Unity Recorder Package (내장)

### Unity Recorder 사용법

1. Window → Package Manager
2. Unity Registry에서 "Recorder" 검색 후 설치
3. Window → General → Recorder → Recorder Window
4. Add Recorder → Movie
5. 설정:
   - Output Resolution: 1920x1080
   - Frame Rate: 60 FPS
   - Quality: High
6. Start Recording

---

## 💎 선택 과제 구현 힌트

### ① ScriptableObject 데이터 관리
✅ 이미 구현됨 (`ItemData.cs`)

### ② 인벤토리 전용 창 (I 키)

**InventoryUI.cs** 추가 생성:
```csharp
private void Update()
{
    if (Input.GetKeyDown(KeyCode.I))
    {
        ToggleInventory();
    }
}
```

### ③ 희귀도 시스템

`ItemData.cs`에 이미 `ItemRarity` enum 포함됨.
가격 계산 시 희귀도에 따라 배율 적용:

```csharp
public int GetFinalPrice()
{
    float multiplier = 1.0f;
    switch (rarity)
    {
        case ItemRarity.Rare: multiplier = 1.5f; break;
        case ItemRarity.Epic: multiplier = 2.0f; break;
        case ItemRarity.Legendary: multiplier = 3.0f; break;
    }
    return Mathf.FloorToInt(buyPrice * multiplier);
}
```

### ④ 재고 시스템

`ShopManager.cs`에 재고 딕셔너리 추가:

```csharp
private Dictionary<string, int> shopStock = new Dictionary<string, int>();

public bool BuyItem(ItemData item)
{
    if (shopStock[item.itemID] <= 0)
    {
        Debug.Log("재고가 없습니다!");
        return false;
    }

    // ... 기존 구매 로직

    shopStock[item.itemID]--;
    return true;
}
```

### ⑤ 랜덤 진열

```csharp
private void Start()
{
    ItemData[] allItems = Resources.LoadAll<ItemData>("Items");
    shopItems = allItems.OrderBy(x => Random.value).Take(5).ToList();
}
```

### ⑥ MVVM 패턴

ViewModel 레이어 추가:
- `ShopViewModel.cs`: 상점 데이터 바인딩
- `InventoryViewModel.cs`: 인벤토리 데이터 바인딩
- Observable 패턴으로 UI 자동 갱신

### ⑦ 3D 게임 연동

별도의 3D 수집 게임에서 `GameManager.Instance.AddGold(coins)` 호출

### ⑧ 저장/불러오기

✅ 이미 구현됨 (`SaveSystem.cs`, `GameManager.cs`)

---

## 🐛 일반적인 문제 해결

### 1. "NullReferenceException" 발생
- Inspector에서 모든 참조가 올바르게 연결되었는지 확인
- ShopUI의 모든 public 필드 체크

### 2. 아이템이 표시되지 않음
- Resources/Items 폴더 경로 확인
- ItemData의 itemID가 고유한지 확인
- Icon 스프라이트가 할당되었는지 확인

### 3. 상점이 열리지 않음
- ShopPanel이 비활성화 상태인지 확인
- EventSystem이 Scene에 존재하는지 확인

### 4. 저장이 안됨
- Console에서 저장 경로 확인 (Application.persistentDataPath)
- 파일 권한 문제 확인

### 5. UI가 클릭되지 않음
- Canvas에 GraphicRaycaster가 있는지 확인
- Button에 Image 컴포넌트가 있는지 확인 (투명해도 OK)

---

## 📚 추가 학습 자료

- Unity UI 공식 문서: https://docs.unity3d.com/Manual/UISystem.html
- TextMeshPro: https://docs.unity3d.com/Manual/com.unity.textmeshpro.html
- ScriptableObject: https://docs.unity3d.com/Manual/class-ScriptableObject.html
- JSON 직렬화: https://docs.unity3d.com/Manual/JSONSerialization.html

---

## ✅ 최종 제출 전 체크리스트

- [ ] 모든 필수 기능 구현 완료
- [ ] 빌드 파일 생성 완료
- [ ] 시연 영상 촬영 완료 (30~60초)
- [ ] Git 저장소에 코드 푸시 완료
- [ ] README.md에 설치 및 실행 방법 작성
- [ ] 선택 과제 구현 여부 문서화

---

**구현 중 질문이나 문제가 있다면 Git Issue를 통해 문의하세요!**

**Good Luck! 🚀**
