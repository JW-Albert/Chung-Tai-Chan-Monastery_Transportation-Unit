# 車輛進出管理系統

中台禪寺交通組車輛進出管理平台，用於法會期間統一調度各精舍車輛、即時掌握進出場動態，並支援攝影機自動辨識與指揮中心覆蓋管理。

---

## 功能概覽

- **活動管理**：每次法會建立獨立活動，配置專屬檢查點與精舍清單
- **車輛追蹤**：各車領隊手動回報出發、下交流道等狀態，系統留存完整動態紀錄
- **攝影機辨識**：檢查點鏡頭自動辨識車牌，信心度足夠時直接寫入動態紀錄；不足時送審
- **車牌鎖定**：車輛出發後自動鎖定車牌，中途換車須經指揮中心審核解鎖
- **覆蓋紀錄**：指揮中心可手動調整任意狀態，所有操作完整保留稽核軌跡

---

## 角色與權限

| 角色 | 代碼 | 權限 |
|------|------|------|
| 指揮中心 | `command_center` | 最高權限：覆蓋所有狀態、審核解鎖申請、手動調整檢查點狀態 |
| 管理員 | `admin` | 管理精舍清單、手動調整檢查點狀態 |
| 精舍總領隊 | `temple_leader` | 管理本精舍車輛資料，僅能查看自己精舍資料 |
| 各車領隊 | `driver_leader` | 操作單一車輛狀態，出發前可修改車輛基本資料（含車牌） |

---

## 資料庫結構

### 資料表一覽

| 資料表 | 說明 |
|--------|------|
| `USER` | 系統使用者，包含角色與登入資訊 |
| `ACTIVITY` | 活動（法會），為所有資料的頂層容器 |
| `CHECKPOINT` | 活動下的檢查點，如下車點、停車場 |
| `TEMPLE` | 參與活動的精舍 |
| `VEHICLE` | 精舍旗下車輛，含車牌鎖定狀態 |
| `VEHICLE_LOG` | 車輛所有狀態變更紀錄（append-only） |
| `CAMERA` | 檢查點攝影機，區分進場 / 出場方向 |
| `CAMERA_EVENT` | 攝影機辨識結果，含信心度與審核狀態 |
| `PLATE_UNLOCK_REQUEST` | 車牌解鎖申請與審核流程 |

### 資料關聯

```
ACTIVITY ──< CHECKPOINT ──< CAMERA ──< CAMERA_EVENT
    │
    └──< TEMPLE ──< VEHICLE ──< VEHICLE_LOG
                       │
                       └──< PLATE_UNLOCK_REQUEST
```

---

## 主要業務流程

### 1. 活動建立

1. 指揮中心 / 管理員建立活動（`ACTIVITY`）
2. 新增檢查點（`CHECKPOINT`），設定 `order_index` 供顯示排序，不強制流程順序
3. 新增各精舍（`TEMPLE`），精舍總領隊開始登錄車輛資料

### 2. 車輛出發前設定

- 精舍總領隊 / 各車領隊建立 `VEHICLE`，填入車牌、車次、座位數、出發與回程時間
- 出發前 `plate_locked = false`，可隨時修改資料（含車牌）

### 3. 去程動態回報

**手動回報（領隊操作）**

| 動作 | 寫入 VEHICLE_LOG |
|------|----------------|
| 按下「出發」 | `event_type = departed`，系統自動將 `plate_locked` 設為 `true` |
| 按下「下交流道」 | `event_type = highway_exit` |
| 自訂狀態 | `event_type = other`，填入 `custom_note` |

**自動觸發（攝影機）**

1. 車輛經過檢查點 → 寫入 `CAMERA_EVENT`（含 `raw_plate`、`confidence`）
2. 系統嘗試比對 `VEHICLE.plate_number`
   - 比對成功且信心度高 → 自動寫入 `VEHICLE_LOG`（`source = camera`）
   - 比對失敗 / 信心度低 → 等待指揮中心 / 管理員人工審核後寫入

### 4. 車牌解鎖（中途換車）

1. 領隊建立 `PLATE_UNLOCK_REQUEST`，說明申請原因，`status = pending`
2. 指揮中心審核：
   - **通過** → `status = approved`，系統將 `plate_locked` 設為 `false`，領隊修改車牌後自動重新鎖定
   - **拒絕** → `status = rejected`

### 5. 回程

- 領隊按下「離開」→ 寫入 `VEHICLE_LOG`（`event_type = checkpoint_exit`）
- 攝影機偵測出口 → 比對成功後寫入 `VEHICLE_LOG`

### 6. 指揮中心覆蓋

- 可手動調整任意車輛狀態
- 寫入 `VEHICLE_LOG`，`source = command_center`，`override_by` 記錄操作者
- 原有紀錄保留，以最新一筆為當前狀態

---

## 設計原則

- **Append-only 動態紀錄**：`VEHICLE_LOG` 不覆蓋舊資料，以最新一筆作為當前狀態，完整保留稽核軌跡
- **攝影機事件分離**：`CAMERA_EVENT` 與 `VEHICLE_LOG` 解耦，信心度不足時不自動觸發狀態更新
- **彈性檢查點流程**：`order_index` 僅供顯示排序，車輛可視情況跳過特定檢查點
- **車牌鎖定機制**：出發後車牌不可直接修改，換車場景須走正式解鎖審核流程

---

## 相關文件

- [技術選型](Tech.md)
- [資料庫設計文件（ERD）](vehicle_management_erd.md)