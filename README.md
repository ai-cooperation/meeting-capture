# Meeting Capture Studio

> 手機錄音分享到 LINE，約 90 秒後在你**自己的** GitHub repo 收到結構化會議紀錄。
> 全雲端、零實體機器、全部跑在免費 tier。

把一場會議的錄音丟進 LINE，系統自動轉寫、校正、摘要成「議題 / 決議 / 未解 / 行動項目」，commit 到你自己的 repo，並回推一則摘要到 LINE。音檔不留存、資料全留在你手上。

> 開源釋放進行中：架構簡報先上，核心程式碼整理後陸續釋出。

## 特色

- **零依賴**：只用 Cloudflare Worker + GitHub Actions + 三方 API，沒有要顧的伺服器
- **零成本**：全部設計在免費 tier 內運作（個人用一場/天綽綽有餘）
- **資料主權**：音檔不留存，逐字稿／摘要全進你自己的 GitHub repo，可 diff、可版控
- **三層 STT fallback**：轉寫永不失敗
- **分層校正 + 自學習**：字典 + 模糊比對 + LLM 挖詞 + deterministic 驗證，跨場累積術語、越用越準
- **反幻覺摘要**：責任歸屬不編造，每條決議／行動附原文引用驗證

## 架構

```
手機錄音 App ──分享──▶ LINE Bot
                          │ webhook
                          ▼
                 Cloudflare Worker（驗簽 + per-user lock + dispatch）
                          │ repository_dispatch
                          ▼
                 GitHub Actions Pipeline
                  ├─ Stage 1  STT 三層 fallback
                  ├─ Stage 2  四層校正（字典 + jieba + LLM 挖詞 + gate）
                  ├─ Stage 3  5-stage 結構化摘要（反幻覺 + quote 驗證）
                  └─ Stage 5  commit meeting note + LINE 推播
                          │
                          ▼
                 你自己的 GitHub repo（meetings/ + logs）
```

資料流轉：**音檔 → 逐字稿 → 校正稿 → 結構化摘要 → meeting note**。
完整架構簡報見 [`slides/architecture-v2.html`](slides/architecture-v2.html)（瀏覽器開啟，← → 翻頁）。

## 為什麼是 Cloudflare Worker + GitHub Actions

- **Worker** 當 edge dispatcher：秒回避免 webhook timeout，只做驗簽 + lock + dispatch
- **GitHub Actions** 扛重活：30 分鐘 timeout 足以處理長音檔，產物直接進 repo，免費額度夠個人用
- 不用 GAS（6 分鐘上限）、不用自架 VM（要顧機器要錢）、不用 n8n（要 self-host）

## 自架（Self-host）

需要的服務（都有免費 tier）：

| 服務 | 用途 |
|---|---|
| Cloudflare Workers | webhook 接收 + dispatch |
| GitHub repo + Actions | 處理 pipeline + 存會議紀錄 |
| Groq | STT（Whisper） |
| Google Gemini / OpenRouter | 摘要 LLM |
| LINE Messaging API | 入口（手機錄音分享） |

步驟（概要，詳細待程式碼釋出後補 `docs/SETUP.md`）：

1. Clone 本 repo 到**你自己的** GitHub 帳號
2. 建一個 LINE Messaging API channel
3. 用 `wrangler` 部署 `worker/`（填你自己的 account / KV）
4. 在 GitHub repo 設定以下 Secrets（見 `.env.example`）：
   `GROQ_API_KEY`、`GEMINI_API_KEY_1`、`GEMINI_API_KEY_2`、`OPENROUTER_API_KEY`、`LINE_CHANNEL_SECRET`、`LINE_CHANNEL_TOKEN`、`RELEASE_TOKEN`
5. 把錄音分享到你的 LINE Bot，會議紀錄會 commit 進你的 repo

## 資料主權

這個系統**不幫你保管資料**——這是刻意的設計。音檔處理完不留存，逐字稿與摘要 commit 進**你自己的** repo。沒有任何第三方雲端持有你的會議內容。

## 授權

MIT（預定）。

---

本 repo 是 Meeting Capture Studio 的**開源版**，只含可公開的程式邏輯與文件，不含任何真實會議資料或密鑰。
