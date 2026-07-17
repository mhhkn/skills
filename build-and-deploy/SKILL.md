---
name: "build-and-deploy"
description: "构建原型页面成静态资源，用于部署演示。Invoke when user wants to build and deploy the project to a web server."
---

# 构建原型页面成静态资源，用于部署演示

将 Axhub Make 项目的原型页面构建为独立可运行的静态资源，生成可直接部署到 Nginx/Apache 的 `dist/` 文件夹。

全自动完成，自动检测 `src/prototypes/` 下的所有原型，自动扫描代码中引用的本地资源文件并复制到 `dist/` 中。

## 前置条件

- Node.js 已安装
- 项目依赖已安装（`npm install`）
- Axhub Make 项目，原型页面在 `src/prototypes/`

## 步骤

### Step 1: 自动检测原型

用 Glob 查找所有原型目录：

```
Glob: src/prototypes/*/index.tsx
```

提取每个原型名称（目录名），并读取 `index.tsx` 前 3 行的 `@name` 注释作为显示标题。

### Step 2: 构建所有原型

为每个检测到的原型运行：

```powershell
$env:ENTRY_KEY = "prototypes/<prototype-name>"
npx vite build 2>&1 | Select-String -NotMatch "Duplicate key"
```

构建产物 JS 文件位于 `dist/prototypes/<prototype-name>.js`。

### Step 2.5: 复制代码中引用的本地资源文件

扫描 `src/` 下所有 `.ts` / `.tsx` 代码，查找字符串中引用的本地资源文件路径（如 `/resources/xxx.docx`、`/assets/xxx.png`），只将确实存在于磁盘上的文件复制到 `dist/` 对应目录，且仅复制被引用的文件。

```powershell
# 1. 创建 dist/resources 目录（兜底，有 assets/ 等的也会自动创建）
New-Item -ItemType Directory -Force -Path "dist/resources" | Out-Null

# 2. 从代码中提取所有带扩展名的本地资源引用路径
$pattern = '/[^\"''\s]+\.(docx?|png|jpe?g|gif|svg|pdf|xlsx?|csv|json|xml|ico|webp|woff2?|eot|ttf|mp4|webm)'
$refs = @()
$refs += Select-String -Path "src\**\*.ts" -Pattern $pattern -AllMatches `
  | ForEach-Object { $_.Matches.Value } | Sort-Object -Unique
$refs += Select-String -Path "src\**\*.tsx" -Pattern $pattern -AllMatches `
  | ForEach-Object { $_.Matches.Value } | Sort-Object -Unique
$refs = $refs | Sort-Object -Unique

# 3. 复制每个被引用的文件（仅当文件存在时）
foreach ($ref in $refs) {
    # 去掉开头的 /，得到相对于 src/ 的路径
    $relPath = $ref.TrimStart('/')
    $srcPath = Join-Path "src" $relPath
    $destPath = Join-Path "dist" $relPath
    if (Test-Path $srcPath) {
        $destDir = Split-Path $destPath -Parent
        New-Item -ItemType Directory -Force -Path $destDir | Out-Null
        Copy-Item $srcPath $destPath
        Write-Host "  Copied: $relPath"
    }
}
if ($refs.Count -eq 0) {
    Write-Host "  No local resource files referenced in source code."
}
```

这会将所有被代码引用的本地资源（不限目录）按原始路径复制到 `dist/` 下，保持目录结构一致。

### Step 3: 生成原型 HTML 文件

为每个原型创建 `dist/prototypes/<prototype-name>/index.html`：

- 使用 `@name` 注释值作为 `<title>`，若无可使用原型目录名
- 从 CDN 加载 React 18：`https://unpkg.com/react@18/umd/react.production.min.js`
- 从 CDN 加载 ReactDOM：`https://unpkg.com/react-dom@18/umd/react-dom.production.min.js`
- 挂载点 div：`<div id="root"></div>`
- 通过 `window.__AXHUB_DEFINE_COMPONENT__` 用 ReactDOM.createRoot 注册组件
- 加载 JS：`<script src="../<prototype-name>.js"></script>`

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title><auto-detected-title></title>
  <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
</head>
<body>
  <div id="root"></div>
  <script>
    window.__AXHUB_DEFINE_COMPONENT__ = function (Component) {
      var container = document.getElementById('root');
      if (container) {
        var root = ReactDOM.createRoot(container);
        root.render(React.createElement(Component));
      }
    };
  </script>
  <script src="../<prototype-name>.js"></script>
</body>
</html>
```

### Step 4: 生成需求文档入口

在生成导航页之前，先处理需求文档：

1. 用 Glob 查找 `src/resources/` 下的 Markdown 文件：`Glob: src/resources/*需求*.md` 或 `Glob: src/resources/*.md`
2. 如果找到需求文档（文件名含"需求"或"需求文档"关键词），将其转为 HTML 文件放入 `dist/resources/`（浏览器可以直接预览 HTML，而 `.md` 文件会触发下载），记录文件名
3. 如果未找到，**先询问用户**需求文档的文件名是什么，然后再处理

```powershell
# 查找需求文档
$reqDoc = Get-ChildItem "src/resources" -Filter "*需求*.md" | Select-Object -First 1
if (-not $reqDoc) {
    $reqDoc = Get-ChildItem "src/resources" -Filter "*.md" | Select-Object -First 1
}
if ($reqDoc) {
    New-Item -ItemType Directory -Force -Path "dist/resources" | Out-Null

    # 读取 markdown 内容并转为 HTML，确保浏览器可预览（不触发下载）
    $mdText = [System.IO.File]::ReadAllText($reqDoc.FullName)
    $escaped = [System.Net.WebUtility]::HtmlEncode($mdText)
    $htmlName = "$($reqDoc.BaseName).html"
    $htmlContent = @"
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>$($reqDoc.BaseName)</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
      background: #f5f5f5;
      color: #333;
      padding: 40px 20px;
    }
    .container {
      max-width: 900px;
      margin: 0 auto;
      background: #fff;
      border-radius: 8px;
      box-shadow: 0 1px 3px rgba(0,0,0,0.08);
      padding: 40px;
    }
    pre {
      white-space: pre-wrap;
      word-wrap: break-word;
      font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
      font-size: 14px;
      line-height: 1.7;
      color: #333;
    }
  </style>
</head>
<body>
  <div class="container">
    <pre>$escaped</pre>
  </div>
</body>
</html>
"@
    Set-Content -Path "dist/resources/$htmlName" -Value $htmlContent -Encoding UTF8
    Write-Host "  Copied 需求文档: $htmlName"
} else {
    Write-Host "  未找到需求文档，请联系用户确认文件名"
    # 此处应暂停并 AskUserQuestion，询问需求文档文件名
}
```

> **注意**：将 `.md` 转为 `.html` 是为了浏览器可直接预览。`.md` 文件在静态服务器上会被识别为 `application/octet-stream` 或 `text/markdown`，导致浏览器直接下载而非渲染。

### Step 5: 生成导航首页 index.html

按原型目录名的公共前缀自动分组（取第一个 `-` 之前的部分作为分组名，首字母大写），无前缀的归入"其他"组。

从 `@name` 注释中提取每个原型的短名称作为显示标题，若无 `@name` 则从目录名推导。

生成 `dist/index.html` 卡片式导航页面，要求：
- 所有原型链接使用 `target="_blank"` 新标签页打开
- 页面左下角显示更新时间，格式为「更新于：YYYY-MM-DD HH:mm:ss」，使用 PowerShell 获取构建时刻的固定时间戳（如 `(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')`）直接硬编码到 HTML 中，不可用 JavaScript 动态生成
- 如果需求文档已复制（`dist/resources/需求文档.html`），在导航页底部增加一个「需求文档」入口链接 `<a href="resources/需求文档.html" target="_blank">`，点击在新标签页打开

### Step 6: 验证目录结构

生成完成后，验证 `dist/` 目录结构：

```
dist/
├── index.html                              (导航首页)
├── resources/                              (运行时资源文件)
│   ├── 需求文档.html                         (需求文档，可选)
│   ├── <template.docx>
│   └── ...
├── assets/                                 (其他代码引用的本地资源，可选)
│   └── ...
├── prototypes/
│   ├── <prototype-name>/index.html         (单个原型 HTML)
│   ├── <prototype-name>.js                 (构建的 IIFE JS)
│   ├── <proto2-name>/index.html
│   ├── <proto2-name>.js
│   └── ...
```

### Step 7: 部署到 Nginx

将 `dist/` 目录上传到服务器。Nginx 配置参考：

```nginx
server {
    listen 80;
    server_name your-domain.com;
    root /path/to/dist;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Step 8: 验证部署

测试部署后的 URL：

```
http://your-server/                              → 导航首页
http://your-server/prototypes/<prototype-name>/   → 单个原型页面
http://your-server/resources/需求文档.html           → 需求文档（如有）
```

## 说明

- **CSS 内联在 JS 中** — Vite IIFE 构建已将 CSS 打包进 JS，无需额外 CSS 文件
- **React 从 CDN 加载** — 服务器需要联网。离线部署时需下载 React/ReactDOM UMD 包并本地托管
- **JS 路径规则**：HTML 在子目录中，JS 在父级 `prototypes/` 文件夹，因此路径为 `../<name>.js`
- **资源文件**：只复制代码（`.ts`/`.tsx`）中通过字符串引用且实际存在于磁盘上的本地资源文件，不复制未引用的文档或占位文件
- **需求文档**：导航页会自动包含 `需求文档.html` 入口（将 `.md` 转为 `.html` 以确保浏览器可预览），如果代码目录中找不到，在执行 skill 时会询问用户确认文件名
