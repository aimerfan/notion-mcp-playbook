# notion-mcp-playbook

維護 `notion-mcp-playbook` skill 的 repo,產出可分發的 `notion-mcp-playbook.zip`。

Skill 名稱、repo 資料夾、輸出檔名、zip 內部根目錄一律 `notion-mcp-playbook`。

輸出選 `.zip` 而非 `.skill`:web 端上傳常擋非標準副檔名,本機 / CLI 載入兩種都吃。

## Source of truth

- `SKILL.md` — skill 內容變更都在這裡。
- Frontmatter 需有 `name` 與 `description`,description 是 agent 決定是否載入 skill 的依據。

## 打包流程

```powershell
Add-Type -AssemblyName System.IO.Compression.FileSystem
New-Item -ItemType Directory -Force -Path .\build | Out-Null
$zipPath = Join-Path (Get-Location) 'build\notion-mcp-playbook.zip'
if (Test-Path $zipPath) { Remove-Item $zipPath -Force }
$zip = [System.IO.Compression.ZipFile]::Open($zipPath, 'Create')
[System.IO.Compression.ZipFileExtensions]::CreateEntryFromFile($zip, (Resolve-Path .\SKILL.md).Path, 'notion-mcp-playbook/SKILL.md') | Out-Null
$zip.Dispose()
```

不能用 `Compress-Archive`:PowerShell 5.1 的版本會把反斜線 `\` 寫進 zip 內部路徑,違反 zip 規格,寬鬆的載入端會吞,嚴格的 web 上傳會擋 `Zip file contains path with invalid characters`。`System.IO.Compression.ZipFile` 可以顯式指定 entry name(用 `/`)。

關鍵是 `CreateEntryFromFile` 的第三個參數 `notion-mcp-playbook/SKILL.md` ——含資料夾前綴的完整路徑,不能只寫 `SKILL.md`,否則 zip 內會平鋪,失去 `notion-mcp-playbook/` 根目錄結構。

未來若 skill 擴充為多檔(reference docs、scripts 等),為每個檔案多呼叫一次 `CreateEntryFromFile`,entry name 用 `notion-mcp-playbook/<relative-path>`。

## Git

- `.zip`、`.skill`、`build/` 不進 git。
