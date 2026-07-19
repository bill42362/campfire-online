# UE5 Cookbook — 從零到「可互動營火」

> 對象：**完全沒碰過 UE5** 的人。
> 目標：① 親眼看到真・Tier 3 流體火焰；② 從零建一個「會被程式即時控制(送禮 → 加柴雛形)」的營火。
> 撰寫日期：2026-07-19。相關背景見 [estimate.md](./estimate.md)。
>
> ⚠️ **版本提醒**：本文以 **UE 5.4 / 5.5** 為準。UE 各版選單名稱偶有差異，找不到時用文中給的「搜尋關鍵字」找即可。Niagara Fluids 目前仍是 **Experimental(實驗性)**。

---

## 0. 先看這張圖：整件事的全貌

```
【建構期 · 一次性 · 在編輯器裡做】       【執行期 · 直播時 · 程式驅動】
  用 Niagara 建好火焰流體本體      ──►   火焰照規則跑，但暴露「可即時改的參數」
  (拉節點，Claude Code 幫不上這塊)        送禮事件 → 改參數 → 火焰當場變大/加柴
                                          (這塊 Claude Code 是主力)
```

**關鍵觀念**：編輯器是「建火焰」用的，不是「火焰運作的方式」。上線後的互動是**執行期改參數**，跟「當初怎麼建的」無關。所以「GUI 拉節點」只影響**開發期**、不影響**上線互動性**。

---

## 1. 硬體與平台：哪台機器做哪件事

| 用途 | Apple M2 Max (32GB) | 2080Ti / Windows |
| --- | --- | --- |
| 學 UE5、介面、Blueprint、寫互動邏輯 | ✅ 適合 | ✅ 適合 |
| **Niagara Fluids 流體火焰擬真度/效能評估** | ⚠️ 有風險(Metal 對 GPU 流體支援有限、且非目標硬體) | ✅ **建議在這裡做** |
| 正式播出(串流 + 模擬) | ❌ 非目標硬體 | ✅ 目標硬體 |

**做法建議**：M2 Max 上裝 UE5 熟悉介面、學 Blueprint、寫「事件 → 參數」邏輯；**「火夠不夠真、跑不跑得動」的決定性評估放到 2080Ti**。本文步驟兩台都適用；遇到 Mac 專屬注意事項會標「🍎 Mac」。

---

## 2. 名詞速成（五分鐘看懂 UE5 在講什麼）

| 名詞 | 白話解釋 |
| --- | --- |
| **Project(專案)** | 一個資料夾，裝你這個作品的所有東西 |
| **Level / Map(關卡 / 場景)** | 一個 3D 場景。營火就擺在這裡 |
| **Actor** | 任何被放進場景的東西(火焰、木頭、燈、相機) |
| **Component(元件)** | 掛在 Actor 上的功能零件 |
| **Niagara System** | UE 的粒子/特效/**流體**系統。我們的火焰就是一個 Niagara System |
| **Blueprint(藍圖)** | UE 的**視覺化程式**——用連線節點寫邏輯，不用打字寫 C++ |
| **User Parameter(`User.*`)** | Niagara 對外暴露、**執行期可即時改**的參數。互動的關鍵 |
| **Play / PIE** | 按下播放，在編輯器裡即時預覽 |

---

## 3. 安裝 UE5

1. 到 **epicgames.com** 註冊免費 Epic 帳號。
2. 下載安裝 **Epic Games Launcher**(啟動器)。
3. 打開 Launcher → 上方 **Unreal Engine** → 左側 **Library(程式庫)** → 按 **+** 安裝引擎，選 **5.4** 或 **5.5** 正式版。
   - 空間：引擎本體約 **30–50 GB**，請留足硬碟。
   - 時間：下載 + 安裝約 30–60 分鐘(看網速)。
4. 🍎 **Mac**：UE5 原生支援 Apple Silicon，直接裝即可。
   - 若之後要用 **C++**，需要先裝 **Xcode**(App Store 免費)。**本文全程走 Blueprint，先不用碰 C++**。

---

## 4. 第一次開 UE5：建專案 + 介面導覽

### 4.1 建立專案
1. Launcher 裡按引擎版本的 **Launch(啟動)**。
2. 出現專案瀏覽器 → 選 **Games → Blank(空白)**。
3. 右側設定：
   - **Blueprint**(不是 C++)。
   - **Starter Content**：勾著(有內建方塊、材質可用)。
   - 專案名稱如 `CampfirePoC`、選好存放位置 → **Create**。

### 4.2 介面四大塊(記住這四個就夠開始)

```
┌───────────────────────────────────────────────┐
│  Toolbar(上方工具列：Play 播放鍵在這)          │
├──────────────┬─────────────────┬───────────────┤
│              │                 │               │
│  (無固定)    │   Viewport      │   Details      │
│              │   3D 視窗        │   選中物件的    │
│              │   場景在這       │   所有屬性      │
│              │                 │               │
├──────────────┴─────────────────┴───────────────┤
│  Content Browser(內容瀏覽器：你所有的資產)      │
└───────────────────────────────────────────────┘
        Outliner(大綱)：場景裡所有 Actor 清單，通常在右上
```

### 4.3 視角操作(最容易卡住的地方，先練熟)
- **按住滑鼠右鍵 + 拖曳**：轉動視角。
- **右鍵按著同時按 W / A / S / D**：在場景中前後左右飛。滾輪調飛行速度。
- **滑鼠中鍵拖曳**：平移。
- **選一個物件後按 `F`**：鏡頭聚焦到它。
- 🍎 **Mac**：沒有實體右鍵/中鍵時，用觸控板的對應手勢或外接滑鼠會順很多(強烈建議接滑鼠)。

---

## 5. 最快看到真火（15 分鐘，零建置）

**先別自己建**——直接看 Epic 官方做好的流體火焰，判斷「這個擬真度是不是你要的」。

1. Epic Games Launcher → **Unreal Engine → Learn(學習)** 分頁(或到 **Fab** 搜尋)。
2. 搜尋 **「Niagara Fluids」**,找 Epic 官方的免費範例專案 / Content Examples。
3. 下載 → 用 UE 5.4/5.5 開啟。
4. 在 **Content Browser** 找火焰 / combustion / fire 相關的關卡,雙擊開啟 → 在 Viewport 直接看,或按 **Play**。

> 🎯 **這步的目的**:在**不寫任何東西**的情況下,先確認 Tier 3 流體火焰的擬真度值不值得你投資。**建議在 2080Ti 上看**(見第 1 節)。

---

## 6. 從零搭一個營火（30–60 分鐘）

### 6.1 啟用 Niagara Fluids 外掛
1. 上方選單 **Edit(編輯) → Plugins(外掛)**。
2. 搜尋框打 **`Niagara Fluids`** → 勾選啟用。
3. 依提示 **重啟編輯器**。(沒重啟就看不到下一步的模板)

### 6.2 建立火焰流體
1. 在 **Content Browser** 空白處 **按右鍵 → FX → Niagara System**。
2. 精靈選 **「New system from a template or behavior example」**(從模板建立)。
3. 在 **Fluids** 分類裡選一個 **3D Gas / Combustion(燃燒)** 類的模板(會直接給你火 + 煙)。
   - 找不到就在精靈搜尋框打 `gas` 或 `combustion`。
4. 命名為 `NS_Campfire`,完成。
5. 把 `NS_Campfire` 從 Content Browser **拖進 Viewport** → 場景裡就出現一團火。用 `F` 聚焦看它。

### 6.3 放一根「木頭」讓火與它互動
1. 上方 **內容/Quickly Add(加號圖示) → Shapes → Cylinder**(或 Cube)拖進場景,壓扁旋轉當木頭。
2. 選它 → **Details** 面板確認有 **Collision(碰撞)**。
3. 回到 `NS_Campfire`:流體要「感知障礙物」需在 Niagara 系統裡設定 **Collision / Obstacle**(不同模板位置不同,搜尋 `collision`)。設好後,火流體就會**繞著木頭走** —— 這就是純 shader 給不了的「火與木互動」。

> 這步是 Fluids 較進階的部分,第一次做不順很正常;先讓火出來、木頭之後再調。

### 6.4 調解析度(依機器調順)
- 選 `NS_Campfire`,在 Niagara 編輯器找 **Grid Resolution / Resolution Max Axis** 之類參數。
- 卡頓就**調低解析度**;2080Ti 可往上加。這是 fps 與細緻度的取捨。

---

## 7. 證明「火能被程式即時控制」（互動雛形）

這一節最重要——它**證明上線後聊天室互動可行**。我們先用「按鍵盤」代替「觀眾送禮」,把資料流打通。

### 7.1 在 Niagara 開一個對外參數
1. 打開 `NS_Campfire`(雙擊)。
2. 在 **Parameters(參數)** 面板找 **User Parameters** → 按 **+** 新增一個 **Float**,命名 `FireIntensity`(完整名會是 `User.FireIntensity`)。
3. 把這個參數接到會影響火大小的地方(例如 **Spawn Rate / 溫度 / 燃料量**)——在對應模組把原本的固定值改成引用 `User.FireIntensity`。
4. 存檔。

### 7.2 用 Blueprint「按鍵改參數」
1. 上方 **Blueprints → Open Level Blueprint(開啟關卡藍圖)**。
2. 在藍圖空白處按右鍵,加一個 **鍵盤事件**(例如搜 `F key` → Keyboard Events → F)。
3. 從場景選中 `NS_Campfire`,回藍圖右鍵 → **Create a Reference to NS_Campfire**。
4. 加節點 **Set Niagara Variable (Float)**:
   - Target 接 `NS_Campfire` 參考。
   - In Variable Name 填 `User.FireIntensity`。
   - In Value 填一個較大的值(或接一個每次 +1 的變數)。
5. 把「按 F 鍵」事件連到這個 Set 節點。
6. **Compile(編譯) → Play** → 按 **F**,火應該**當場變大**。

> ✅ **這一刻等於證明了**:「事件(現在是按鍵)→ 改 Niagara 參數 → 火即時反應」。上線時,只要把「按鍵」換成「後端送來的**送禮事件**」,就是**送禮 → 加柴**。天氣複寫、風向同理(多開幾個 `User.*` 參數即可)。

---

## 8. 效能與平台注意事項

- **解析度是效能主旋鈕**:3D 流體很吃 GPU,先從低解析度確認畫面,再往上加到 2080Ti 跑得動的甜蜜點。
- **Substeps / 迭代次數**:越高越穩定但越慢,調到「看起來穩定」即可。
- **7×24 穩定性**:流體長時間跑要注意記憶體與數值穩定(見 [estimate.md](./estimate.md) 的穩定性策略);正式播出務必配 watchdog + 定時重啟 + 狀態可回復。
- 🍎 **Mac/Metal**:Niagara Fluids 的 GPU 模擬在 Metal 上支援較有限,**不要用 Mac 的結果當作最終擬真度/效能判斷**——以 2080Ti 為準。

---

## 9. 常見坑（Troubleshooting）

| 症狀 | 多半原因 |
| --- | --- |
| 建 Niagara System 時找不到 Fluids 模板 | 沒啟用 Niagara Fluids 外掛,或啟用後沒重啟編輯器 |
| 火拖進場景看不到 | 太小 / 相機沒對到(按 `F` 聚焦) / 在原點被別的物件擋住 |
| 按 Play 沒反應或藍圖沒作用 | 忘了 **Compile**;或 Set Niagara Variable 的變數名稱沒寫成 `User.xxx` |
| Mac 上編譯 C++ 報錯 | 沒裝 Xcode;本文走 Blueprint 可先略過 C++ |
| 流體超級卡 | 解析度太高 / substeps 太多 → 往下調 |
| 火不繞木頭 | 沒設 Collision/Obstacle,或木頭沒碰撞 |

---

## 10. 這一切怎麼接回整個專案（下一步）

對照 [estimate.md](./estimate.md) 的里程碑,火焰是「**送禮/聊天事件的又一個消費者**」:

```
觀眾送禮 (YouTube / SWAG)
  → 後端服務 (M1 的 Node,抽象事件介面)
  → gift 事件 經 WebSocket / 本機 IPC
  → UE5 執行期收到 (UE 內用 WebSocket 外掛或 C++)
  → Set Niagara Variable("User.FireIntensity", 新值)
  → 流體火焰當場加柴
```

**建議節奏**:
1. 先照第 7 節,在 UE 內用「**按鍵 = 假送禮**」把「事件 → 火反應」跑通。
2. 再把「按鍵」換成「後端經 WebSocket 送來的真送禮事件」。
3. 天氣複寫、風向、濕度都用同樣手法多開幾個 `User.*` 參數。

> 誰做哪塊(呼應 estimate 的判斷):
> - **Niagara 火焰本體(拉節點)** → 你/人手動,一次性,Claude Code 幫不上。
> - **事件橋接、參數驅動邏輯、後台、聊天聚合、串流整合** → **Claude Code 主力**。

---

## 附錄：最短路徑摘要

1. 裝 Epic Launcher → 裝 UE 5.4/5.5。
2. **先**下載官方 Niagara Fluids 範例,在 **2080Ti** 上看真火 → 決定擬真度值不值得。
3. 值得 → 新 Blueprint 專案 → 啟用 Niagara Fluids 外掛 → 建 3D Gas/Combustion 系統 → 拖進場景。
4. 加 `User.FireIntensity` 參數 + Level Blueprint 按鍵改它 → 證明「事件 → 火反應」。
5. 把按鍵換成後端送禮事件 → 送禮加柴上線。
