# Supabase 暫停、保活與備份紀錄

## 2026-11-20：專案被暫停後還原

- 專案 `community-facility-manager`（ref `liqvunlcqlxkvyzggyxe`）因閒置被暫停，還原時狀態為 `RESTORING`，比以往慢；成因未確認（暫停時間長短無法從 API 看到）。
- 還原完成後唯讀檢查（已驗證）：`church_speaker_roster` 共 21 筆、日期 2026-12-06 ～ 2027-11-14、最後更新 2026-11-12；SELECT／INSERT／UPDATE／DELETE 四個 anon policy 都在。
- 沒驗證：暫停前的筆數（沒有基準可比對），所以無法證明沒有遺失資料。

## 決定：保活與備份分開做（Jean 決定，2026-11-20）

| 項目 | 做法 | 為什麼 |
| :-- | :-- | :-- |
| 保活 | GitHub Actions `.github/workflows/keepalive.yml`，每 3 天用公開 anon key 查一次資料表 | 電腦不用開機 |
| 備份 | 本機工作排程器 `church-speaker-roster-backup`，每週一 12:00 執行 repo 外的 `backup.ps1`，保留最近 12 份 | 備份含人名與經歷，repo 是公開的，不能放進 repo |

沒選的做法：只用 Actions（無法安全存備份）、只用本機排程（電腦沒開機就不跑）。

## 已知限制

- 公開 repo 若 60 天沒有活動，GitHub 會自動停用排程 workflow，需到 Actions 頁面手動重新啟用。
- 本機排程只在這台電腦開機時跑；設了「錯過就補跑」。
- 備份存放在 `D:\Git\church-speaker-roster-backups\`（不在 repo 內，不可 commit）。
- 這台電腦的網路回應標頭格式不標準（`curl` 回傳 exit 8、PowerShell 預設解析失敗），備份腳本因此啟用寬鬆標頭解析；成因未確認。
- Actions 排程尚未實際跑過（需合併到 `main` 後才會啟用）；本機備份腳本已手動測試成功（21 筆）。
