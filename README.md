# 幻世錄 重製版 · Steam 評論監控

自動收集《幻世錄 重製版》（Steam appid `4030150`）的玩家評論與社群討論版，
產出可互動的議題彙整儀表板。

**給客服看的問題回報日報：** https://<你的帳號>.github.io/<repo 名稱>/
**完整儀表板（議題彙整、好評率、語言分佈）：** https://<你的帳號>.github.io/<repo 名稱>/full.html

客服端不需要任何操作 — 存一個書籤，打開就是今天的負評與故障回報，按日期分組、可一鍵複製當日清單。

## 這個東西做什麼

| 資料 | 來源 |
|---|---|
| 玩家評論全文 | Steam 官方 `appreviews` API（公開、免金鑰，cursor 分頁） |
| 各語言評分摘要 | 同上，`num_per_page=0` |
| 討論版主題 | `steamcommunity.com/app/4030150/discussions/` |

輸出 `docs/index.html`：好評率、每日走勢、各語言分佈、**17 類議題彙整**
（同議題的評論與討論串併在一起），以及可全文搜尋、依語言／好評負評／議題篩選的評論瀏覽器。

## 自動更新

`.github/workflows/update.yml` 每天台北時間 10:00 執行：抓取 → 去重合併進
`data/reviews_4030150.jsonl` → 重建 `docs/` → 自動 commit。
也可以到 Actions 頁面按 **Run workflow** 手動觸發。

## 啟用 GitHub Pages

Settings → Pages → Source 選 **Deploy from a branch**，branch 選 `main`、資料夾選 **`/docs`**。

## 本機執行

```bash
pip install requests
python steam_watch.py init
python steam_watch.py fetch --full      # 第一次抓全部
python steam_watch.py report            # 產出 reports/dashboard_4030150.html
python steam_watch.py fetch             # 之後只抓新的
```

## 設定

打開 `steam_watch.py`：

- `APPS` — 要追蹤的 appid
- `THEMES` — 議題關鍵詞表，想追新議題就加一列
- `SINCE_DEFAULT` — 只採計此日期之後的評論（預設 `2026-09-09`，正式版上線日）

## 資料說明

- `data/reviews_4030150.jsonl` — 累積的評論資料，每行一筆
- 評論總數以 Steam 商店頁顯示為準；API 的 `purchase_type=all` 會多算非 Steam 購買的評論
- 議題分類是關鍵詞比對，用於快速定位，細節請點開原文確認

## 授權

MIT（腳本）。評論內容著作權屬於各自的 Steam 使用者，此處僅作彙整與引用。
