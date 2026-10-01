# Supabase 刪除功能踩坑紀錄

## 2026-10-01：刪除按鈕沒有作用

### 症狀

主日月曆浮動視窗已經顯示「刪除」按鈕，輸入原登錄人姓名並確認後，資料仍然沒有被刪除。

### 根因

前端事件有正常觸發，程式也確實送出：

```js
supabaseClient
  .from('church_speaker_roster')
  .delete()
  .eq('id', record.id)
```

但 Supabase 資料表的 RLS policies 只有：

- `SELECT`：公開讀取
- `INSERT`：公開新增
- `UPDATE`：公開修改

缺少 `DELETE` policy，因此 Supabase 會拒絕匿名前端的刪除請求。這不是 GitHub Pages 快取問題，也不是按鈕沒有綁定事件。

### 診斷方式

先查詢資料表 policies：

```sql
select policyname, roles, cmd, qual
from pg_policies
where schemaname = 'public'
  and tablename = 'church_speaker_roster';
```

如果結果沒有 `cmd = 'DELETE'`，就會重現這個問題。

本次確認到的原有 policies：

```text
church_roster_public_insert  INSERT  anon
church_roster_public_read    SELECT  anon
church_roster_public_update  UPDATE  anon
```

### 修正方式

本專案是小型、公開使用的教會排班系統，決定採用簡單的公開 DELETE policy：

```sql
create policy "church_roster_public_delete"
on public.church_speaker_roster
for delete
to anon
using (true);
```

套用後再次查詢確認：

```text
church_roster_public_delete  DELETE  anon  qual=true
```

### 前端目前的刪除流程

1. 使用者在月曆點選已排班日期。
2. 點擊「刪除」。
3. 輸入原登錄人姓名。
4. 再次確認。
5. 呼叫 Supabase DELETE。
6. 成功後關閉視窗並重新載入排班資料。

### 下次遇到類似問題的順序

1. 先確認按鈕事件是否有觸發。
2. 檢查瀏覽器 console 是否出現 Supabase error。
3. 查 `pg_policies` 是否有對應的操作 policy。
4. 確認 Data API 與 RLS 都允許該操作。
5. 修正 policy 後重新查詢 policy，最後再用網站測試。

> 注意：目前沒有 Supabase Auth，輸入登錄人姓名只是前端流程上的確認，不是真正的身分驗證。若未來資料敏感度提高，應改用 Auth 與依使用者身分限制的 RLS policy。
