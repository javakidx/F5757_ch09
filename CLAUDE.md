# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> 文件撰寫與回應一律使用繁體中文。

## 專案概述

**TaskMaster** 是一個個人任務管理系統，同時提供 Tkinter 桌面 GUI 與 Flask Web API 兩種介面，資料儲存於本地 SQLite (`tasks.db`)。

⚠️ **重要：本專案是「重構教學用」的範例程式碼，刻意保留大量不良實作（程式碼異味、安全漏洞、重複邏輯）。** 原始碼中的英文註解（如 `BAD!`、`VERY BAD!`、`Security risk!`）與中文註解都是在標記待修正的問題點。修改程式時請以「重構改善」為目標，不要模仿現有風格。

## 常用指令

```bash
# 安裝相依套件（擇一）
uv sync                         # 依 pyproject.toml / uv.lock 安裝
pip install -r requirements.txt

# 啟動桌面 GUI（main.py 內的 TaskGUI）
uv run python main.py gui

# 啟動 Web API（main.py 內的 Flask app，port 5000）
uv run python main.py api

# 啟動另一套獨立的 API（api_server.py，port 8080，路由與 main.py 不同）
uv run python api_server.py

# 檢視資料庫表格結構
uv run python check_db.py
```

> 目前專案沒有任何測試框架或測試檔案，也沒有設定 lint / formatter。

## 架構與檔案職責

各檔案開頭的中文 docstring 通常會說明「此檔案應該做什麼」與「目前的問題」，是理解重構意圖的關鍵。

| 檔案 | 現況職責 | 設計意圖（重構目標） |
|------|----------|----------------------|
| `main.py` | 一個檔案塞入全部功能：DB 操作、`Task` 類別、`TaskGUI`、Flask 路由、進入點 | 應只做為入口，呼叫各模組功能 |
| `api_server.py` | 獨立的 Flask app（路由 `/tasks`，port 8080） | 應收容由 `main.py` 切分出來的 API；目前功能不全（缺刪除、狀態轉換） |
| `task_gui.py` | 另一套 `TaskManagerGUI`（Treeview 版） | 應收容由 `main.py` 切分出來的 GUI；目前與 `main.py` 的 GUI 衝突重複 |
| `database.py` | `DatabaseManager` 類別 + 全域 `db_manager` 實例 | 應為唯一的 DB 存取層，但目前其他檔案都各自直連 sqlite |
| `config.py` | 集中設定（含明文密鑰） | 設定來源 |
| `utils.py` | 雜項工具（日期、雜湊、驗證、備份、log） | 工具函式集 |
| `backup.py` | 未完成的舊版備份/GUI 殘骸 | 已廢棄 |
| `check_db.py` | 列印資料表結構的小腳本 | 開發輔助 |

### 資料模型

所有介面共用單一資料表 `tasks`：
`id` (PK)、`title`、`description`、`priority`(low/medium/high)、`status`(pending/completed 等)、`created_at`。

注意：`main.py` 用字串存 `created_at`，`database.py` 用 `TIMESTAMP DEFAULT CURRENT_TIMESTAMP`，兩者建表定義不一致。

## 主要相依套件

- **Flask 3.0.3** — Web API 框架（`main.py`、`api_server.py`）。
- **requests >= 2.32.5** — HTTP 用戶端（`main.py` 有 import，目前未實際使用）。
- **tkinter** — 桌面 GUI，Python 內建，不在 requirements.txt 列出。
- **sqlite3** — 資料庫，Python 內建。
- 套件管理同時存在 `pyproject.toml` + `uv.lock`（uv）與 `requirements.txt`（pip）兩套，需注意一致性。

## 執行環境

- **Python：** `requires-python >= 3.11.8`（`.python-version` 指定 `3.11.8`）。
- **套件管理器：** 建議使用 `uv`（有 `uv.lock`），亦可用 `pip`。
- **資料庫：** 本地 SQLite 檔 `tasks.db`，與程式同目錄、路徑為硬編碼。
- **服務埠：** `main.py api` → `0.0.0.0:5000`；`api_server.py` → `0.0.0.0:8080`。兩者都綁定 `0.0.0.0` 並開啟 debug，僅適合本機開發。
- **GUI：** 需要本機圖形環境；在純文字 / WSL 無 X server 時無法啟動 Tkinter。

## 程式碼風格問題（現存待改善）

這些是刻意留下的反面教材，重構時應一併處理：

- **全域變數氾濫：** `main.py` 使用 `db_connection`、`app`、`window`、`current_user`、`DEBUG_MODE` 等全域狀態。
- **命名與空白不一致：** 如 `self.title=title`、`self.desc= desc`、`self.priority =priority`，運算子前後空白混亂；參數命名 `desc`/`description` 混用。
- **每次操作都新建 DB 連線：** `add_task`/`get_tasks`/`update_task` 等各自 `sqlite3.connect()`，未共用連線池。
- **錯誤處理粗糙：** 多處 `except:` 裸捕捉並僅 `print`，吞掉例外。
- **回應格式不統一：** `main.py` 直接 `jsonify(tasks)`（裸 tuple 陣列）或回傳純字串 `"OK"`/`"DELETED"`；`api_server.py` 則回傳結構化 dict，兩套 API 不相容。
- **輸入驗證不足：** API 直接 `data['title']` 取值，缺少 content-type 與欄位驗證，易發生 `KeyError`。
- **脆弱的 UI 解析：** `get_task_id_from_selection` 以字串切割還原 task id。

## 已知架構問題（重構重點）

- **單體 God file：** `main.py` 混合資料層、GUI、API、進入點，違反其 docstring 宣告的「只當入口」。
- **重複且衝突的實作：** GUI 有兩套（`main.py` 的 `TaskGUI` vs `task_gui.py` 的 `TaskManagerGUI`）；API 有兩套（`main.py` 的 `/api/tasks` vs `api_server.py` 的 `/tasks`）；備份邏輯重複於 `main.py` 與 `utils.py`。
- **資料存取未集中：** 已有 `database.py` 的 `DatabaseManager`，但幾乎沒人使用，各檔案仍直連 sqlite，建表定義也分歧。
- **設定散落：** `config.py` 與 `main.py` 各自定義 `API_KEY`、`DEBUG`、`current_user`/`DEFAULT_USER`，未統一來源。

## 安全性問題（最優先處理）

帳號與密碼等敏感資訊務必安全保存。本專案目前嚴重違反此原則，修改時應優先修復：

- **明文硬編碼機密：** `main.py` 與 `config.py` 內有 `API_KEY = "sk-1234567890abcdef"`、`SECRET_KEY = "your-secret-key-here"`，應改由環境變數注入。
- **弱密碼雜湊：** `utils.py` 的 `hash_password` 使用 MD5，應改用 `bcrypt`/`argon2` 等。
- **Debug 模式 + 對外綁定：** 兩個 Flask 服務皆 `host='0.0.0.0'` 並啟用 debug，正式環境須關閉。
- **預設管理者身分：** 全域 `current_user = "admin"` / `DEFAULT_USER = "admin"`，無實際認證機制。

（SQL 注入問題在 `main.py` 的 `update_task`/`delete_task` 已改用參數化查詢，新增程式碼也須沿用參數化寫法。）
