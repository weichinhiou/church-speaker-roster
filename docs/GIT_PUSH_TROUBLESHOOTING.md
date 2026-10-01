# Git Push 失敗自救手冊（給 Codex／任何 AI Agent）

## 背景

2026-09-22 這次 `church-speaker-roster` push 失敗，Codex 把錯誤訊息 `SEC_E_NO_CREDENTIALS` 直接解讀成「認證/執行環境問題」，寫了一份交接文件請 Claude 出手，最後 Jean 自己手動在 GitHub 網頁上改了檔案，才發現真正原因跟認證完全無關——是 **push 被拒絕（rejected），不是認證失敗（failed）**。

這份文件教你下次自己先判斷、自己先修，不要看到一個陌生錯誤訊息就假設是「環境差異」。

---

## 第一步：永遠先分清楚「認證失敗」還是「push 被拒絕」

執行 `git push` 之後，看**錯誤訊息的關鍵字**，這決定了完全不同的處理方向：

| 看到的關鍵字 | 意思 | 該往哪查 |
| :-- | :-- | :-- |
| `Authentication failed` `SEC_E_NO_CREDENTIALS` `could not read Username` `403` | 認證層失敗，Git 連 GitHub 都還沒建立起有效連線 | 檢查 credential helper、token、SSH key（見第三節） |
| `! [rejected]` `fetch first` `non-fast-forward` | **認證其實成功了**，Git 已經連上 GitHub，只是遠端有你本機沒有的新 commit，GitHub 正常拒絕覆蓋 | 這不是認證問題，直接跳到第二節做 fetch + rebase |
| `denied to <user>` `Permission to ... denied` | 認證成功，但這個帳號對這個 repo 沒有寫入權限 | 檢查是不是登入了對的 GitHub 帳號、repo 權限設定 |

**這次的教訓**：`SEC_E_NO_CREDENTIALS` 這個訊息看起來很像認證問題，但只要你**實際重新執行一次 `git push`**，就會發現它其實走到了 `! [rejected]`——代表認證早就沒問題了。不要只看錯誤訊息的表面文字就下結論，永遠先動手重試一次、看清楚最新的錯誤訊息再判斷。

---

## 第二步：`[rejected]` / `fetch first` 的標準處理流程

這是最常見的情況（本機分支落後遠端），流程固定，照抄即可：

```bash
# 1. 先看目前狀態，搞清楚自己在哪個分支、乾不乾淨
git branch --show-current
git status -sb

# 2. 抓遠端最新狀態（這一步不會改動你的檔案，安全）
git fetch origin

# 3. 比對本機分支跟 origin 分支的差異，搞清楚兩邊各自多了什麼
git log --oneline main..origin/main   # 遠端有、本機沒有的 commit
git log --oneline origin/main..main   # 本機有、遠端沒有的 commit

# 4. 看兩邊改的是不是同一個檔案（決定會不會衝突）
git diff --name-only <共同祖先 commit> main
git diff --name-only <共同祖先 commit> origin/main
# 共同祖先可以用：
git merge-base main origin/main
```

看完這些，你就知道：
- 如果兩邊改的檔案完全不重疊 → 直接 `git rebase origin/main`，大機率不會衝突。
- 如果兩邊改了同一個檔案 → 会衝突，繼續看第三步，不要慌，衝突是正常的，一步一步解就好。

**不要做的事**：不要因為看到 `[rejected]` 就直接 `git push --force`。這會把遠端別人（或你自己在別的地方）新增的 commit 蓋掉，資料會不見。`--force` 只有在你**明確知道**遠端那些 commit 不需要保留、而且經過確認時才能用。

---

## 第三步：處理 rebase 衝突（不要怕，照這個順序做）

```bash
git rebase origin/main
```

如果出現 `CONFLICT`：

```bash
# 1. 看哪些檔案衝突
git status
# 或更明確：
git diff --name-only --diff-filter=U

# 2. 打開衝突檔案，會看到這種標記
# <<<<<<< HEAD                  ← 這是 origin（遠端）的版本
# ...origin 的內容...
# =======
# ...你自己 commit 的內容...
# >>>>>>> <你的 commit hash> (你的 commit message)

# 3. 判斷怎麼合併：
#    - 如果兩邊是「新增了不同功能」，通常是兩邊都留（刪掉標記行就好）
#    - 如果兩邊改了同一段邏輯，要讀懂兩邊在做什麼，選一個正確版本，
#      或手動整合成同時滿足兩邊需求的版本
#    - 拿不準就先用 git show <commit> -- <檔案> 看該 commit 原始的完整 diff，
#      理解這個 commit「原本想做什麼」，再決定怎麼合併

# 4. 確認沒有殘留衝突標記（這一步不能省）
grep -n "^<<<<<<<\|^=======\|^>>>>>>>" <檔案>
# 沒有輸出才算真的解完

# 5. 標記解決、繼續 rebase
git add <檔案>
git rebase --continue

# 如果還有下一個 commit 衝突，重複 3～5

# 中途想放棄、回到 rebase 前的狀態
git rebase --abort
```

**不要做的事**：不要用 `git add -A` 或 `git checkout --theirs / --ours` 整檔案二選一去「解決」衝突，除非你確定其中一邊的版本是完全過時、不需要保留的。這次的衝突是「兩邊各自新增了不同功能」，正確做法是兩邊都留、不是二選一。

---

## 第四步：合併完，push 前的最後檢查

```bash
# 確認 rebase 後的分支跟 origin 的差異是「預期中」的（只多不少）
git diff --stat origin/main main

# 如果專案有本機可測（網頁、程式），先實際跑一次，
# 不要只憑「檔案看起來合併對了」就 push
```

確認沒問題後才 push：

```bash
git push origin main
```

---

## 第五步：如果真的是認證問題（`Authentication failed` 這類）

跟 rejected 完全不同的處理方向：

```bash
# 檢查目前用的 credential helper
git config --show-origin --get-all credential.helper

# 檢查 Git 版本與執行檔路徑（不同環境可能用到不同的 git.exe）
git --version
where git        # Windows
which git        # macOS/Linux

# 讓 GitHub CLI 重新設定 git 的認證串接
gh auth status
gh auth setup-git

# 如果 HTTPS + Credential Manager 一直有問題，改用 SSH（避開 Schannel/HTTPS 認證層）
git remote set-url origin git@github.com:<owner>/<repo>.git
git push origin main
```

如果換了 SSH 還是不行，才需要往「執行環境（sandbox、使用者權限、APPDATA 路徑）跟平常操作 Git 的環境不一樣」這個方向查——但**這應該是排除了「push 被拒絕」這個更常見、更簡單的可能性之後，才輪到的第二順位懷疑**，不要一開始就跳去懷疑環境。

---

## 第六步：連 `git fetch` 都失敗（`SEC_E_NO_CREDENTIALS` 在 TLS 握手階段就出現）

2026-09-22 第二次事件：這次不是 push 被拒絕，而是 `git fetch origin` 本身就失敗，錯誤還是 `SEC_E_NO_CREDENTIALS`。這代表連 TLS 連線／認證都還沒建立起來，問題比第一節嚴重，要往「這個 process 有沒有辦法讀到憑證」查，不是往「分支落後」查。

**已經排除的可能性**（Claude 在同一台機器、同一個 repo，於同一時間點實測過）：
- Git 版本、`credential.helper=manager`、`http.sslbackend=schannel` 這些設定都是寫在 `C:\Program Files\Git\etc\gitconfig`（系統層級），Codex 跟其他程式讀到的是同一份設定，**不是設定檔案的問題**。
- `git-credential-manager diagnose` 全部項目（Environment／File system／Networking／Credential storage／Microsoft authentication／GitHub API）都是 `[ OK ]`。
- 同一個 remote，Claude 這邊 `git fetch` + `git push` 都直接成功，沒有任何錯誤。

**結論**：問題不在 repo、不在 Git 設定、不在 GitHub 帳號權限，而是在 **Codex 執行 Git 指令時所在的那個 process/session，缺少存取 Windows 憑證層的能力**。`SEC_E_NO_CREDENTIALS` 本質上是 Windows SSPI/Schannel 的錯誤，代表這個 process 在 TLS 交握時找不到可用的用戶端憑證/認證 context——常見於「這個 process 是在跟平常互動式使用者不同的安全性 context 下執行」，例如被限制網路存取的 sandbox、不同的 Windows access token、或看不到互動式使用者憑證存放區的執行環境。

**請 Codex 自己動手查這幾件事，不要再繼續猜：**

```bash
# 1. Codex 自己的執行環境，跟平常這台機器的互動式使用者環境是不是同一個
whoami
echo $HOME
echo $USERPROFILE   # Windows cmd/PowerShell 語法，視 Codex 實際 shell 而定
echo $APPDATA

# 2. 最關鍵的一步：繞過 Git，直接測試這個 process 能不能對 github.com 做 TLS 交握
#    如果這個指令也失敗，就百分之百證實是這個 process 的網路/TLS 層被限制，
#    跟 Git、GitHub 帳號完全無關
curl -v https://github.com 2>&1 | head -40

# 3. 如果 curl 也失敗，檢查這個 process 是不是被限制對外網路存取
#    （Codex CLI 若跑在 sandbox 模式，可能預設關閉網路存取，
#      需要在啟動 Codex 時明確允許 network access / 允許存取 github.com）

# 4. 如果 curl 成功、但 git 還是失敗，改用 openssl 當 SSL backend 而非 schannel，
#    排除 schannel 這個「讀 Windows 憑證存放區」的機制本身有問題
git config --global http.sslbackend openssl
git fetch origin
# 測完記得改回來（除非確認 openssl 比較穩定要固定使用）：
# git config --global --unset http.sslbackend
```

**最直接的解法（建議優先做）**：既然問題出在 HTTPS/Schannel 這條路徑，直接**整個 repo 改用 SSH remote**，完全避開 Schannel／Windows 憑證存放區：

```bash
# 前提：這台機器／這個帳號的 SSH key 要先加到 GitHub（Settings → SSH and GPG keys）
# 檢查有沒有 key，沒有就先產生一把
ls ~/.ssh/id_ed25519.pub 2>/dev/null || ssh-keygen -t ed25519 -C "weichinhiou@gmail.com"

# 測試 SSH 連線是否通（不需要 fetch/push 就能測）
ssh -T git@github.com

# 成功的話，把這個 repo 的 remote 改成 SSH
git remote set-url origin git@github.com:weichinhiou/church-speaker-roster.git
git fetch origin
git push origin main
```

SSH 用的是完全不同的認證機制（key pair，不經過 Windows Schannel／Credential Manager），如果 Codex 的執行環境只是「讀不到 Windows 憑證層」而網路本身是通的，改 SSH 通常可以直接解決，而且以後每次都穩定，不用再猜。

---

## 一句話總結

> 看到 push 失敗，先重跑一次、看清楚最新錯誤訊息：
> - `[rejected]` / `fetch first` → 分支落後，走第二～四步 fetch + rebase。
> - `Authentication failed` 但 fetch 有跑起來 → 走第五步查 credential helper/token。
> - **連 fetch 都失敗、TLS 交握階段就報 `SEC_E_NO_CREDENTIALS`** → 走第六步，先用 `curl -v https://github.com` 確認是不是這個 process 本身沒有網路/憑證存取能力，最快的解法通常是改用 SSH remote，不要一直在 HTTPS/Schannel 這條路上打轉。
