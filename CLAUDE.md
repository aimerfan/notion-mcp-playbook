# notion-mcp-playbook

維護 `notion-mcp-playbook` skill 的 repo,產出可分發的 `notion-mcp-playbook.skill`(本質是 zip)。

Skill 名稱、repo 資料夾、輸出檔名、zip 內部根目錄一律 `notion-mcp-playbook`。

## Source of truth

- `SKILL.md` — skill 內容變更都在這裡。
- Frontmatter 需有 `name` 與 `description`,description 是 agent 決定是否載入 skill 的依據。

## 打包流程

```powershell
$staging = New-Item -ItemType Directory -Force -Path .\build\notion-mcp-playbook
Copy-Item .\SKILL.md $staging.FullName
Compress-Archive -Path .\build\notion-mcp-playbook -DestinationPath .\build\notion-mcp-playbook.zip -Force
Move-Item .\build\notion-mcp-playbook.zip .\build\notion-mcp-playbook.skill -Force
Remove-Item .\build\notion-mcp-playbook -Recurse -Force
```

`Compress-Archive -Path` 傳的是**資料夾**,不是 `SKILL.md` 本身,否則 zip 會平鋪,失去 `notion-mcp-playbook/SKILL.md` 結構。

未來若 skill 擴充為多檔(reference docs、scripts 等),調整 `Copy-Item` 那行。

## Git

- `.skill`、`.zip`、`build/` 不進 git。
