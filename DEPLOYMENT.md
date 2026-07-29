# GitHub Pages 双入口部署

本文档说明如何在保留私有 Sites 地址的同时，用一个公开部署专用仓库免费提供
GitHub Pages 第二入口。

## 生产地址与仓库职责

| 项目 | 地址或仓库 | 可见性 | 职责 |
| --- | --- | --- | --- |
| 私有 Sites | `https://dividend-research-dashboard.lybwzy2002.chatgpt.site` | 私有 | 现有 Sites 生产入口 |
| GitHub Pages | `https://invisiblehomozygous.github.io/agent-security-dividend-pages/` | 公开 | 免费静态网页入口 |
| 源码仓库 | `invisiblehomozygous/agent-security-dividend` | 私有 | 源码、测试、构建和数据快照 |
| 部署仓库 | `invisiblehomozygous/agent-security-dividend-pages` | 公开 | 只保存生成后的静态产物 |

两个入口的代码部署互相独立。GitHub Pages发布不会修改
`web/.openai/hosting.json`，也不会替换或取消私有Sites部署；私有Sites运行时读取
Pages发布的同一份`dashboard-data.json`，因此日常数据更新只需发布一次快照。

## 部署链路

```text
私有源码仓库 main
        |
        | push 修改 web/**、工作流或本文档
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
- `web/.openai/hosting.json`及Sites部署元数据；
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

修改`main`分支中的以下路径会触发`.github/workflows/pages.yml`：

- `web/**`；
- `.github/workflows/pages.yml`；
- `docs/deployment-github-pages.md`。

常规数据更新与发布只需运行：

```bash
uv run dividend-research web export --live
```

命令要求本地处于已与GitHub同步的`main`分支，并使用已登录的GitHub CLI。它会：

1. 生成并校验`web/public/dashboard-data.json`；
2. 通过GitHub API只提交该快照，避免提交其他本地修改；
3. 等待`publish-public-pages`构建与测试成功；
4. 等待公开部署仓库的Pages任务成功；
5. 返回两个线上入口和对应提交。

私有Sites页面加载时会绕过缓存读取该公开快照，因此不需要为每次行情更新重新部署
Worker。只想生成本地快照时使用：

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

Pages展示的是私有源码仓库已提交的`web/public/dashboard-data.json`，不会在浏览器中
实时抓取成分数据。运行`uv run dividend-research web export --live`后，命令会自动提交
快照并等待发布。

### 私有 Sites 与 GitHub Pages 内容不同

两者的代码仍是独立发布通道。GitHub Pages跟随私有源码仓库`main`自动发布；Sites代码
变化仍需按Sites流程部署新版本。但日常行情数据来自同一Pages快照，执行
`web export --live`即可让两个入口同步更新。

## 停用

停用GitHub Pages时，在公开部署仓库`Settings -> Pages`取消发布，并删除私有源码仓库
中的`PAGES_DEPLOY_KEY`。如不再需要部署仓库，再单独归档或删除它。以上操作不会影响
私有Sites地址。
