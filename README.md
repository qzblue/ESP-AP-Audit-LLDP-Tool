# 🛰️ AP 審計工具（AP Audit LLDP Tool v16）

[![Python](https://img.shields.io/badge/Python-3.9%2B-blue)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Status](https://img.shields.io/badge/狀態-穩定-success.svg)]()

> **基於 LLDP 的企業級 AP 網路審計工具**  
> 僅以 **LLDP neighbor-information** 判定 AP 的「在線」與「速率（100M / 1000M）」；  
> 以 **AC 的 `display wlan ap all`** 作為權威全集；  
> `display interface brief` 只用於統計「描述含 AP、但實際 LINK=DOWN」的遺留端口。

---

## 📖 目錄

1. [項目簡介](#項目簡介)
2. [主要功能](#主要功能)
3. [系統需求](#系統需求)
4. [安裝步驟](#安裝步驟)
5. [資料結構](#資料結構)
6. [輸入格式與範例](#輸入格式與範例)
7. [使用說明](#使用說明)
8. [命令參數說明](#命令參數說明)
9. [輸出結果說明](#輸出結果說明)
10. [測試與範例運行](#測試與範例運行)
11. [驗證規則](#驗證規則)
12. [常見問題](#常見問題)
13. [貢獻指南](#貢獻指南)
14. [授權條款](#授權條款)
15. [作者資訊](#作者資訊)

---

## 🧭 項目簡介

`ap_audit_lldp_v16.py` 是一個用於企業網路管理的 Python 工具，能自動分析 AC 與交換機日誌，產出包含 **在線 AP 狀態、速率統計、離線與遺留端口** 的 Excel 報表。

### 核心流程（口徑）

| 步驟 | 資料來源 | 功能說明 |
|------|-----------|-----------|
| 1️⃣ | `display wlan ap all` | 從 AC 擷取 AP 名單（權威全集） |
| 2️⃣ | `LLDP neighbor-information` | 解析 AP 名稱（System name）與 **OperMau** 速率（100/1000）→ 決定「在線/速率」 |
| 3️⃣ | `display interface brief` | 僅用於標識 **描述含 "AP" 且 LINK=DOWN** 的端口（遺留） |
| 4️⃣ | AC × LLDP 比對 | AC 有但 LLDP 無 → Offline |
| 5️⃣ | 匯出報表 | 生成多工作表 Excel 結果（含校驗） |

---

## ⚙️ 主要功能

- ✅ 自動解析多台交換機 LLDP（支援 H3C/Cisco/Huawei 等常見格式）  
- ✅ 匯總 **百兆 / 千兆** 在線 AP 清單  
- ✅ 精準標註 **離線 AP**（AC 有、LLDP 無）與 **遺留端口**（desc 中含 AP 且 LINK=DOWN）  
- ✅ 產出 **多工作表 Excel 報表**（UTF-8）  
- ✅ 內建 **完整性校驗**（在線 = 1000M + 100M）

---

## 💻 系統需求

| 項目 | 最低需求 |
|------|-----------|
| 作業系統 | Windows / Linux / macOS |
| Python 版本 | ≥ 3.9 |
| 依賴套件 | pandas, xlsxwriter |

安裝依賴：
```bash
pip install -r requirements.txt
# 或
pip install pandas xlsxwriter
```

---

## 📂 資料結構

建議的專案結構：
```
AP-Audit-LLDP/
├── ap_audit_lldp_v16.py          # 主程式
├── README.md                     # 本文件
├── requirements.txt              # 依賴
├── LICENSE                       # 授權
├── examples/
│   ├── sample_ac_log.log         # AC 日誌範例
│   └── sample_switch_log.log     # 交換機日誌（LLDP + brief）範例
├── data/
│   ├── ac_logs/                  # 你的 AC 日誌
│   └── switch_logs/              # 你的交換機日誌
└── output/
    └── AP_Statistics_v16_LLDP.xlsx
```

---

## 📥 輸入格式與範例

### 1) AC：`display wlan ap all`
- 每行包含一個 AP 名稱，**慣例形如**：`xxx_apN`
- 例：
```
sec_101_ap1                    2     R/M   WA5320i         219801A1BX8196E0010W
sec_102_ap1                    3     R/M   WA5320i         219801A1BX8196E00112
...
```

### 2) 交換機：`LLDP neighbor-information`
- 關鍵欄位：`System name`（AP 名稱）、`OperMau : speed(100|1000)`  
- 例：
```
LLDP neighbor-information of port 3[GigabitEthernet1/0/3]:
  System name : n_1f_ap3
  OperMau     : speed(1000)/duplex(Full)
```
→ 代表 AP **n_1f_ap3** 在線，速率 **1G 全雙工**。

### 3) 交換機：`display interface brief`（只用於遺留端口）
- 用來找 **描述含 "AP" 且 LINK=DOWN** 的端口：
```
GE1/0/22  DOWN  1G(a)  F(a)  A  1322  sec_402_ap1 downlink
```

---

## 🚀 使用說明

```bash
python ap_audit_lldp_v16.py \
  --ac-logs ./data/ac_logs \
  --lldp-logs ./data/switch_logs \
  --out ./output/AP_Statistics_v16_LLDP.xlsx
```

> 可同時指定多個 `--lldp-logs` 目錄，工具會自動彙整。

---

## ⚙️ 命令參數說明

| 參數 | 說明 | 範例 |
|------|------|------|
| `--ac-logs` | AC 日誌所在資料夾 | `./data/ac_logs` |
| `--lldp-logs` | 交換機日誌資料夾（可多個） | `./data/switch_logs ./data/CRT_logs` |
| `--out` | Excel 匯出路徑與檔名 | `./output/AP_Statistics_v16_LLDP.xlsx` |

---

## 📊 輸出結果說明

執行完成後會生成一份 Excel，包含以下工作表：

| 工作表 | 說明 | 主要欄位 |
|--------|------|----------|
| **統計摘要_LLDP口徑** | 總覽統計與校驗 | 項目、數值 |
| **在線AP_LLDP** | 所有在線 AP | ap_name、AC控制器、switch_log、interface、uplink_speed、AC_log |
| **在線AP_100M_LLDP** | 在線且速率為 100M 的 AP | 同上 |
| **在線AP_1000M_LLDP** | 在線且速率為 1G 的 AP | 同上 |
| **Offline_AP(AC有LLDP無)** | AC 有但 LLDP 無的 AP | ap_name、AC控制器、AC_log |
| **遺留AP_描述含AP但DOWN** | 描述含 "AP" 且 LINK=DOWN 的端口 | switch_log、iface、link、pvid、desc |

### 校驗（自動）
- 在線 = 在線_100M + 在線_1000M ✅  
- Offline = AC 總數 − 在線 ✅

---

## 🧪 測試與範例運行

### 1) 準備測試資料
```
examples/
├── sample_ac_log.log
└── sample_switch_log.log
```

### 2) 執行
```bash
python ap_audit_lldp_v16.py \
  --ac-logs ./examples \
  --lldp-logs ./examples \
  --out ./output/demo.xlsx
```

### 3) 終端輸出範例
```
開始解析 AC 日誌...
已讀取 AP 數量：176
解析 LLDP 鄰居資訊中...
匹配到在線 AP：174
離線 AP 數量：2
報表輸出完成： ./output/demo.xlsx
```

### 4) 驗證
- 打開 `demo.xlsx`，在 `統計摘要_LLDP口徑` 中確認校驗為 ✅；
- 在線 AP 均有 `uplink_speed`（100M/1G）；
- Offline 清單與 AC 清單一致。

---

## ✅ 驗證規則

| 項目 | 規則 |
|------|------|
| **速率來源** | 僅來自 LLDP `OperMau`（`speed(100)`→100M；`speed(1000)`→1G） |
| **在線 AP** | 出現在 LLDP 鄰居資訊中 |
| **離線 AP** | 出現在 AC 清單但 **未** 出現在 LLDP |
| **遺留端口** | `display interface brief`：描述含 "AP" 且 LINK=DOWN |
| **完整性** | 在線 = 100M + 1000M ✅ |

---

## 🔍 常見問題

| 問題 | 可能原因 | 處理方式 |
|------|----------|----------|
| Excel 為空 | AC 日誌未含 `_ap` 標識 | 檢查 AC log 格式或採集指令 |
| 在線數過少 | 交換機未啟用 LLDP / 日誌不完整 | 啟用 LLDP，使用 **verbose** 模式導出 |
| 無速率 | LLDP 缺少 `OperMau` 欄位 | 改用 `display lldp neighbor-information verbose` |
| 中文亂碼 | Windows 預設編碼導致 | 執行 `chcp 65001`，或確保檔案為 UTF-8 |

---

## 🤝 貢獻指南

1. Fork 本倉庫  
2. 建立新分支：`git checkout -b feature/your-feature`  
3. 提交修改：`git commit -m "Add new feature"`  
4. 推送：`git push origin feature/your-feature`  
5. 發起 Pull Request

---

## 📜 授權條款

本專案採用 [MIT License](LICENSE)。

---

## 👤 作者資訊

- 作者：朱振宇（Zhu Zhenyu）  
- 單位：澳門聖保祿學校（St. Paul School, Macau）資訊部  
- 版本：v16  
- 聯絡：請於 GitHub Issues 建立問題單

---

### 🧭 LLDP 資料範例（解析樣本）
```
LLDP neighbor-information of port 3[GigabitEthernet1/0/3]:
  System name : n_1f_ap3
  OperMau     : speed(1000)/duplex(Full)
```
→ 判定：AP **n_1f_ap3** 在線、速率 **1G 全雙工**。

> 建議在導出前於交換機執行：  
> `display lldp neighbor-information verbose` 與 `display interface brief`。
