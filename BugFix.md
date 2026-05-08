这是一份专门针对 **Svelte 3 + Rollup + PostCSS + Tailwind** 项目部署到 GitHub Pages 的全流程指南。它总结了我们之前解决的所有“血泪教训”，建议收藏作为以后的标准模板。

---

# 🚀 Svelte 项目部署 GitHub Pages 终极指南

## 一、 核心准备（代码层面）

在部署之前，必须确保代码能够适配 GitHub Pages 的 **子路径环境**（例如 `username.github.io/repo-name/`）。

### 1. 相对路径（最重要！）
GitHub Pages 部署在子目录下，绝对路径（以 `/` 开头）会导致资源 404。
*   **HTML 模板** (`src/template.html`)：
    将所有 `/bundle.js` 改为 `./bundle.js`，所有 `/logo-192.png` 改为 `./logo-192.png`。
*   **Manifest** (`manifest.json`)：
    内部所有图标路径必须改为 `./` 开头。

### 2. 禁用 Jekyll
GitHub 默认使用 Jekyll 处理网页，它会忽略以 `_` 开头的文件夹，并可能尝试转换你的 CSS。
*   **操作**：在 `static` 文件夹（或打包后的根目录）创建一个名为 `.nojekyll` 的空文件。

### 3. 环境一致性
确保 `package.json` 中的依赖版本逻辑自洽。
*   **Svelte 3** 建议搭配 **PostCSS 8** 和 **Tailwind v3**。
*   如果安装失败，务必使用 `--legacy-peer-deps` 标志。

---

## 二、 自动化部署（GitHub Actions 流程）

不要手动上传 `dist` 文件夹，使用 GitHub Actions 可以实现“推送即更新”。

### 1. 创建配置文件
路径：`.github/workflows/deploy.yml`

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: ["main"] # 监听主分支推送

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install Dependencies
        # 使用 --legacy-peer-deps 解决 Svelte 3 与新版插件的版本冲突
        run: npm install --legacy-peer-deps

      - name: Build
        run: npm run build

      - name: Create .nojekyll
        run: touch dist/.nojekyll

      - name: Upload Pages Artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./dist # 必须指向你的打包输出目录

      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v4
```

---

## 三、 GitHub 仓库设置

1.  **开启权限**：进入仓库 `Settings` -> `Pages`。
2.  **选择源**：在 `Build and deployment` -> `Source` 中选择 **GitHub Actions**。
3.  **等待运行**：在 `Actions` 选项卡可以看到进度，绿色勾选后即可通过生成的链接访问。

---

## 四、 避坑指南（Hall of Fame of Errors）

### 1. `node.getIterator is not a function`
*   **原因**：PostCSS 8 删除了旧接口，但你安装了只支持 PostCSS 7 的旧插件（如 `tailwindcss v1` 或 `postcss-clean`）。
*   **解决**：升级 `tailwindcss` 到 v3，使用 `cssnano` 替换 `postcss-clean`。

### 2. 控制台刷屏 404 (manifest.json, favicon, logo)
*   **原因**：路径里多了一个斜杠 `/`，导致浏览器去 GitHub 的根域名找资源。
*   **解决**：全局搜索并替换为相对路径 `./`。

### 3. Tailwind JIT 模式下样式消失
*   **原因**：Tailwind v3 不支持动态拼接类名（如 `row-start-{n}`）。
*   **解决**：改用原生 CSS Grid 属性（`grid-template-columns`）或将动态类名写入 `safelist`。

### 4. `npm ci` 报错
*   **原因**：`npm ci` 极其严格，`package-lock.json` 稍微有一点版本不匹配（比如 picomatch 或 yaml 版本微调）就会崩。
*   **解决**：在 CI 脚本中使用 `npm install --legacy-peer-deps` 代替 `npm ci`。

### 5. `ReferenceError: production is not defined`
*   **原因**：在 `rollup.config.js` 中，变量定义顺序错误（先用了 `production` 后定义的它）。
*   **解决**：将 `const production = ...` 挪到文件最顶部。

### 6. UI 颜色被覆盖/看不见笔记
*   **原因**：CSS 权重问题，或者使用了不透明背景色。
*   **解决**：使用 `ring` 边框代替背景色表示选中，使用 `!important` 或增加 CSS 选择器权重。

---

## 五、 部署后的日常维护
*   **更新图标**：直接替换 `static/` 下的 PNG，并确保 `manifest.json` 引用正确。
*   **本地验证**：在 Push 之前，务必先运行一次 `npm run build`，只要本地 build 文件夹里的 `index.html` 能在浏览器正常（虽然 JS 可能路径不对，但 HTML 结构要对），Actions 成功率就很高。
