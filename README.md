# notion-mcp-playbook

可分發的 Claude skill,內容是 Notion MCP tool 的 quirks 與 working patterns,給 agent 在呼叫 Notion MCP 前先讀,避免 silent wrong behaviour 或 confusing errors。

## 這個 repo 是什麼

- 產出物:`notion-mcp-playbook.skill`(zip,丟給 Claude Code / Claude.ai 載入)
- 內容來源:`SKILL.md` 單一檔案
- 範疇:只寫 MCP tool 機制(DDL parser 限制、read/write format 落差等),不寫使用者該如何組織 Notion content

## 維護切入點

要改 skill 內容 → 編輯 `SKILL.md`,其他都不用動。

要重打包 → 跑 `CLAUDE.md` 的 PowerShell 區塊,輸出在 `build/notion-mcp-playbook.skill`。

要新增非 `SKILL.md` 的檔案(reference docs、scripts)→ 改 `CLAUDE.md` 打包流程那一行 `Copy-Item`。

## 注意事項

- Frontmatter 的 `description` 是 agent 判斷是否載入 skill 的關鍵,改它前先想清楚 trigger 條件
- `.skill` / `.zip` / `build/` 已在 `.gitignore`,不要 commit build artifact
- Skill 名稱、資料夾、檔名、zip 內部根目錄都是 `notion-mcp-playbook`,改名要全部一起改

## 更多細節

詳細的編輯與打包規範看 `CLAUDE.md`。
