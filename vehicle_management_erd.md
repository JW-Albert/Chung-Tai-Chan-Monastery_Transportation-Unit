# 車輛進出管理系統 — 資料庫設計文件

## 目錄

1. [角色與權限](#角色與權限)
2. [資料表定義](#資料表定義)
3. [業務流程](#業務流程)
4. [資料關聯](#資料關聯)

---

## 角色與權限

| 角色 | 代碼 | 權限說明 |
|------|------|---------|
| 指揮中心 | `command_center` | 最高權限，可覆蓋所有狀態、審核解鎖申請、手動調整檢查點狀態 |
| 管理員 | `admin` | 管理精舍清單，可手動調整檢查點狀態 |
| 精舍總領隊 | `temple_leader` | 管理本精舍車輛資料，僅能查看自己精舍資料 |
| 各車領隊 | `driver_leader` | 操作單一車輛狀態，出發前可修改車輛基本資料（含車牌） |

---

## 資料表定義

### USER — 使用者

| 欄位 | 型別 | Key | 說明 |
|------|------|-----|------|
| id | uuid | PK | 唯一識別碼 |
| name | string | | 姓名 |
| email | string | | 登入帳號（唯一） |
| password_hash | string | | 雜湊後密碼 |
| role | enum | | `command_center` \| `admin` \| `temple_leader` \| `driver_leader` |
| created_at | timestamp | | 建立時間 |

---

### ACTIVITY — 活動

每次法會即為一筆活動紀錄，所有精舍與車輛皆掛載於同一活動下。

| 欄位 | 型別 | Key | 說明 |
|------|------|-----|------|
| id | uuid | PK | 唯一識別碼 |
| name | string | | 活動名稱（例：2026 浴佛法會） |
| description | text | | 活動描述 |
| created_by | uuid | FK → USER | 建立者 |
| created_at | timestamp | | 建立時間 |

---

### CHECKPOINT — 檢查點

屬於活動層級，每次活動可自訂不同的檢查點配置。車輛可跳過某些檢查點（例如直接進 B 點），`order_index` 僅供顯示排序，不代表強制流程。

| 欄位 | 型別 | Key | 說明 |
|------|------|-----|------|
| id | uuid | PK | 唯一識別碼 |
| activity_id | uuid | FK → ACTIVITY | 所屬活動 |
| name | string | | 檢查點名稱（例：下車點A、停車場B） |
| location_desc | string | | 地點描述 |
| order_index | int | | 顯示排序 |
| created_at | timestamp | | 建立時間 |

---

### TEMPLE — 精舍

| 欄位 | 型別 | Key | 說明 |
|------|------|-----|------|
| id | uuid | PK | 唯一識別碼 |
| activity_id | uuid | FK → ACTIVITY | 所屬活動 |
| name | string | | 精舍名稱 |
| region | string | | 所在地區（例：台中、高雄） |
| created_at | timestamp | | 建立時間 |

---

### VEHICLE — 車輛

出發後車牌自動鎖定（`plate_locked = true`）。若需解鎖，須透過 `PLATE_UNLOCK_REQUEST` 流程由指揮中心審核後方可修改。

| 欄位 | 型別 | Key | 說明 |
|------|------|-----|------|
| id | uuid | PK | 唯一識別碼 |
| temple_id | uuid | FK → TEMPLE | 所屬精舍 |
| leader_user_id | uuid | FK → USER | 該車領隊 |
| plate_number | string | | 車牌號碼 |
| trip_number | int | | 車次編號 |
| seat_count | int | | 座位數 |
| depart_time | timestamp | | 預計出發時間 |
| return_time | timestamp | | 預計回程時間 |
| plate_locked | boolean | | 出發後鎖定，預設 `false` |
| created_at | timestamp | | 建立時間 |

---

### VEHICLE_LOG — 車輛動態紀錄

每筆狀態變更皆留存紀錄，不覆蓋舊資料，以最新一筆作為當前狀態。`source` 欄位標記來源以利事後審計。

| 欄位 | 型別 | Key | 說明 |
|------|------|-----|------|
| id | uuid | PK | 唯一識別碼 |
| vehicle_id | uuid | FK → VEHICLE | 所屬車輛 |
| checkpoint_id | uuid | FK → CHECKPOINT | 關聯檢查點（可為 null） |
| source | enum | | `leader` \| `camera` \| `command_center` |
| event_type | enum | | `departed` \| `highway_exit` \| `checkpoint_enter` \| `checkpoint_exit` \| `other` |
| custom_note | text | | 自由文字備註（event_type = `other` 時使用） |
| override_by | uuid | FK → USER | 覆蓋操作者（可為 null） |
| logged_at | timestamp | | 事件發生時間 |

---

### CAMERA — 鏡頭

每個檢查點可設置多支鏡頭，並區分進場與出場方向。

| 欄位 | 型別 | Key | 說明 |
|------|------|-----|------|
| id | uuid | PK | 唯一識別碼 |
| checkpoint_id | uuid | FK → CHECKPOINT | 所在檢查點 |
| name | string | | 鏡頭名稱 |
| stream_url | string | | 串流位址 |
| direction | enum | | `enter` \| `exit` |
| created_at | timestamp | | 建立時間 |

---

### CAMERA_EVENT — 鏡頭辨識事件

鏡頭辨識結果不直接觸發 `VEHICLE_LOG`，須先存入此表，由系統嘗試比對。信心度不足時需人工審核，確認後才產生對應的動態紀錄。

| 欄位 | 型別 | Key | 說明 |
|------|------|-----|------|
| id | uuid | PK | 唯一識別碼 |
| camera_id | uuid | FK → CAMERA | 觸發鏡頭 |
| vehicle_id | uuid | FK → VEHICLE | 比對到的車輛（可為 null） |
| raw_plate | string | | 鏡頭辨識到的原始車牌字串 |
| confidence | float | | 辨識信心度 0.0 ~ 1.0 |
| matched | boolean | | 是否成功比對到系統內車輛 |
| reviewed | boolean | | 是否經過人工審核 |
| reviewed_by | uuid | FK → USER | 審核者（可為 null） |
| captured_at | timestamp | | 拍攝時間 |

---

### PLATE_UNLOCK_REQUEST — 車牌解鎖申請

車牌解鎖為獨立審核流程，完整記錄申請與審核歷程。

| 欄位 | 型別 | Key | 說明 |
|------|------|-----|------|
| id | uuid | PK | 唯一識別碼 |
| vehicle_id | uuid | FK → VEHICLE | 申請解鎖的車輛 |
| requested_by | uuid | FK → USER | 申請者 |
| reason | text | | 申請原因（例：車輛故障換車） |
| status | enum | | `pending` \| `approved` \| `rejected` |
| reviewed_by | uuid | FK → USER | 審核者（可為 null） |
| created_at | timestamp | | 申請時間 |
| reviewed_at | timestamp | | 審核時間（可為 null） |

---

## 業務流程

### 1. 活動建立流程

```
指揮中心 / 管理員
  │
  ├─ 建立 ACTIVITY（活動名稱、描述）
  │
  ├─ 新增 CHECKPOINT（下車點、停車場等，可多個）
  │     └─ 設定 order_index（僅供顯示，不強制流程）
  │
  └─ 新增 TEMPLE（各精舍）
        └─ 精舍總領隊開始建立車輛資料
```

---

### 2. 車輛出發前設定流程

```
精舍總領隊 / 各車領隊
  │
  ├─ 建立 VEHICLE
  │     ├─ 填入：車牌、車次、座位數、領隊、出發/回程時間
  │     └─ plate_locked = false（可修改）
  │
  └─ 出發前可隨時修改車輛基本資料（含車牌）
```

---

### 3. 去程動態回報流程

```
各車領隊（手動回報）
  │
  ├─ 按下「出發」
  │     └─ 寫入 VEHICLE_LOG：event_type = departed, source = leader
  │     └─ 系統自動將 plate_locked 設為 true
  │
  ├─ 按下「下交流道」
  │     └─ 寫入 VEHICLE_LOG：event_type = highway_exit, source = leader
  │
  └─ 其他狀態（自由文字）
        └─ 寫入 VEHICLE_LOG：event_type = other, custom_note = "...", source = leader

鏡頭（自動觸發）
  │
  ├─ 車輛經過檢查點入口
  │     └─ 寫入 CAMERA_EVENT：raw_plate, confidence, direction = enter
  │     └─ 系統嘗試比對 VEHICLE.plate_number
  │           ├─ 比對成功（confidence 高）→ matched = true
  │           │     └─ 自動寫入 VEHICLE_LOG：event_type = checkpoint_enter, source = camera
  │           └─ 比對失敗 / 信心度低 → matched = false
  │                 └─ 等待指揮中心 / 管理員人工審核
  │                       └─ 審核確認後寫入 VEHICLE_LOG
  │
  └─ 車輛經過檢查點出口
        └─ 同上流程，event_type = checkpoint_exit, direction = exit
```

---

### 4. 車牌解鎖流程（中途故障換車）

```
各車領隊 / 精舍總領隊
  │
  └─ 建立 PLATE_UNLOCK_REQUEST
        ├─ 填入：申請原因（例：車輛故障）
        └─ status = pending

指揮中心 / 管理員（收到申請通知）
  │
  ├─ 審核通過 → status = approved
  │     └─ 系統將 VEHICLE.plate_locked 設為 false
  │           └─ 領隊修改車牌後，系統自動重新鎖定（plate_locked = true）
  │
  └─ 審核拒絕 → status = rejected
```

---

### 5. 回程流程

```
各車領隊
  │
  └─ 按下「離開」
        └─ 寫入 VEHICLE_LOG：event_type = checkpoint_exit, source = leader
              └─ 記錄離場時間，不強制記錄其他回程狀態

鏡頭（自動觸發）
  │
  └─ 偵測車輛離開出口
        └─ 寫入 CAMERA_EVENT：direction = exit
              └─ 比對成功後寫入 VEHICLE_LOG
```

---

### 6. 指揮中心覆蓋流程

```
指揮中心 / 管理員
  │
  └─ 手動調整任意車輛狀態
        └─ 寫入 VEHICLE_LOG
              ├─ source = command_center
              ├─ override_by = USER.id（操作者）
              └─ 原有紀錄保留，以最新一筆為當前狀態
```

---

## 資料關聯

```
ACTIVITY ──< CHECKPOINT ──< CAMERA ──< CAMERA_EVENT
    │
    └──< TEMPLE ──< VEHICLE ──< VEHICLE_LOG
                       │
                       └──< PLATE_UNLOCK_REQUEST
```

| 關聯 | 說明 |
|------|------|
| ACTIVITY → CHECKPOINT | 一個活動有多個檢查點 |
| ACTIVITY → TEMPLE | 一個活動包含多個精舍 |
| TEMPLE → VEHICLE | 一個精舍擁有多台車輛 |
| CHECKPOINT → CAMERA | 一個檢查點可設多支鏡頭 |
| CAMERA → CAMERA_EVENT | 鏡頭產生辨識事件 |
| VEHICLE → VEHICLE_LOG | 車輛所有狀態變更紀錄 |
| VEHICLE → PLATE_UNLOCK_REQUEST | 車牌解鎖申請紀錄 |
