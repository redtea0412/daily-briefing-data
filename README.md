# 每日財經晨報 — 資料

這個 repo **只放資料，不放任何程式碼**。

內容是每天自動產生的財經簡報，由 AI 彙整國際新聞與市場行情而成。
Android App 直接從這裡讀取。

> ⚠️ 內容由 AI 自動產生，可能有錯誤或遺漏，僅供參考，不構成任何投資建議。

---

## 這個 repo 在整個系統的位置

```
[後端 Python] 每天 06:00（台北）由 GitHub Actions 執行
      ↓ 抓 10 個 RSS 來源 + 27 檔行情 → Gemini 產生報告
      ↓ 寫成 JSON 並 git push
[就是這裡：daily-briefing-data]
      ↓ raw.githubusercontent.com
[Android App] 存進本機資料庫，離線也能看
```

後端程式在另一個私有 repo，不在這裡。

**這個 repo 必須維持公開**，App 是透過 `raw.githubusercontent.com` 讀取的，
一旦改成私有就需要授權，App 會立刻壞掉。

---

## 檔案結構

```
.
├── index.json          目錄：所有日期與標題的清單
└── data/
    ├── 2026-07-28.json 單日簡報（完整內容）
    ├── 2026-07-29.json
    └── …
```

App 的讀取順序是：先抓 `index.json` 知道有哪幾天，再按需要抓 `data/<日期>.json`。

---

## JSON 格式（跨專案契約）

> 📌 **這一節是後端與 App 之間唯一的介面。**
>
> 欄位名稱同時出現在兩個地方，改動時**兩邊必須一起改**：
>
> - 後端：`daily-briefing/exporter.py`
> - App：`daily-briefing-app/.../data/remote/dto/Dtos.kt`
>
> 只改一邊的話 App 不會報錯，而是**靜默地顯示空白**——這種 bug 特別難找。

### `index.json`

```json
{
  "updated_at": "2026-09-19T06:03:12+08:00",
  "briefings": [
    { "date": "2026-09-19", "title": "聯準會升息美股跌，台股逆勢走揚" },
    { "date": "2026-09-17", "title": "聯準會決策前市場觀望，AI支出估增49.5%" }
  ]
}
```

| 欄位 | 型別 | 說明 |
|---|---|---|
| `updated_at` | string | ISO 8601，含 `+08:00` 時區 |
| `briefings` | array | **由新到舊排序**，App 直接照順序顯示，不會自己再排 |
| `briefings[].date` | string | `YYYY-MM-DD`，同時也是 `data/` 底下的檔名 |
| `briefings[].title` | string | 20 字以內，由 Gemini 產生 |

上限 365 筆（`config.py` 的 `INDEX_MAX_ENTRIES`），超過會丟棄最舊的。

### `data/<日期>.json`

```json
{
  "date": "2026-09-19",
  "generated_at": "2026-09-19T06:03:12+08:00",
  "title": "聯準會升息美股跌，台股逆勢走揚",
  "summary": "## 一、今日重點快速提醒\n- …",
  "market": [
    { "ticker": "^GSPC", "name": "S&P 500", "price": 7377.71, "change_pct": 0.84 },
    { "ticker": "2330.TW", "name": "台積電", "price": null, "change_pct": null }
  ],
  "sources": [
    {
      "headline": "US launches strikes on Iran…",
      "url": "https://www.bbc.co.uk/news/articles/…",
      "thumbnail": "https://ichef.bbci.co.uk/…jpg",
      "publisher": "BBC World",
      "published_at": "2026-09-19T18:49:14+08:00"
    }
  ]
}
```

| 欄位 | 型別 | 可為 null | 說明 |
|---|---|---|---|
| `date` | string | ✗ | `YYYY-MM-DD` |
| `generated_at` | string | ✓ | ISO 8601 |
| `title` | string | ✗ | 20 字內 |
| `summary` | string | ✗ | **Markdown 格式**，見下方限制 |
| `market` | array | ✗ | 約 27 筆 |
| `sources` | array | ✗ | 約 20 筆 |

**`market[]`**

| 欄位 | 型別 | 可為 null | 說明 |
|---|---|---|---|
| `ticker` | string | ✗ | `^` 開頭代表大盤指數，App 靠這個判斷要不要預設顯示 |
| `name` | string | ✗ | 顯示名稱 |
| `price` | number | ✓ | **是數字不是字串**。抓不到時為 `null`，App 顯示「—」 |
| `change_pct` | number | ✓ | 同上。正數綠色、負數紅色 |

**`sources[]`**

| 欄位 | 型別 | 可為 null | 說明 |
|---|---|---|---|
| `headline` | string | ✗ | 新聞標題 |
| `url` | string | ✗ | 原文網址。沒有網址的項目 App 會直接濾掉 |
| `thumbnail` | string | ✓ | og:image。抓不到時為 `null`，App 顯示佔位圖 |
| `publisher` | string | ✗ | 媒體名稱 |
| `published_at` | string | ✓ | ISO 8601 |

### `summary` 的 Markdown 限制

App 用的是自製的輕量渲染器（`MarkdownText.kt`），只支援這些語法：

✅ 標題（`##`、`###`）、清單、引言、分隔線、粗體、斜體、行內程式碼、表格

❌ 巢狀清單、圖片、HTML、程式碼區塊

**表格一律三欄「名稱｜收盤｜漲跌」**，多於三欄在手機螢幕上會擠爆。
這條規則寫在後端 `briefing.py` 的 `SYSTEM_PROMPT` 裡。

---

## 常見疑問

**為什麼 `price` 有時候是 `null`？**

Yahoo Finance 偶爾抓不到某些標的（代號失效、當天沒交易、API 暫時異常）。
後端刻意不讓這種狀況中斷整個流程，抓不到就填 `null`，其他照常。

**為什麼瀏覽器看到的是舊資料？**

`raw.githubusercontent.com` 有約 5 分鐘的 CDN 快取。按 `Ctrl + Shift + R` 強制重新整理。

App 不受影響，它每次請求都會附上時間戳避開快取。

**某一天沒有檔案，是壞了嗎？**

不一定。GitHub Actions 的排程可能延後或極少數情況跳過，
另外 repo 連續 60 天沒動靜時排程會被自動停用（GitHub 會先寄信通知）。

到 `daily-briefing-bot` repo 的 Actions 分頁看執行紀錄就知道原因，
也可以在那裡按 **Run workflow** 手動補跑。

**能不能直接改這裡的 JSON？**

可以，但下次排程執行時會被覆蓋。要調整內容應該改後端的 prompt。
