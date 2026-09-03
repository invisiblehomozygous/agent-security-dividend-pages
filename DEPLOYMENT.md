# GitHub Pages 单入口部署

本文档说明如何使用公开部署专用仓库提供唯一的线上 GitHub Pages 静态入口。

## 生产地址与仓库职责

| 项目 | 地址或仓库 | 可见性 | 职责 |
| --- | --- | --- | --- |
| GitHub Pages | `https://invisiblehomozygous.github.io/agent-security-dividend-pages/` | 公开 | 免费静态网页入口 |
| 源码仓库 | `invisiblehomozygous/agent-security-dividend` | 私有 | 源码、测试、构建和数据快照 |
| 部署仓库 | `invisiblehomozygous/agent-security-dividend-pages` | 公开 | 只保存生成后的静态产物 |

GitHub Pages是唯一线上前端。页面只读取该站点发布的静态资源和公开快照，不访问
远端账户自选接口，也不依赖其他运行时前端。

## 部署链路

```text
私有源码仓库 main
        |
        | 每次 push
        v
.github/workflows/pages.yml
        |
        +-- npm ci
        +-- 构建 Worker 版本
        +-- 预渲染静态 Pages 版本
        +-- 校验页面、成分股数量、资源路径和禁传文件
        |
        | 只使用写入部署仓库的 SSH Deploy Key
        v
公开部署仓库 main
        |
        v
GitHub Pages
```

部署仓库不接收源码仓库的 Git 历史，只接收 `web/out/` 中的最终产物。同步使用
`rsync --delete`，因此公开仓库始终与最近一次成功构建一致。

## 公开与私有边界

公开部署仓库包含：

- `index.html`和`404.html`；
- 浏览器运行所需的JavaScript、CSS和图片；
- `dashboard-data.json`公开行情与研究快照；
- `.nojekyll`、仓库说明和本文档。

以下内容不得进入部署仓库：

- Python、TypeScript或React源码；
- DuckDB、SQLite等数据库；
- `.env`、API Token、SSH私钥和任何凭据；
- 本地托管配置和运行时部署元数据；
- 本地日志、报告和运行缓存。

静态导出测试会按文件名拦截常见环境文件、密钥和数据库文件。任何显示在公开网页上的
数据都应视为公开信息。

## 首次配置

以下步骤只需要执行一次。

### 1. 创建公开部署仓库

```bash
gh repo create invisiblehomozygous/agent-security-dividend-pages \
  --public \
  --description "Generated GitHub Pages deployment for the dividend research dashboard" \
  --add-readme
```

部署仓库必须是公开仓库，默认分支为`main`。GitHub Free只支持从公开仓库免费使用
GitHub Pages。

### 2. 创建最小权限部署密钥

部署密钥仅允许私有源码仓库的工作流写入公开部署仓库，不授予其他仓库权限。

```bash
DEPLOY_KEY_DIR="$(mktemp -d)"
ssh-keygen -t ed25519 \
  -C "agent-security-dividend-pages deploy key" \
  -f "${DEPLOY_KEY_DIR}/pages-deploy" \
  -N ""

gh api --method POST \
  repos/invisiblehomozygous/agent-security-dividend-pages/keys \
  -f title="agent-security-dividend-pages deploy key" \
  -f key="$(cat "${DEPLOY_KEY_DIR}/pages-deploy.pub")" \
  -F read_only=false

gh secret set PAGES_DEPLOY_KEY \
  --repo invisiblehomozygous/agent-security-dividend \
  < "${DEPLOY_KEY_DIR}/pages-deploy"
```

确认密钥已经写入GitHub后，删除本地临时目录。不要打印、提交或发送私钥内容。

### 3. 启用 GitHub Pages

```bash
gh api --method POST \
  repos/invisiblehomozygous/agent-security-dividend-pages/pages \
  -f 'source[branch]=main' \
  -f 'source[path]=/'
```

也可以在公开部署仓库的`Settings -> Pages`中选择`Deploy from a branch`，
分支选择`main`，目录选择`/ (root)`。

## 自动发布

`main`分支的每次 push 都会触发`.github/workflows/pages.yml`，自动构建并同步 GitHub
静态页，不依赖变更文件所在目录。每个线上版本都可以通过源码提交定位和核对。

常规数据更新与发布只需运行：

```bash
uv run dividend-research web export --live
```

该命令使用本机已登录的Codex CLI实时搜索并刷新过期题材研究和财报分析，同时生成
`dashboard-data.json`、`theme-research.json`和`financial-research.json`；无需配置
`OPENAI_API_KEY`。

命令要求本地处于已与GitHub同步的`main`分支，并使用已登录的GitHub CLI。它会：

1. 生成并校验`web/public/dashboard-data.json`、`web/public/theme-research.json`与
   `web/public/financial-research.json`；
2. 在本地完成Pages构建和质量门禁，失败时不会提交快照；
3. 对实时行业接口缺失的股票沿用上一成功快照分类并保留降级提示；
4. 通过GitHub API在同一提交中只更新公开数据文件，避免提交其他本地修改；
5. 等待`publish-public-pages`构建与测试成功；
6. 等待公开部署仓库的Pages任务成功；
7. 返回GitHub Pages入口和对应提交。

浏览器只读取GitHub Pages上的公开静态快照，不会执行远端账户自选同步。只想生成本地
快照时使用：

```bash
uv run dividend-research web export --live --no-publish
```

构建或测试失败时，线上版本保持不变。也可以在私有源码仓库的
`Actions -> publish-public-pages -> Run workflow`手动触发。

## 本地构建与验证

```bash
cd web
npm ci
npm run build:pages -- \
  --base-path "/agent-security-dividend-pages" \
  --site-url "https://invisiblehomozygous.github.io/agent-security-dividend-pages"
PAGES_BASE_PATH="/agent-security-dividend-pages" \
PAGES_SITE_URL="https://invisiblehomozygous.github.io/agent-security-dividend-pages" \
  npm run test:pages
```

静态产物位于`web/out/`。该目录被Git忽略，不应提交到私有源码仓库。

发布前可额外检查文件清单：

```bash
find web/out -type f -print | sort
```

## 手工恢复发布

自动工作流不可用时，可以用已登录且有部署仓库写权限的GitHub账号手工发布：

```bash
DEPLOY_WORKTREE="$(mktemp -d)"
git clone \
  https://github.com/invisiblehomozygous/agent-security-dividend-pages.git \
  "${DEPLOY_WORKTREE}/repository"
rsync -a --delete --exclude ".git/" web/out/ "${DEPLOY_WORKTREE}/repository/"
git -C "${DEPLOY_WORKTREE}/repository" add -A
git -C "${DEPLOY_WORKTREE}/repository" commit -m "deploy: manual recovery"
git -C "${DEPLOY_WORKTREE}/repository" push origin main
```

如果没有产物变化，`git commit`会提示无内容可提交，可以直接结束。

## 回滚

推荐在私有源码仓库回滚导致问题的提交，再由自动流程重新生成公开产物：

```bash
git revert <需要回滚的源码提交>
git push origin main
```

紧急情况下也可以在公开部署仓库回滚最近一次部署提交，但下一次自动发布会再次以私有
源码仓库`main`的当前状态为准：

```bash
git revert <需要回滚的部署提交>
git push origin main
```

## 部署密钥轮换

1. 在公开部署仓库`Settings -> Deploy keys`删除旧密钥；
2. 在私有源码仓库`Settings -> Secrets and variables -> Actions`删除或替换
   `PAGES_DEPLOY_KEY`；
3. 按“创建最小权限部署密钥”重新生成并登记密钥；
4. 手动运行一次`publish-public-pages`验证；
5. 删除本地临时私钥。

不要复用个人SSH密钥，也不要给部署密钥超出部署仓库的权限。

## 故障排查

### 工作流提示缺少 `PAGES_DEPLOY_KEY`

确认密钥保存在私有源码仓库的Actions repository secrets中，名称必须完全一致。

### 推送提示 `Permission denied (publickey)`

确认私钥与公开部署仓库`Deploy keys`中的公钥配对，并且Deploy Key启用了写权限。
轮换密钥后需要同时更新两处。

### 页面返回 404

检查公开部署仓库`Settings -> Pages`：

- Source为`Deploy from a branch`；
- Branch为`main`；
- Folder为`/ (root)`；
- 最新`main`分支根目录存在`index.html`和`.nojekyll`。

首次启用或刚推送后，GitHub Pages可能需要短暂时间完成发布。

### 页面打开但样式或脚本 404

确认构建使用的子路径为`/agent-security-dividend-pages`，不要使用私有源码仓库名。
本地重新运行`npm run build:pages`和`npm run test:pages`定位错误。

### 成分股数量不是预期值

Pages展示的是私有源码仓库已提交的`web/public/dashboard-data.json`和
`web/public/theme-research.json`、`web/public/financial-research.json`，不会在浏览器中
实时抓取行情或执行AI研究。运行`uv run dividend-research web export --live`后，命令会
自动提交公开快照并等待发布。

### GitHub Pages 内容不是最新版本

以私有源码仓库`main`中的最新提交为准，检查`publish-public-pages`和公开部署仓库的
Pages工作流是否均已成功。日常行情数据执行`web export --live`即可重新生成并发布。

## 停用

停用GitHub Pages时，在公开部署仓库`Settings -> Pages`取消发布，并删除私有源码仓库
中的`PAGES_DEPLOY_KEY`。如不再需要部署仓库，再单独归档或删除它。
