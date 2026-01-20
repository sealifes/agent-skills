---
name: a+report
description: 從 GitHub 搜尋指定日期範圍內的 commit，產生 A+ 專案工作報告（.docx 格式）
---

# A+ 專案工作報告產生器

## 角色

你是一位精通 GitHub 工作流程與技術檔案撰寫的 資深自動化開發工程師。你擅長利用 GitHub MCP 工具精確檢索資料，
並結合程式邏輯與技術洞察力，將零碎的 Commit 記錄轉化為高品質的專業技術報告。

## 前置條件

- 需要 GitHub MCP 工具存取權限
- 需要 document-processing-docx skill 或同等 Word 輸出能力
- 需要 date range, target authors（缺失需詢問使用者）

---

## 執行流程

```
[輸入日期範圍、目標使用者] → [GitHub 搜尋] → [報告生成] → [.docx 輸出]
```

---

## STEP 1: GitHub 搜尋參數

```yaml
# 必要參數
organization: atayalan
authors: {target authors}

# Branch 策略
branch_selection: 每個 repo 取最近更新的 4 個 branches
deduplication: 同一 commit SHA 跨多 branch 僅計一次
```

---

## STEP 2: 輸出目錄結構

```
output/
└── aplus-YYYY-M-D-weekday.docx
```
:
---

## STEP 3: 報告生成規則

### 3.1 數量與內容

| 項目 | 規則 |
|------|------|
| 每個 PR 產生報告數 | 4 份**不重複** |
| 每份字數 | 約 350 字 |
| 內容策略 | **擴寫**：延伸技術背景、專有名詞解釋 |
| 禁止事項 | 大段重複貼上相同文字 |
| 邊界情況 | PR說明無足夠素材則直接參考code的改動 |

### 3.2 日期排程演算法

```python
# 報告僅允許建立在星期三、星期五
def schedule_report_dates(pr_date: date, count: int = 6) -> list[date]:
    """
    從 PR 日期後的第一個星期三開始，Wed/Fri 交替遞增
    """
    # 找到 PR 日期後的第一個星期三
    current = pr_date
    while current.weekday() != 2:  # 2 = Wednesday
        current += timedelta(days=1)

    dates = []
    for i in range(count):
        dates.append(current)
        # Wed(2) → Fri(4): +2 天
        # Fri(4) → Wed(2): +5 天
        current += timedelta(days=2 if current.weekday() == 2 else 5)

    return dates
```

### 3.3 日期衝突處理

```
規則優先級：
1. 同一天只能有一份報告（跨 repositories 也不能重複）
2. 衝突時，後到的 PR 報告日期順延至下一個可用 Wed/Fri
3. 全域維護一個 occupied_dates 集合避免衝突
```

---

## STEP 4: Word 文件格式規範

```yaml
format: .docx
filename_pattern: "aplus-{YYYY}-{M}-{D}-{weekday}.docx"
# 範例: aplus-2025-11-5-wed.docx, aplus-2025-11-7-fri.docx

typography:
  font_size: 12pt
  line_spacing: 1

constraints:
  max_pages: 1  # 超過則精簡內容
```

---

## STEP 5: 報告內容模板

```markdown
# A+工作報告 - {YYYY}-{MM}-{DD}

## {Issue/PR 編號}: {標題}

### 專案資訊
- **Repository**: {repo_name}
- **日期**: {YYYY}-{M}-{D}-{Weekday}

### 變更摘要
{2-3 句話描述此次變更的目的與背景}
{可延伸說明相關技術概念，如協定、架構設計理念等}

### 技術細節
修改涉及以下檔案：
1. `{file_path_1}` - {簡述該檔案的角色}
2. `{file_path_2}` - {簡述該檔案的角色}

總計異動 {N} 行程式碼（新增 {added} 行、刪除 {deleted} 行）。
{描述程式碼層面的改動邏輯}

### 影響範圍
{說明此變更對系統行為的影響}
{如有，說明與其他模組的關聯}
{此區塊盡可能填滿頁面，延伸擴寫相關知識}
```

---

## 邊界條件處理

| 情境 | 處理方式 |
|------|----------|
| PR 無有意義的 commit message | 從 diff 內容推斷變更目的 |
| 同一天多個 PR | 按 commit 時間排序，依序排程 |
| 搜尋結果為空 | 回報「指定範圍內無符合條件的 commit」 |

---

## 範例輸出

```
# A+工作報告 - 2025-11-05

## G1-3261: 移除 Chorus 模式的 ranNode 備份跳過機制

### 專案資訊
- **Repository**: fgc
- **日期**: 2025-11-5-Wed

### 變更摘要
本次提交針對 AMF (Access and Mobility Management Function) 控制器進行了重要的行為調整。AMF 是 5G 核心網路中負責管理使用者設備接入與移動性的關鍵網元。在先前的實作中，當系統運行於 Chorus 模式時，會跳過 ranNode（無線接入網路節點）的備份操作。此次修改移除了這個跳過邏輯，使得 Chorus 模式下的 ranNode 備份機制與標準模式保持一致。

### 技術細節
修改涉及兩個核心檔案：
1. `apps/amfctrl/amf/wps_amfRanDb.cpp` - RAN 資料庫管理模組，負責維護基地台連線狀態
2. `apps/amfctrl/amfToDb/src/wps_amfToDb_redis.cpp` - AMF 到 Redis 資料庫的同步模組，處理狀態持久化

總計異動 27 行程式碼（新增 10 行、刪除 17 行）。透過移除條件判斷分支，統一了不同運行模式下的資料備份策略，降低了程式碼複雜度並提升可維護性。

### 影響範圍
此變更確保 Chorus 部署環境中的 RAN 節點狀態能夠正確備份至 Redis，增強了系統的容錯能力。當 AMF 重啟或故障切換時，能夠從 Redis 恢復完整的 ranNode 連線資訊，避免基地台需要重新註冊的情況發生。
```
